---
layout: post
date: 2015-01-27
title: Clojure Tip - "contains?"
---

There are a number of questions and gotchas that come up over and over in Clojure forums. I thought it might be useful to write some of them up. I initially planned to write them up in a single post. But then the first one was long enough to be a whole entry.

Often one of the first gotchas many Clojure programmers encounter is the `contains?` function. It sounds like it checks whether a data structure contains a value:

{% highlight clojure %}
=> (contains? [3 4 5] 0)
true   ;; wat
=> (contains? [3 4 5] 3)
false  ;; wat
{% endhighlight %}

The gotcha here is that `contains?` checks whether an associative data structure contains a *key*. Vectors are associative from index to element so the first question above is asking: does the vector have an index 0 (yes, it does). The second asks whether it has an index 3 (nope, just 0,1,2). 

`contains?` on a map checks whether the map has a key: 

{% highlight clojure %}
=> (contains? {:a 1} :a)
true
=> (contains? {:a 1} :GRUE)
false  ;; whew
{% endhighlight %}

`contains?` on a set confusingly looks like it is a value check at first: 

{% highlight clojure %}
=> (contains? #{:a :b :c} :a)
true
{% endhighlight %}

However, sets are hashed, just like the keys in a map. So, it really works exactly the same as a map.

`contains?` is for fast (constant or log time) lookups, not linear time lookups. If you need fast lookups in your algorithm, then you should use a data structure that supports it (like sets and maps). 

In general, when creating generic operations on data structures you can choose to be loose in how they work regardless of performance or strict in only allowing data structures that can meet performance expectations participate in an operation. Clojure strongly prefers the latter approach and functions in the core library often discuss performance expectations.

## Sequential search

How do you actually do a linear search in a sequential collection? The most common recommendation is to use `some` which takes a predicate and a sequential collection. Then use a set, invoked as a function, as the predicate:

{% highlight clojure %}
=> (some #{0} [3 4 5])
nil
=> (some #{3} [3 4 5])
3
{% endhighlight %}

Each element (3, 4, 5) is invoked on the predicate (in this case, a set acting as a function). If none matches, nil (a falsey value) is returned. If one does match, checking stops and the matched value is returned (a truthy value).

If you find the falsey/truthy values of nil (falsey) and 3 (truthy) results to be different than what you want, the `boolean` function can be wrapped around it to return false/true instead.

I am the first to admit that using `some` is non-obvious because it leverages several Clojure features in a surprising way for a newcomer (collections as functions, literal sets, and truthy/falsey values). 

But! Note that `some` breaks if you happen to want to know whether a collection contains a falsey value (`false` or `nil`). In those cases, you will get back a falsey value regardless.

If you're interested in a sequential search with early exit that works on falsey values, you'll want something like this:

{% highlight clojure %}
(defn contains-val? 
  [coll val]
  (when (seq coll)
    (or (= val (first coll))
        (recur (next coll) val))))
{% endhighlight %}

We could also envision this in terms of reduce (since we can more efficiently apply reduce in some cases):

{% highlight clojure %}
(defn contains-val?
  [coll val]
  (reduce #(if (= val %2) (reduced true) %1) false coll))
{% endhighlight %}

I ran a quick benchmark on finding the last value in (vec (range 1000)) and found that the first version took 27 µs and the second reduce-based one took 16 µs. For comparison, a set lookup took 81 ns. So use the right data structure!

## Going Java

Another tricky approach is to leverage that Clojure collections implement java.util.Collection, which includes a .contains() that really does check for value containment (and works with nil):

{% highlight clojure %}
=> (.contains [3 4 5] 5)
{% endhighlight %}

## Last word

[Rich on contains?](https://groups.google.com/forum/#!msg/clojure/qNLBQkSB6jk/XUbQnBRcWPIJ)
