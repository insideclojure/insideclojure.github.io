---
layout: post
date: 2022-06-15
title: Efficient partitioning
---

Certain tasks call for breaking up sequential data into segments to work on each segment. In Clojure, we often use functions like `partition-all` and `partition` to break sequential data (or code) into a lazy sequence of realized partitions that you can consume as needed. 

For example, a common use case in macros that parse syntax is to `(partition 2 ...)` over key-value pairs in binding or destructuring forms.

The realization of the lazy seq provided by partition is out of the consumer's control, so you might also see a less encapsulated version of partitioning that `take`s and then recurs or come back to do more work later on the `drop` of the sequence. The core function `split-at` is a common helper for that.

## Take and drop

Taking a closer look at what's happening, we see there are two parts of this job: retrieving the first N items for the next partition, and dropping the first N items to get the "rest" of the collection. All of the functions mentioned above work on sequences, which are logical linked lists and do this work by traversing the sequence item by item. The `take` returns the retrieved items but has no way to return the "rest" of the tail, so the `drop` (or `nthrest`) must again sequentially walk the first N items to find the tail.

Considering just the problem of efficiently building the partition, these functions have not been reconsidered since Clojure added the new capabilities of transducers and self-reducible collections. Using a `take` transducer will generally be faster than the seq alternative, especially when the underlying collection is self-reducible (like vectors, maps, ranges, etc). Transducers are faster here for several reasons. First there's no need to construct the chain of immutable cons-like cells (good for immutability but unnecessary cost when you are constructing, then garbage collecting those cells). Second, if the source is self-reducible, the iteration is more efficient, often a tight loop in the collection. Third, transducers don't produce an intermediate output, they are given a function that can directly accumulate the output. And finally, that makes it easy to use transient collections to create the partition batch collections.

Fortunately, all of these benefits are already built into an existing function: `into`. Switching the partition building from `(take n coll)` to `(into [] (take n) coll)` is a small change with a lot of benefits. One important difference here is that `into` will produce vector partitions, rather than realized seqs.

But what about moving past the first partition to get a reference to the tail of the original collection? This job is typically done with `drop` or `nthrest`. Using a `drop` transducer helps if the collection is self reducible, but what we get at the end is typically a lazy seq, not the original reducible collection, and thus this only helps on the first partition, later ones devolve into seq walking.

In thinking about this problem we came up with two primary alternatives - relying more on iterators and being able to retain their connection to the original collection OR introducing a path to faster `drop` when that's available. 

## The iterator approach

To explore the first one a little bit, the idea here is that if you partition a collection like `(vec (range 1000))`, you'll start with a PersistentVector. Iterators are stateful and generally not thread-safe for concurrency, but in constrained usage, they can be helpful. If PV can produce an iterator that retains it's connection to the underlying vector, it can iterate efficiently, and when iteration is complete, the iterator has enough information to be converted back into a sequence (specifically the sequence impl inside PersistentVector).

Note that this approach is not generic - it's limited to persistent collections and algorithmically computed side-effect free collections (notably `range` and `repeat`). But for those use cases, this worked fairly well. Changes were required at several points to create better iterators (not using the default seq iterator) and to enhance the ability of `iterator-seq` to detect and use this capability.

In the end though, the places where we could take advantage of this were fairly limited, and the reliance on stateful iterators was both gross and not as efficient as it could be as it still required enumeration of all dropped values.

## IDrop

We ended up creating a new interface that allows a collection to mark itself as being able to efficiently (and safely) drop N items at a time. This should only be implemented in collections that are either backed by immutable data (effectively moving a pointer around) or algorithmically computed (to support a "jump"). 

The interface for IDrop is fairly simple:

```
Sequential drop(int n);
```

This interface commits to as little as possible but the docs state that the returned type should be Sequential, ISeq, and IReduceInit. 

We made some enhancements to implement IDrop for the following types:

* `PersistentVector` and its sequence (vectors)
* `PersistentArrayMap` and its sequence (array maps)
* `LongRange` (`range`)
* `Repeat` (`repeat`)
* `StringSeq` (used by seqs over strings)

To take advantage of this in the core, we changed `drop`, `nthrest`, and `nthnext` to now check whether the collection implements IDrop and use that if so (otherwise falling back to the prior behavior). As a result of that, anything using these will also see the effects (this includes functions like `partition`, `partition-all`, and `split-at`).

However, the partition functions currently return a lazy seq of realized seq partitions - to fully get the benefits of the scheme above on the take side we want to create partitions using a take transducer into a transient vector. It was easy to find cases where code used the partition functions to generate/manipulate code, which won't work with vectors, so we really can't change the return types here without violating expectations.

Instead, we've added new `partitionv`, `partitionv-all`, and `splitv-at` functions that instead return vector partitions. For data use cases, where returning ISeq partitions doesn't matter, these are going to be a new faster option.

## Performance

There was a lot of performance testing done but some highlights here (tests run using Java 11, best time of large set of trials reported) and the changes in [CLJ-2713](https://clojure.atlassian.net/browse/CLJ-2713).

{% highlight clojure %}
(dotimes [_ 1000]
  (doall
    (f 100 coll))))
{% endhighlight %}

coll | 1.11.1 | CLJ-2713 | CLJ-2713
(count=10k) | f=partition-all (ms) | f=partition-all (ms) | f=partitionv-all (ms)
----- | ---- | ---- | ----
range | 561 | 518 | 217
repeat | 692 | 490 | 208
vector | 756 | 612 | 239
string | 782 | 538 | 247
array-map | 795 | 512 | 260
seq | 790 | 672 | 385

The latter two columns show that `partition-all` times improved - this is mostly the `IDrop` improvements. But the `partitionv-all` times show all the benfits from that plus using the take transducer, especially on reducible collections.

## LongRange revisited

One side path in the changes above is that the `partition-all` times for ranges (really the special case `LongRange` which is optimized for long start/step/end) was initially slower for `partition-all`, and that was particularly magnified at small partition sizes.

What was happening in that case is the `IDrop` optimization would skip the range forward, but the chunk caching being done by LongRange would then get lost, and chunks were being computed multiple times during the take phases, such that the overall perf was worse (it was still better in the `partitionv-all` case).

This led me ultimately into looking at a lot of the complexity in LongRange, which I implemented way back in Clojure 1.7 in the context of [CLJ-1515](https://clojure.atlassian.net/browse/CLJ-1515). There were a couple concerns driving things - the bounds checking was pretty complicated to specifically handle the case of long ranges with either large counts (> Integer/MAX_VALUE) or pathological inputs that could cause overflow. And the chunk caching was the result of several late sets of changes in CLJ-1515 that simplified chunks to calculate rather than pre-compute values, but did not fully take advantage of that optimization. 

I addressed both of these complexity sources by deleting a lot of code, the best way to address complexity. :) For the bounds checking, these are cases that are rare and easy to avoid by checking during LongRange construction and falling back to our slow path Range class instead (which already handles these cases). For the chunk caching, I simplified to just use exactly one chunk to match the entire range.

The upside of these changes is that LongRange is greatly simplified and is now faster for both seq and reduce based traversal. One downside is that the fields of LongRange changed and in 1.12, it will not be serialization-compatible. We've never guaranteed that (and have regularly broken serializability over the history of Clojure), but it's something we are newly aware of due to some unintentional effects in 1.11. Our intent going forward is to not break serialization when possible by retaining serialVersionUIDs (we have a separate [issue](https://clojure.atlassian.net/browse/CLJ-1327) to deal with this) and to be explicit when serialization does change.

## Plans

So anyways, this is all headed towards a 1.12 alpha soon and we look forward to seeing your feedback.
