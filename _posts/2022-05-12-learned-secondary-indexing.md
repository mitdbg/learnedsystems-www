---
layout: post
title: "Announcing LSI: A Learned Secondary Index Structure"
image: "/assets/lsi/overview.png"
---

_Authors: [Dominik Horn](https://www.linkedin.com/in/dominik-horn-9b9187220/),
[Andreas Kipf](https://people.csail.mit.edu/kipf/),
[Pascal Pfeil](https://www.linkedin.com/in/pascalpfeil/)_

<!--[Ryan Marcus](https://rmarcus.info/blog/),
and [Tim Kraska](https://people.csail.mit.edu/kraska/)_-->

We are proud to announce our new publication [LSI: A Learned Secondary Index
Structure](https://doi.org/10.1145/3533702.3534912) and accompanying
[**MIT licensed open source C++ implementation**](https://github.com/learnedsystems/LearnedSecondaryIndex), that
can easily be included in other projects using
[CMake FetchContent](https://cmake.org/cmake/help/latest/module/FetchContent.html).
LSI is the first of its kind learned secondary index. It offers competitive
lookup performance on real world datasets while reducing space utilization up
to 6x compared to state-of-the-art secondary index stuctures.

### Motivation

Most research concerning [learned index structures](https://dl.acm.org/doi/pdf/10.1145/3183713.3196909)
focuses on primary indexing, where, crucially, base data is stored in sorted
order. However, this assumption prevents directly utilizing existing approaches
for secondary indexing. It is therefore of high interest to study the
feasibility of applying learned techniques in this setting. Specifically, we
were aiming to come up with an off-the-shelf usable index structure that
manages to retain the [proven benefits of learned primary indices](https://learnedsystems.github.io/SOSDLeaderboard/leaderboard/)
as best as possible.

<!-- TODO: find better title -->

## LSI Explanation

LSI is comprised of two parts. A pre-existing learned index structure acting as
an approximate index, e.g., a [PLEX](https://arxiv.org/abs/2108.05117), and a
permutation vector. The learned index is trained over a temporary sorted copy
of the base data's keys. The permutation vector retains a mapping from the
sorted key's positions to offsets into the base data, e.g., tuple IDs. The
following paragraph contains a schematic illustration of LSI. A more detailed
explanation, especially concerning how we achieved LSI's space efficiency,
may be found in our [paper](https://doi.org/10.1145/3533702.3534912).

### Lookups

Querying LSI works in three steps, as illustrated below:

![LSI Architecture Overview](/assets/lsi/overview.png)

1. Query the learned index for an approximate position in the sorted base data.
   Again, note that the sorted representation is not actually stored anywhere
   and only exists temporarily during construction.
2. Obtain search bounds based on the approximate position and the learned
   index's error bounds. These can either be guaranteed by construction, as is
   the case with [PLEX](https://arxiv.org/abs/2108.05117), or could be measured
   and retained during training.
3. Conduct a local search over the permutation vector to locate the key.
   - **Lower Bound Lookups** perform a binary search. Note that this requires
     `O(log k)` probes into the base data for a search interval of size k. This
     could potentially be prohibitively costly in practice, e.g., when base
     data is not fully loaded into main memory.
   - **Equality Lookups** can be sped up by retaining fingerprint bits for each
     key and performing a linear scan over the search interval. This requires
     `O(k)` base data probes for a search interval of size k. However, each
     additional fingerprint bit roughly halves the actually required accesses.

## Evaluation

Evaluations were conducted on a `c5.9xlarge` AWS machine with 36vCPUs and 72GiB
of RAM using four real-world datasets from
[SOSD](https://arxiv.org/abs/1911.13014), each with 200 million 64-bit unsigned
integer keys:

- `amzn` book popularity data
- `fb` randomly sampled Facebook user IDs
- `osm` cell IDs from Open Street Map
- `wiki` timestamps of edits from Wikipedia

All experiments ran single threaded. To prevent skew from potential
out-of-order execution, we insert a full memory fence using the compiler's
built in `__sync_synchronize()` after each lookup. A more detailed writeup of
our findings may be found in our [paper](https://doi.org/10.1145/3533702.3534912).

### Lower Bound Lookups

To benchmark lower bound lookups, we uniform randomly remove 10% of the
elements from the datasets before building each index. We only query for
non-keys.
LSI with [PLEX](https://arxiv.org/abs/2108.05117) as its approximate index is
competitive with [ART](https://db.in.tum.de/~leis/papers/ART.pdf)'s latency,
while only taking up 15% of the storage. It is twice as fast as
[BTree](https://github.com/bingmann/stx-btree) while reducing space consumption
by 50%.

<img src="/assets/lsi/lowerbounds.png" style="width:66%;display:block;margin-left:auto;margin-right:auto;" alt="lowerbounds benchmark on all four datasets" />

Note that BTree was constructed by first sorting the data and bulk
loading afterwards. This enables denser nodes and therefore faster lookups.
Inserting keys in their uniform random order one after another increased space
usage by roughly 33% and slowed down lookups by more than 20% compared to the
datapoint shown. The text annotations denote the LSI model's error bound.

### Equality Lookups

Although LSI with [PLEX](https://arxiv.org/abs/2108.05117) as its approximate
index is around 1.5 times slower than the robin-hood hash table implementation
[RobinMap](https://github.com/Tessil/robin-map), it consumes less than a
quarter of the space. The text annotations again denote the LSI model's error
bound. Note that we used the `amzn` dataset for this test, only to later
discover that RobinMap's hash function appears to be unfavourably biased in
this case. On `fb` and `osm`, RobinMap's lookup only takes 340ns compared to
the 440ns on books. Space consumption does not change. We do not have equality
lookup numbers for LSI on other datasets at the moment of writing.

<img src="/assets/lsi/equality.png" style="width:45%;display:block;margin-left:auto;margin-right:auto;" alt="equality benchmark on amzn"/>

### Build time

We measured the true end to end build time for each index given an unsorted
continuous in-memory representation of each dataset's 200 million keys. LSI is
competitive in terms of build time. It fairs slightly worse than BTree, but
surpasses [ART](https://db.in.tum.de/~leis/papers/ART.pdf) on every dataset we
tested. As expected, lower error bounds on the approximate index increase build
time.

[BTree](https://github.com/bingmann/stx-btree) is constructed by first sorting
and the bulk loading all keys. This is about 8 times faster than inserting keys
in unsorted order one by one. ART does not have a bulk loading interface,
however its construction time is still cut in half when keys are first sorted
and then inserted one after another. We suspect this effect is primarily caused
by the better cache coherency during construction.

<img src="/assets/lsi/build_throughput.png" style="width:66%;display:block;margin-left:auto;margin-right:auto;" alt="build time comparison on all four datasets" />

Note that we would have expected
[RobinMap](https://github.com/Tessil/robin-map) to exhibit equal build
throughput on `amzn`, `fb` and `osm`. Further investigation is required to
determine why it fairs so poorly on `amzn`. We suspect that the hash function,
which we have no control over, might be unfavourably biased in this case. More
than half of all elements from `wiki` are duplicates. For this reason, build
times for RobinMap on this dataset appear faster.

### Fingerprint configuration

We conducted a micro experiment to study the trade off between binary search
and linear search with fingerprints. For this purpose, we trained LSI with
[PLEX](https://arxiv.org/abs/2108.05117) as an approximate index on `amzn`
using various errors and fingerprint bit sizes (denoted in brackets after
name), and compared lookup latencies:

<img src="/assets/lsi/binary_vs_linear.png" style="width:47%;" alt="micro experiment as line plot" />
<img src="/assets/lsi/error_fingerprint_study_cpu_time.png" style="width:52%;" alt="micro experiment as heat map" />

For a search bound of size `k`, binary search always performs `O(log k)` probes
into base data, while linear search requires `O(k)`. Each additional
fingerprint bit will roughly halve the neccessary base data accesses. To better
quantify linear search with fingerprints, we measured the amount of relative
false positive accesses. A false positive access occures when base data is
probed during a linear scan but the search key is not actually found. For our
visualization, we normalized the measured absolute amount of false positives
against the search range size, i.e., 2x the model's error bound:

<img src="/assets/lsi/error_fingerprint_study_normalized_fp.png" style="width:50%;display:block;margin-left:auto;margin-right:auto;" alt="" />

## Future Work

We are excited about our promising first results of approaching secondary
indexing using learned techniques. For production readiness, we will however
most likely need to implement inserts next. This will require two main
adjustments. Firstly, the approximate index has to be updateable. We could
either attempt to extend [PLEX](https://arxiv.org/abs/2108.05117) with support
for inserts, or use an existing of the shelf model that supports this like
[PGM](https://dl.acm.org/doi/abs/10.14778/3389133.3389135). Additionally, we
will have to find a better way to deal with dynamic updates to the permutation
vector. As implemented currently, each update will require shifting and
incrementing values, or `O(n)` time.

Another interesting direction is to try and reduce base data accesses during
lower bound lookups. While we could just store a copy of all keys in the
permutation vector, similar to BTree's leaf layer, this will negate most of
LSI's space savings. However, there might be an opportunity to store just
enough information to reconstruct enough information about the keys to enable
efficient binary searching or similar. Stay tuned for updates!
