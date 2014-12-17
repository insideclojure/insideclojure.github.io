---
layout: post
date: 2014-12-17
title: Creating a distinct transducer
---

I wanted to talk in this post about implementing some new transducer impls on existing sequence functions which is based on [CLJ-1601](http://dev.clojure.org/jira/browse/CLJ-1601). 
*NOTE: As of the writing date, this talk has not yet been screened or incorporated into Clojure. It is still subject to change based on feedback during screening!!*

If you haven't checked into transducers yet, it's probably best if you take a look at [this talk](https://www.youtube.com/watch?v=6mTbuzafcII), [this talk](https://www.youtube.com/watch?v=4KqUvG8HPYo), and [this page](http://clojure.org/transducers) as background.

The ticket (based on a request from Rich) asked for new transducer arities for three existing sequence functions: sequence, interpose, and map-indexed. In this post we'll look specifically at created a transducer arity of distinct.

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
     (let [seen (java.util.HashSet.)]
       (fn
         ([] (rf))              ;; init arity
         ([result] (rf result)) ;; completion arity
         ([result input]        ;; reduction arity
           (if (.add seen input)
             (rf result input)
             result))))))
{% endhighlight %}

Just as in the sequence form of distinct, we need a set to keep track of what values we've already seen. Notice that distinct will return a (stateless) function. 
The set is created new every time a transducing process starts stepping with the composite reducing function in the transducer. 
Once this process starts, the transducing process must not hand out the composite reducing function as it is stateful!
All of the existing processes (transduce, core.async channels, etc) follow this rule.

Because the state is encapsulated inside the composite reducing function, we can use something stateful. 
One option is to use an atom or volatile to create a stateful identity - we'll see an example of those later. 
In this case, there happens to be a mutable stateful set already available to us: java.util.HashSet. 
Usage of this set will be safely published (as seen will be closed over in a final field of the inner compiled function) and only used by a single thread at a time, thus usage of an unsynchronized set is safe here.

The init and completion arities merely call through to the nested transducer, so nothing interesting on those. 
This is the default implementation for those transducer arities - just calling through to the wrapped reducing function.

The reduction arity is where the meat of the distinct is implemented.
We simply check whether the seen set contains the new input.
If so, we pass it on through to the wrapped reducing function.
If not, we don't need to loop as we did in the sequence - the outer transducing process will take care of that for us.
Instead, we simply return the accumulator value as is.
In other words, if the value has already been seen, we simply do not produce a value.

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

The first two (transduce and into) both eagerly produce a vector as the result (but you can see how to return a set or other data structure instead). 
The last one produces a sequence that will be computed element by element as needed.

If we compare the performance of into on the sequence and transducer versions, you can see it has a big impact (timings done with [Criterium](https://github.com/hugoduncan/criterium)):

expr | time 
----- | ----
(into [] (distinct v)) | 821.3 µs
(into [] (distinct) v) | 43.6 µs

I intentionally used a vector as the input source here (although a seq would have worked as well).
Inputs support the internal IReduce interface and can quickly reduce themselves by traversing their inner data structure rather than needing to realize a sequence element by element. 
Transducers return a vector as well (in the transduce and into examples above), allowing you to continue working in collections, which have this faster reduction capability.

While the distinct transducer will work with core.async channels, you should do this cautiously. 
Channels are transducible processes that can be used on arbitrarily lenghty streams of values and the internal "seen" set can thus grow without bound.
It would be possible to build a variant of distinct that had other properties, limiting the size of the seen values based on maximum size, least recently seen, or explicit clearing.

Clojure 1.7 also contains a new transducer and sequence function dedupe that crunches runs of repeated values down to a single value. The cache in this case is just a single value so it is much safer to use with channels.

In future posts, I'll talk about the other transducer functions in this ticket - interpose and map-indexed which have their own interesting implementations.