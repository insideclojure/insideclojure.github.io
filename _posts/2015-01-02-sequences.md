---
layout: post
date: 2015-01-02
title: Sequences
---

Sequences are the key abstraction that connects two of the most important parts of Clojure - immutable persistent collections and the sequence library. Lisp has a deep tradition of list-oriented transformation functions. In Clojure, these functions have been separated from the specific linked list data structure and built instead upon the sequence abstraction. Sequences are *logical* lists that can be sequentially traversed.

The key sequence abstraction functions are:

- `(first coll)` - return the first element
- `(rest coll)` - returns a logical collection of the rest of elements (not necessarily a seq). Never returns nil. 
- `(next coll)` - returns a seq of the rest of the elements, which will be nil if no elements remain.
- `(cons item seq)` - construct a new sequence with the item prepended to seq

Sequences are traversed by a series of `first` and `rest` or `next` calls. As elements of the sequence are realized, they are cached. Sequences are immutable unfolding views of a source (whether a collection or something else).

`rest` and `next` are importantly different in their usage. `rest` returns a logical collection (possibly a sequence, possibly not). This allows some sequences to be lazier about forcing the evaluation of the next element before it's needed. As the lazier choice, `rest` is usually preferred when traversing a sequence. 

`next` may evaluate the next element and will return either nil or a sequence containing at least one element. `(next s)` is equivalent to `(seq (rest s))`.

## Terminology and empty sequences

There is some very careful language used in the oldest docs and docstrings around the terminology of *sequence*, *seq*, etc. As far as I read it, a collection produces a *seq*, a `lazy-seq` returns a *lazy-seq*, and a *sequence* is either a *seq* or a *lazy-seq*. This distinction was (at one point) reflected in the underlying Java interfaces, however that no longer exists and modern usage treats *seq* and *sequence* as interchangeable terms.

One related and confusing aspect of sequences centers around the representation of a sequence of no elements. A *seq* (or the result of calling `seq` or `next`) will always be either nil or a sequence with at least one element. The result of calling `sequence` or `rest` will represent an empty sequence as a non-nil empty sequence, which may be an empty list, an unrealized empty lazy sequence, or something else. 

Depending on context, either nil or an empty sequence may be the expected representation.

## Seqable

Collections or other things that can produce a sequence are *seqable*. Calling `seq` on a seqable will return either nil or a sequence with at least one element. Because of this behavior, it is idiomatic to use `seq` as a terminating condition check (aka nil-punning) while iterating a sequence:

{% highlight clojure %}
(let [s (seq coll)]
  (if s
    (comment "work on (first s) and recur on (rest s)")
    (comment "all done - terminate")))
{% endhighlight %}

Sequence functions (map, filter, etc) implicitly call `seq` on the incoming (seqable) collection and return a sequence (possibly empty, not nil). You must use `seq` on the result to nil pun.

{% highlight clojure %}
user=> (filter odd? [1 2 3]) ;; calls seq on the vector
(1 3)                        ;; returns lazy sequence
user=> (filter odd? [2 4 6]) ;; calls seq on the vector
()                           ;; returns empty lazy sequence
{% endhighlight %}

## Immutability

Because sequences are immutable, not stateful, they are safe to pass between threads, use in reference types, and use almost anywhere you would use persistent collections. Being able to blur the lines between use of collections and sequences is an important part of Clojure usability where things usually "just work".

One place where the rule of immutability is bent is when working with Java interop. Sequences built on Java iterators (which are stateful) will be affected by the stateful nature of the source. If the iterator source is changing, all the normal iterator rules will apply depending on the iterator type, including the possibility of surfacing a `ConcurrentModificationException` or missing values added during sequence traversal for concurrent iterators. 

{% highlight clojure %}
(let [x (doto (java.util.ArrayList.)
              (.add 1)
			  (.add 2))
	  is (iterator-seq (.iterator x))]
  (println (first is))   ;; 0
  (.add x 3)
  (println (first (rest is))))
;; ConcurrentModificationException !!
{% endhighlight %}

Sequences built on Java arrays are subject to mutability and do not separately cache values for performance. If you wish to create a safer sequence on a Java array, you must copy it first.

For example, as a Clojure programmer you should find it deeply disturbing to get different answers to calling first on a sequence over a mutated array:

{% highlight clojure %}
(let [arr (long-array (range 5))
      as (seq arr)]
  (println (first as))    ;; 0
  (aset arr 0 99)
  (println (first as)))   ;; 99 !!
{% endhighlight %}

So don't do that. If your code fully controls the Java array backing the sequence, this is not an issue. If you need to protect against array mutation, copy the array into a new array instead:

{% highlight clojure %}
(let [arr (long-array (range 5))
      copy (into-array Long/TYPE arr)
      as (seq copy)]
  (println (first as))   ;; 0
  (aset arr 0 99)
  (println (first as)))   ;; 0 as expected
{% endhighlight %}

## Laziness 

Because sequences are only evaluated an element at a time, they can represent an infinite stream of computed (or extracted) values. Lazy sequences can be created with the [lazy-seq](http://clojure.github.io/clojure/clojure.core-api.html#clojure.core%2Flazy-seq) macro, which takes a body that yields a seq, nil, or anything seqable. 

For example, a simplified version of map could be written:

{% highlight clojure %}
(defn map [f coll]
   (lazy-seq
    (when-let [s (seq coll)]
      (cons (f (first s)) (map f (rest s))))))
{% endhighlight %}

Here, `seq` is called on the incoming coll, which will return either nil (in which case nil is returned, terminating the lazy-seq) or a sequence with at least one element. We return a new sequence built using `cons` on the mapped first element and a a nested call to `map`. This looks like recursion, but as `lazy-seq` is a macro, this is really an unfolding that occurs in separate stacks as the seq evaluates, not recursion.

One particular issue with `lazy-seq` is that the body of the lazy-seq can close over locals in some cases, causing the entire lazy sequence to inadvertently be held in memory, even when no users of the lazy seq are retaining a reference to the sequence "head". This is called "head-holding". 

The `lazy-seq` macro specfically addresses this by clearing the locals in the tail call of the body. Because seqs cache their values, the body will only be invoked once per evaluation and clearing locals is safe.

Many of the lazy sequence functions in Clojure amortize the realization of elements by producing a "chunk" of N elements at a time. Chunking is an interesting (and very effective) optimization that is woven throughout various important parts of Clojure. I'm going to avoid talking about it further here though and leave that for a future post.

## Java perspective

Clojure is implemented in both Java and Clojure. Many of the most important sequence interfaces and implementations are implemented in Java:

- `clojure.lang.ISeq` - the sequence abstraction interface
- `clojure.lang.ASeq` - an abstract sequence implementation for easier implementations
- `clojure.lang.LazySeq` - lazy sequence implementation
- `clojure.lang.Seqable` - seqable marker
- `clojure.lang.Sequential` - a collection trait indicating whether a collection is sequential (an ordered series of values). Lists, vectors, and effectively all seqs are sequential.

The Clojure list implementation (`clojure.lang.PersistentList`) is a list data structure and thus is both a concrete data structure and implements the ISeq abstraction directly. All of the other collections are Seqable, but not ISeq.

## Predicates and functions

Some important other sequence predicates:

- `seq?` - checks whether an instance implements ISeq
- `sequential?` - checks whether an instance implements Sequential

There are no `seqable?` or `sequence?` predicates.

## Reference

A few important reference pages:

- [Sequences](http://clojure.org/sequences) - the main sequence documentation
- [Lazy](http://clojure.org/lazy) - a historical document at the time of the last major overhaul of Clojure sequences

