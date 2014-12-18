---
layout: post
date: 2014-12-18
title: Creating an interpose transducer
---

Continuing from yesterday's [post](/2014/12/17/distinct-transducer/), I wanted to cover the interpose transducer from [CLJ-1601](http://dev.clojure.org/jira/browse/CLJ-1601).

The sequence version of interpose is a straightforward combination of other existing sequence functions (repeat, interleave, and drop). 
It's implemented like this:

{% highlight clojure %}
 (defn interpose [sep coll]
 	(drop 1 (interleave (repeat sep) coll)))
{% endhighlight %}

Walking inside out, (repeat seq) will create an infinite sequence of separator strings.
(interleave (repeat sep) coll) will interleave the infinite separate seq and the collection (possibly also an infinite sequence!) like this:

~~~
sep elem0 sep elem1 sep elem2
~~~

And finally the (drop 1) loses the first separator which is unnecessary. 

In the transducer version, I chose to use a volatile to store a flag about whether this was the first input element. 
In the case of the first input element, we simply update the flag and invoke the inner reducing function on the first element.
This effectively does the "drop 1" behavior of the sequence implementation.
Forever after, we invoke the reducing function on both the separator and then on the element:

{% highlight clojure %}
(defn interpose
  ([sep]
   (fn [rf]
     (let [started (volatile! false)]
       (fn
         ([] (rf))
         ([result] (rf result))
         ([result input]
          (if @started
            (let [sepr (rf result sep)]
              (if (reduced? sepr)
                sepr
                (rf sepr input)))
            (do
              (vreset! started true)
              (rf result input))))))))
{% endhighlight %}

As with distinct, the started flag is a volatile created once per transducing process and the real business happens in the reducing arity.

One issue that we need to deal with is being aware of reduced values.
The calls to rf on the input are fine - they may return a reduced value that will be dealt with at a higher level (ultimately the transducing process itself).
The special case is when a reduced value is returned from the separator.
An example where this could happen would be:

{% highlight clojure %}
(into [] (comp (interpose :x) (take 4)) (range 3))
{% endhighlight %}

The (range 3) produces sequence (0 1 2).
The interpose should produce (0 :x 1 :x 2).
The take should then grab just (0 :x 1 :x) and the reduced wrapper will be sent on a separator (:x).

So in the transducer code, we need to check if we've already encountered a reduced value when we invoke rf on the sepr, and if so stop and return the reduced value without invoking on the next input.

That's it! Quick performance comparison:

| expr | time |
| ----- | ---- |
| (into [] (interpose nil v))  | 316.0 µs |
| (into [] (interpose nil) v)  | 35.5 µs |

This code has not yet been screened or added to 1.7, but I expect that it will be.
