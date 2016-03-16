---
layout: post
date: 2016-03-16
title: Inside Clojure's Collection Model
---

What are the Clojure collection types? Most Clojurists would say lists, vectors, maps, and sets are all you need. You might add seqs too - they are collection abstractions (views) which can be backed by a collection, but might also be backed by something else. (See [Sequences](http://insideclojure.org/2015/01/02/sequences/) for lots more on this.)

But in Clojure there are also sorted maps, sorted sets, queues, primitive vectors and support for Java arrays, Java collections and more, not to mention custom types that you might create yourself using deftype. Things seem a bit messier now. And the answers to common predicates like `sequential?`, `seq?`, `coll?` may give answers that you find confusing for some of these collection types. 

If you look a bit deeper, you'll find that there is more to the collection model than is immediately apparent. Clojure provides a few different layers that form the foundation for what you use on a daily basis. The neatest part is that almost nothing in Clojure (compiler, runtime, or standard lib) actually depends directly on collection *implementations*, rather everything depends on abstractions that are open to extension. As a beginning Clojurist, you rarely tap into these extensions, but you will find them increasingly useful in more sophisticated programs.

## Traits and Predicates

Perhaps the most important layer in the collection library are the *traits* (my term). Ideally traits would be defined as Clojure protocols, but their implementation dates back to the earliest days of Clojure, well before protocols existed and as such they are defined by Java interfaces.

Some of the key trait classes (all in `clojure.lang`) are:

* `Counted` - countable collections
  * `count()`
* `Indexed` - extends `Counted`, allows index-based lookup
  * `nth(int i)`
	* `nth(int i, Object notFound)`
* `Sequential` - marker interface for sequential collections
* `Associative` (extends `ILookup`)
  * `containsKey(Object key)`
	* `entryAt(Object key)`
	* `assoc(Object key, Object val)`
	* via parent `ILookup`:
	  * `valAt(Object key)`
		* `valAt(Object key, Object notFound)`
* `Sorted` - marker interface for sorted
* `Seqable` - for collections that can produce a sequence
  * `seq()`
* `Reversible` - for collections that can produce a reversed sequence
  * `rseq()`

Each of these interfaces describes a very narrow slice of capability that a collection might have. You'll find that most of the Clojure predicates are simply checks for whether a class implements one of these interfaces:

* `counted?` - checks whether instance of `Counted`
* `indexed?` - checks whether instance of `Indexed`
* `sequential?` - checks whether instance of `Sequential`
* `associative?` - checks whether instance of `Associative`
* `sorted?` - checks whether instance of `Sorted`
* `reversible?` - checks whether instance of `Reversible`

The ones missing above are `ILookup` and `Seqable`. `ILookup` is really a narrow subset of `Associative` so that's the more common predicate. `Seqable` is kind of tricky as Clojure goes some distance to patch "seq-ability" into things not otherwise seqable, like strings and arrays. Again, if Clojure had protocols at the time of its creation, a "seqable" protocol could be extended to closed types like these directly. See the [sequences post](http://insideclojure.org/2015/01/02/sequences/) for more on `seqable?`.

The important thing to note here is that most of the collection functions in clojure.core or clojure.lang.RT (the Clojure runtime) operate in terms of these traits, NOT in terms even of collection interfaces or ever in terms of concrete collection types. Tracing this requires a bit of hopping. Most of the standard collection functions (`count`, `nth`, `get`, `assoc`, `seq`) have obvious mappings to the interface methods above. 

If you look at the source for most Clojure core functions (like `count`), you'll see that they largely just forward to a method on the `RT` class. If you then look at RT.count(), you'll see that most of these invoke the method above on the Java interface as their first implementation (with some backups to alternate implementations for special cases). Again, protocols would have made this cleaner.

The benefit of this approach is that you can create a new type that implements any set of these traits, and it will work with the existing Clojure library with no changes to Clojure!

## Collections

The core Clojure collections are also implemented first in terms of interfaces. Most of these interfaces primarily define the set of traits a persistent collection type should include. (I'm glossing over the details a bit because how this happens via the class hierarchy is a little confusing.) A good example though is IPersistentVector, which extends `Associative`, `Sequential`, `Reversible`, and `Indexed`. There are similar interfaces for `IPersistentSet`, `IPersistentMap`, `IPersistentList`, and a higher level interface, `IPersistentCollection`. I'm not going to talk about it, but for completeness, there is also `IPersistentStack` which is visible in clojure.core via `peek` and `pop`.

These interfaces also have corresponding predicates that just check the type: `coll?`, `list?`, `vector?`, `set?`, and `map?`. And similarly, functions in the core library or runtime never check a concrete type and only depend on these interfaces (other than for the purposes of needing to construct a concrete instance).

Finally, there are a number of concrete implementations (actually several per type):

* `IPersistentList`
	* `PersistentList` - typical list
	* `PersistentList$EmptyList` - special-cased empty list
	* `PersistentQueue` - a queue implementation
* `IPersistentVector`
	* `PersistentVector` - typical vector
	* `MapEntry` - a map entry acts as a 2-element persistent vector
	* `SubVector` - created via `subvec`, really a view over a source vector
	* `clojure.core/Vec` - the primitive vector implementation
* `IPersistentSet`
	* `PersistentHashSet` - typical set
	* `PersistentTreeSet` - sorted set
* `IPersistentMap`
	* `PersistentArrayMap` - array-based map used for 0-8 pairs
	* `PersistentHashMap` - typical hash map
	* `PersistentTreeMap` - sorted map
	* `PersistentStructMap` - from `defstruct` (effectively deprecated)
	* all defrecords produce instances of `IPersistentMap`

But what about sequences - how do they play in here? This is where things get a little tricky.

## Sequences

Sequences represent a *logical* immutable collection. When sequences are obtained from collections they are backed by the data in the collection (I think of them as "views" of the collection) but they can also be generated from a function on demand or obtained from a variety of other sources (strings, arrays, iterators, etc).

The important thing though is that sequences are again defined by interfaces in the Java implementation:

* `ISeq` - the sequence abstraction, checked via the `seq?` predicate
* `Seqable` (mentioned above) - something that can produce a sequence

Here's the twist: `ISeq` extends `IPersistentCollection`. So sequences are also treated as (virtual) collections, but *not* treated as `IPersistentList`s. This can be confusing because both sequences and lists print with parentheses as if they are lists.

Just like with collections, there are lots of sequence implementations:

* `LazySeq` - a lazy sequence
* `Cons` - the result of a `cons` call
* `ChunkedCons` - the result of a chunked lazy sequence like `map` or `filter`
* `ArraySeq` - sequence over an array
* `StringSeq` - sequence over a string
* and many, many more!

## Equality

Clojure collections define equality based on a set of equality partitions: sequentials, maps, and sets. Equality is always based on a value comparison - is the collection in the same partition, and does the collection contain the same values? 

This is a place where Clojure departs from many other languages. Clojure's collections are 1) immutable and 2) check equality by value. These factors allow you to treat Clojure's collections as composite values, and retaining all the valuable properties of immutable simple values (like numbers, keywords, etc).

Sequential collections (lists, vectors, sequences, and anything marked sequential) are compared from beginning to end, element by element for equality. Sequential collections *do not* include collection type as a comparison criteria. 

This is occasionally not what you want (for example, when testing a return value). However, making a single partition for all sequential collections is one of the great (pragmatic) choices in Clojure for improving ease of use, particularly in blurring the lines between sequences and collections.

Sets are equal if they contain exactly the same set of elements with no extra elements in either set. [By "same", I mean "= returns true", not an identity comparison.]

Maps are equal if they contain exactly the same set of keys and each key returns the same element.

## Wrapping up

Hopefully this provided a good overview of how the Clojure collections are implemented under the hood and integrated with the Clojure runtime via abstractions. Those abstractions are also available for you to plug into, and perhaps that's a good subject for a follow-up post.