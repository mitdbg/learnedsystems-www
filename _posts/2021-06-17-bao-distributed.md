---
layout: post
title: "More Bao Results: Learned Distributed Query Optimization on Vertica, Redshift, and Azure Synapse"
image:
  path: /assets/bao/bao_blog_diag.svg
  height: 100
  width: 100
---

*Author: [Ryan Marcus](https://rmarcus.info)*

Next week, we'll present our new system for learned query optimization, [Bao](https://rm.cab/bao), [SIGMOD21](https://2021.sigmod.org/), where we are thrilled to receive a [best paper award](https://2021.sigmod.org/sigmod_best_papers.shtml).

In our paper, we show how Bao can be applied to the open-source [PostgreSQL DBMS](https://www.postgresql.org/), as well as an [unnamed](https://en.wikipedia.org/wiki/David_DeWitt) commercial system. Both DBMSes ran in a traditional, single-node environment. Here, we'll give a brief overview of the Bao system and then walk through our early attempts at applying Bao to commercial, cloud-based, distributed database management systems.


For more information on Bao in the traditional, single-node context, check out:

* [Our research paper](https://rm.cab/bao), published in SIGMOD 2021.
* For a video overview, the recording of Ryan's SIGMOD talk ([20 minute version](https://www.youtube.com/watch?v=nEy90-WNkjo) or [3 minute version](https://www.youtube.com/watch?v=-tvt8QzZcXM)).
* For an overview of tree convolution as applied to query plans, Ryan's [AIDB 2019 talk](https://www.youtube.com/watch?v=g4iiDVWtQZo).

However, since we wrote the paper almost a year ago, much has happened. First, We worked together with Microsoft to explore how Bao can help with Big Data workloads.
This work will also be presented at SIGMOD as part of the industry session, and received a honorable mention for the industry best paper award.

Second, after the discussion we had with Mike Stonebraker around the potential impact Bao could have on distributed database warehouse systems (obviously, he was very skeptical), we ran a whole range of additional experiments on Vertica, Redshift, and Azure Synapse using a real-world dataset we received from an anonymous corporation.


# How Bao works

Previous approaches[^prev] to learned query optimization attempted to replace large parts of traditional query optimizers. In contrast, Bao sits on top of a traditional query optimizer (called the *underlying optimizer*) and learns to steer the underlying optimizer in the right direction.

![Overview of the Bao process. Text in image repeated below.](/assets/bao/bao_blog_diag.svg)

The image above illustrates the process Bao uses to steer the underlying query optimizer.

1. When a query arrives, the underlying optimizer is used to generate a number of plan variants for the query. For example, we might have the optimizer generate a plan using index nested-loop joins, another plan using sort-merge joins, and a third plan using hash joins.[^arms]
2. Once these plan variants are constructed, a predictive model (a deep neural network) predicts the latency of each plan.
3. The plan with the best predicted latency is selected and executed. The result is sent to the user, and the actual latency is recorded.
4. The actual latency of the recorded query is added to Bao's experience set, which is used to further refine the predictive model using [Thompson sampling](https://en.wikipedia.org/wiki/Thompson_sampling).[^nosup]

Over time, Bao's predictive model learns from its mistakes, hopefully making increasingly accurate predictions. Experimentally, we've [shown](https://rm.cab/bao) that Bao can learn in the presence of dynamic workloads, shifting data distributions, and schema modifications.

# From single-node to distributed

In the original Bao paper, we evaluated Bao on a single-node database system (e.g., Oracle and PostgreSQL). However, many data warehouse databases are, in fact, distributed. Luckily, Bao is largely agnostic to the underlying execution engine or storage layout: as long as you have a set of hints, Bao can pick and choose from them and learn from its mistakes. In the paper, we discuss what good hints for a single-node DBMS look like: for example, forcing particular join types, or forcing particular access paths (i.e., index vs. table scan). These choices can impact the performance of the plan on their own, but can also impact the join order selected by the underlying query optimizer, [potentially resulting in drastically different run times](https://www.vldb.org/pvldb/vol9/p204-leis.pdf).

**Adapting Bao to a distributed DBMS is simply a matter of finding the right hints.** While many aspects of the single-node case apply to the distributed case as well (operator choice matters, join order matters), distributed DBMSes bring about other important performance considerations:

Suppose a fact table is distributed across multiple nodes based on a key column. Should a join of that fact table with a dimension table be done via sending the dimension table to all nodes and using a hash algorithm? By partitioning the dimension table on the foreign key and using a merge join and union? By collecting the matching rows of the fact table on a single node, then performing a non-distributed merge?

Depending on network costs, query selectivity, materialization strategy, what data is already present at each node, and a wide range of other factors, any of these strategies might be applicable. Different DBMSes will lean towards different options depending on which code paths have been optimized. Most distributed DBMSes choose between these different strategies using the same tools that non-distributed DBMSes use: heuristics, cost models, and cardinality estimation. These tools are already highly error-prone (and require significant tuning) in non-distributed settings, so you can imagine how tricky things get when entire clusters are involved!

Next, we'll walk through how Bao can be applied to three different state-of-the-art commercial cloud distributed DBMSes: Vertica, Amazon Redshift, and Azure Synapse (an analytics-focused offering of SQL Server). After discussing each system, we'll show a small experiment highlighting potential gains (or lack thereof) from applying Bao. *We'll be running each DBMS on different hardware, so please do not attempt to draw comparisons between these DBMSes*.

In each test, we'll be executing an analytic dashboarding workload called `Corp`. The workload was donated to our research team by a large corporation under the condition of anonymity. The workload contains about 1 TB of data and 2000 unique queries which change over time. A large schema change (normalizing a fact table) happens in the middle of the workload -- we do not count the time required to perform this modification. The workload makes use of analytic functions, (materialized) views, and other advanced SQL features.

## Vertica

Vertica is a distributed columnar DBMS, and is the commercial adaptation of the [C-Store paper](https://dl.acm.org/doi/pdf/10.1145/3226595.3226638). Vertica's optimizer is reasonably transparent, and the [documented set of hints](https://www.vertica.com/docs/10.0.x/HTML/Content/Authoring/SQLReferenceManual/LanguageElements/Hints/Hints.htm) gives us a lot of control over what the optimizer considers.

For Vertica, we select query hints to:
* Force a particular join operator (`JTYPE` hint),
* force a particular group by operator (`GBYTYPE` hint),
* force a particular distributed join algorithm (`DISTRIB` hint, with options `L`, `R`, `B`, or `A`, representing a local join, a resegment join, a broadcast join, or letting the optimizer pick, respectively).

We started up two identical 3-node clusters on AWS using the official Vertica image (which, at time of writing, uses `r4.4xlarge` nodes by default). We loaded the data into both clusters, and used the [Vertica DBD tool](https://www.vertica.com/docs/10.0.x/HTML/Content/Authoring/AdministratorsGuide/ConfiguringTheDB/PhysicalSchema/DBD/AboutDatabaseDesigner.htm) to create a good layout, which we manually verified through testing. One cluster ran Bao on top of Vertica (`Bao`), while the other cluster did not run Bao (`Vertica`). The resulting time and costs are plotted below:

![Cost and latency data for Vertica and Vertica with Bao, described below](/assets/bao/bao_vert.svg)

Bao was able to reduce the end-to-end processing time of this workload by over three hours, while reducing cost by over 25% (about $25, in our case). The time savings with Bao are more significant than the cost savings because Bao must periodically retrain its predictive model, which (temporarily) requires a GPU.

Around 45% of Bao's gains (the plurality) come from forcing the Vertica optimizer to use a broadcast join instead of a resegment join when the Vertica optimizer wrongly over-estimates the cardinality of a particular subplan. The small subplan result can be easily materialized and sent to all nodes, allowing for a faster computation than shuffling around parts of the subplan evenly to each node in the cluster (a resegment).

In the future, we intend to experiment with specific hints to include or exclude projections (`PROJS` and `SKIP_PROJS`) to allow Bao to further custom-tailor its strategy to the user's data.

## Azure Synapse (SQL Server)

Azure Synapse is a analytics offering from Microsoft, based on SQL Server. SQL Server is one of the most widespread commercial database systems. Unlike Vertica, SQL Server can store data in either a row-based or a column-based format. Here, we'll focus on the column store format provided by the Azure Synapse Analytics cloud database. Unfortunately, the types of hints available for Synapse Analytics [are quite limited](https://docs.microsoft.com/en-us/sql/t-sql/queries/option-clause-transact-sql) -- so we had to get a bit creative.

For Synapse, we select query hints to:
* Disable or enable a particular type of join operator (hash, merge, and/or loop),
* use a replicated or partitioned versions of all dimension tables,
* use an indexed or non-indexed version of the fact table

In order to implement the last two hints, we cheat a little bit: we created two versions of each dimension table, one replicated and one partitioned, along with two versions of the fact table (one indexed and one non-indexed). Based on the hint Bao selects, we rewrite the query to use the correct versions of each table.

We spun up two Azure Synapse Analytic instances, each with a dedicated 1000 DWUs ("data warehouse units," a measure of query processing resources available to queries). We ran one instance with Bao and the other instance without Bao. After loading the data, we added the specialized tables described above to the instance using Bao. The other instance, which would use the stock optimizer, had all dimension tables replicated and an indexed version of the fact table. The resulting time and costs are plotted below:

![Cost and latency data for Azure and Azure with Bao, described below](/assets/bao/bao_ss.svg)

Bao was able to reduce the end-to-end runtime of this workload by a little over two hours, while reducing the cost by around 10% (a little under $35). Again, the cost reduction is smaller than the latency reduction because of the cost of training the Bao model.

A plurality of Bao's gains (40%) came from avoiding the index on the fact table (sometimes by removing loop join as an option, sometimes by rewriting the query to use the non-indexed table). Removing the index from the instance running with the stock optimizer led to a decrease in performance, meaning that while the index normally helped, it hurt some queries, and Bao was able to identify those queries automatically.

In the future, Bao could be applied to Azure Synapse in serverless mode, but this mode currently does not support query hints.

## Redshift

Redshift is Amazon's data analytics offering, and is often cited as one of the first "cloud native" analytics databases. As far as we can tell, Redshift only offers *two* query optimization hints, which we'll use alongside the same trick we used for Synapse.[^noindex]

For Redshift, we select query hints to:
* Disable or enable AQUA, the Redshift query accelerator (`activate_aqua`),
* disable or enable using (up-to-date) materialized views in query processing (`mv_enable_aqmv_for_session`),
* use a replicated or partitioned version of all dimension tables (as with Synapse).

Unfortunately, Redshift offers by-far the least visibility and control into its query optimizer, leaving our Bao implementation (seemingly) limited. We started two 3-node clusters (`ra3.4xlarge` nodes), one running Bao and one running the stock optimizer. The results are plotted below:

![Cost and latency data for Redshift and Redshift with Bao, described below](/assets/bao/bao_rs.svg)

Bao speeds up total workload latency by over an hour, and reduces costs by around 10% (in this case, about $10). Most of the gains (65%) came from disabling the usage of materialized views within subtree expressions, which the Redshift optimizer seems to do a little too aggressively (e.g., scanning the entire materialized view is slower than applying the predicates to the relations involved and performing the join). Disabling the use of materialized views in query processing globally hurts query performance overall, again showing that Bao is able to learn when the feature helps and when it hurts.

Redshift seems like a really cool system, but unfortunately it does not provide the visibility or control required by researchers for deep study in query optimization. In the future, it would be awesome to see Amazon build a few more windows and knobs (with sane defaults!) into the optimizer for scientists to use.

# Discussion
Looking over the results, we find ourself returning to the same set of takeaways:

* Don't spend too much time comparing results between systems. The experiments are ran on different clouds, using different hardware, at different times of day. While we followed best practice guidelines to tune the systems, an expert might still do better, in particular for Redshift and Azure Synapse, as we have less experience with them.
* Just because Bao produces large gains on system X but smaller gains on system Y doesn't mean the default optimizer of system X is better than the default optimizer of system Y. In our opinion, it is more likely that system X provides more visibility into the optimizer, whereas system Y keeps things pretty opaque, limiting Bao's possible gains.
* Everyone wants a "knob-free" query optimizer, but nobody tells you that the cost of such is decreased query performance. Without incorporating feedback from query execution in some way, heuristic optimizers are doomed to have edge cases that make them fall flat. "Knob-free" sometimes just means "not sophisticated enough to give you the tools you need to fix the mistakes." A real "knob-free" query optimizer should take query latency feedback into account.
* The idea behind Bao -- use a deep neural network as a predictive model to choose between a number of different variants, then retrain that model progressively using Thompson sampling -- seems widely applicable. We focused on database systems, but maybe there are other applications as well?

So is there a downside of Bao? Certainly! Bao causes query optimization to take a little bit more time (~300ms), requiring quite a bit more computation. We studied this overhead in our SIGMOD paper. For data warehouse workloads, which largely consists of long-running, resource intensive queries, Bao's increased overhead is hardly noticeable. However, for workloads with a lot of short running queries, like OLTP workloads, this might not be the case. We are currently working on new approaches to mitigate that problem -- so stay tuned!

If you feel so inclined, you can [read the Bao paper](https://rm.cab/bao), check out our [open source prototype](https://learned.systems/bao) for PostgreSQL, or take a look at the [other publications](http://dsail.csail.mit.edu/index.php/publications-new/) from our group.


# Notes

[^prev]: Approaches such as [ReJOIN](https://rm.cab/rejoin), [DQ](https://arxiv.org/abs/1808.03196), and [Neo](https://rm.cab/neo) (in chronological order) fall into this category.

[^arms]: The choice of which plan variants to generate is important. Generally, one variant is always just what the underlying query optimizer would have chosen with no intervention, and the other variants are generated by forcing the optimizer to ignore particular optimization paths. See the paper for more details.

[^nosup]: Note that training the predictive model using standard supervised learning techniques will not balance the exploration of new policies (i.e., trying something new to see how well it works) against the exploitation of existing knowledge (i.e., doing what we know works well). Thus, [reinforcement learning](https://en.wikipedia.org/wiki/Reinforcement_learning) techniques are preferred. It is easy to think that, to avoid regression, you would want to maximize exploitation. But this can get you trapped in a local minima: if you mis-predict the value of the optimal plan, you will never select it. In other words, without exploration, you might initially have fewer regressions, but you'll never recover from them!

[^noindex]: Redshift, like Vertica, does not support traditional indexes on tables like Synapse does, so we do not add those indexed variants here.
