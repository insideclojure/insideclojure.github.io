---
layout: post
date: 2014-12-17
title: Creating a distinct transducer
---

I wanted to talk in this post about implementing some new transducer impls for existing sequence functions which is based on [CLJ-1601](http://dev.clojure.org/jira/browse/CLJ-1601). If you haven't checked into transducers yet, it's probably best if you take a look at [this talk](https://www.youtube.com/watch?v=6mTbuzafcII), [this talk](https://www.youtube.com/watch?v=4KqUvG8HPYo), and [this page](http://clojure.org/transducers) as background.

*NOTE: The changes described in this patch are included in Clojure 1.7.0-alpha5.*

The ticket (based on a request from Rich) asked for new transducer arities for three existing sequence functions: distinct, interpose, and map-indexed. In this post we'll look specifically at creating a transducer arity for distinct.

The sequence version of distinct is implemented like this:

{% highlight clojure %}
(defn distinct [coll]
   (let [step (fn step [xs seen]
                  (lazy-seq
                   ((fn [[f :as xs] seen]
                     (when-let [s (seq xs)]
                       (if (contains? seen f) 
                         (recur (rest s) seen)
                         (cons f (step (rest s) (conj seen f))))))
                    xs seen)))]
     (step coll #{})))
{% endhighlight %}

step is an internal function that takes the collection and a set of "seen" values. 
The implementation just invokes the helper step function with the collection and an empty set. 
The step function then creates a lazy sequence using the lazy-seq helper, which takes care of the details of producing a sequence element-by-element. 
The inner function invoked for each element will pull a value (f) from the source collection (xs). If f has already been seen, then we loop, continuing till we find a novel element. When a new element has been found we build the next step of the output sequence with cons, combining f (the element) and a new lazy sequence built on the step where f has been included in the seen values. 

Conceptually, we need to do something similar in the transducer function, however transducers are explicitly independent from both the input iteration and the output construction. I added a new transducer arity (no collection specified) as follows:

{% highlight clojure %}
(defn distinct []
   (fn [rf]
     (let [seen (volatile! #{})]
       (fn
         ([] (rf))              ;; init arity
         ([result] (rf result)) ;; completion arity
         ([result input]        ;; reduction arity
           (if (contains? @seen input)
             result
             (do (vswap! seen conj input)
                 (rf result input))))))))
{% endhighlight %}

Just as in the sequence form of distinct, we need a set to keep track of what values we've already seen. 
Notice that distinct will return a (stateless) function. 
The set is created anew every time a transducing process starts stepping with the composite reducing function in the transducer. 
Once this process starts, the transducing process must not hand out the composite reducing function as it is stateful!
All of the existing processes (transduce, core.async channels, etc) follow this rule.

Because we are not passing state along with the reducing function, we need some state on the side to track what we've seen.
We could use an atom, but we have a lighter-weight alternative now available.
Volatiles were adding in 1.7 and can be used to track a single safely published stateful value.
Volatiles are inherently more dangerous than atoms - while they safely publish changes to other threads they do not provide a means for coordinated read/write actions. 
Thus, volatiles are only safe to use if you can guarantee that something else (program semantics or external locking mechanism) will prevent race conditions.
In this case, the reducing function will only used by a single thread at a time, thus there is no race condition possible in this code.

The init and completion arities merely call through to the nested transducer, so nothing interesting on those. 
This is the default implementation for those transducer arities - just calling through to the wrapped reducing function.

The reduction arity is where the meat of the distinct is implemented.
If the set contains the new input, we don't need to loop as we did in the sequence - the outer transducing process will take care of that for us.
Instead, we simply return the accumulator value as is and let the outer transducible process keep pushing values into the transducer.
If the input is new, we pass it on through to the wrapped reducing function, which will will ultimately decide what to do with the output.

Now that we have an implementation, we can try it out. 
We'll start with a vec that contains the numbers 0-999, then 0-999 again. 
The sequence version simply wraps distinct around the sequence, producing a new sequence:

{% highlight clojure %}
(def v (vec (concat (range 1000) (range 1000))))
(distinct v)
{% endhighlight %}

There are several ways we could invoke a transducer to accomplish the same result: 

{% highlight clojure %}
(transduce (distinct) conj [] v)
(into [] (distinct) v)
(sequence (distinct) v)
{% endhighlight %}

The first two (transduce and into) both eagerly produce a vector as the result but you can see how to return a set, list, or other data structure instead.
This is one of the benefits of separating the output generation from the algorithmic reduction - we can produce any kind of output, not just a lazy sequence as in the sequence version which explicitly uses cons.
The last example produces a sequence that will be incrementally computed element by element as needed, similar to a lazy seq.

If we compare the performance of into on the sequence and transducer versions, you can see it has a big impact (timings done with [Criterium](https://github.com/hugoduncan/criterium)):

expr | time 
----- | ----
(into [] (distinct v)) | 821.3 µs
(into [] (distinct) v) | 388.2 µs

I intentionally used a vector as the input source here (although a seq would have worked as well).
Vectors support the internal IReduce interface and can quickly reduce themselves by traversing their inner data structure rather than needing to realize a sequence element by element. 
Transducers return a vector as well (in the transduce and into examples above), allowing you to continue working in collections, which have this faster reduction capability.

While the distinct transducer will work with core.async channels, you should use some caution with this particular transducer.
Channels are transducible processes that can be used on arbitrarily lengthy streams of values and the internal "seen" set can thus grow without bound, potentially causing problems.
It would be possible to build a variant of distinct that limited the size of the seen values based on maximum size, least recently seen, or some other clearing mechanism.

Clojure 1.7 also contains a new transducer and sequence function dedupe that crunches runs of repeated values down to a single value.
The cache in that case is just a single value so it is much safer to use with channels.

In future posts, I'll talk about the other transducer functions in this ticket (interpose and map-indexed) which have their own interesting implementations.

**NOTE: Thanks to Christophe Grand for pointing out a bug in the implementation used in the original version of this post. The patch on the ticket and this post have been updated to reflect the changes.**
