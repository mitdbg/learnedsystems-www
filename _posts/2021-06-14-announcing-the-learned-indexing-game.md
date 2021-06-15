---
layout: post
title:  "Announcing the Learned Indexing Game"
---

*Authors: Allen Huang, [Andreas Kipf](https://people.csail.mit.edu/kipf/), [Ryan Marcus](https://rmarcus.info/blog/), and [Tim Kraska](https://people.csail.mit.edu/kraska/)*

[Learned indexes](https://dl.acm.org/doi/pdf/10.1145/3183713.3196909) have received a lot of attention over the past few years. The idea is to replace existing index structures, like B-trees, with learned models. In recent a [paper](https://vldb.org/pvldb/vol14/p1-marcus.pdf), which we did in collaboration with TU Munich and are going to present at [VLDB 2021](https://vldb.org/2021/), we compared learned index structures against various highly tuned traditional index structures for in-memory read-only workloads. The benchmark, which we published as [open source](https://github.com/learnedsystems/SOSD) including all datasets and implementations, confirmed that learned indexes are indeed significantly smaller while providing similar or better performance than their traditional counterparts on real-world datasets.


Since the initial release of SOSD, we've made a few additions to the framework:

* Added new competitors ([ALEX](https://github.com/microsoft/ALEX) and a [C++ implementation](https://github.com/stoianmihail/CHT) of [HistTree](http://cidrdb.org/cidr2021/papers/cidr2021_paper20.pdf) contributed by Mihail Stoian).
* Added synthetic datasets as well as smaller (50M rows) datasets.
* Reduced overhead of benchmarking framework further.

While SOSD certainly filled the gap of a standardized benchmark, we feel that due to the sheer number of papers in this area, there's still a lot of discrepancies among experimental evaluations. This is mainly due to the fact that many implementations need to be tuned for the datasets and hardware at hand.

Today, we're happy to announce the Learned Indexing Game (LIG). LIG is an ongoing indexing benchmark on various synthetic and real-world datasets and is based on [SOSD](https://github.com/learnedsystems/SOSD). For each dataset, there are different size categories (e.g., M stands for an index size of up to 1% of the dataset size). We'll be using the **m5zn.metal** AWS instance type for the competition to ensure a common playing field.

We hope that LIG will receive contributions from the community (index implementations, datasets, and workloads) and can serve as a common benchmarking testbed. We'll post the link here in the next few days. Here's a first glance:

![LIG Screenshot](/assets/lig/screenshot.png)
