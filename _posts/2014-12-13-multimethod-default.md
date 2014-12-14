---
layout: post
title: Caching multimethod default value dispatch
---

Ticket [CLJ-1429](http://dev.clojure.org/jira/browse/CLJ-1429) started from a 
[tweet](https://twitter.com/PieCalculus/status/468621724205203456) by Anton 
Logvinenko ([@piecalculus](http://twitter.com/piecalculus)) indicating that a
heavily used multimethod was showing a cold cache. 

I have seen this problem
in my own code in the past (before I was working on Clojure itself) so I 
immediately saw the chance to find and kill a problem that has affected me.
Anton did a phenomenal job feeding me everything I needed to quickly reproduce
and detect the problem.

Backing up slightly, multimethods are functions that can dispatch to many possible
method implementations based on the result of an initial dispatch function. For 
example:

```clojure
;; Define the multimethod mm that dispatches on the function class
(defmulti mm class)

;; Define some method implementations based on different classes
(defmethod mm String [s] s)
(defmethod mm Long [l] l)

;; You can also define a fallthrough default for when nothing matches
(defmethod :default [v] v)
```

Invoking a multimethod involves evaluating two functions: the dispatch function (class in
the example above) and then using the return value to select the matching method 
implementation to actually invoke. The backing [MultiFn](https://github.com/clojure/clojure/blob/83ebf814d5d6663c49c1b2d0d076b57638bff673/src/jvm/clojure/lang/MultiFn.java) class maintains a methodTable
map from dispatch value to IFn (the method). However, finding a match is not exactly
a simple lookup. There are several other features of multimethods:

1. Inheritance hierarchy matching - using either custom hierarchies or Java class hierarchies
2. Preferences - in the case of multiple possible matches, a preference may be stated for resolution
3. Default dispatch - a method to use if no match is found (marked with :default by default)

Due to the possibilities of multiple matches resolved by preferences and falling through to
a default, there is some complicated logic that implements this decision. For performance, 
the decision reached is cached at the end. Because multimethods are open (new methods
or preferences may be added at any time), this logic must deal with the concurrency
issues and possibility of a change happening during invocation.

I've annotated the key method here outlining what happens (note this is prior to the fix):

{% gist 06008478ae7beeaced82 MultiFn-before.java %}

In the gist above you can see where I've noted the bug - if no matching method is found we
do not alter the cache. That means that in the fallthrough case, the matching logic is done
every time, which requires walking through every entry in the method table. 

One situation where I see this come up as a frequent performance issue is when using
clojure.walk with an update function, where there is often a default fall-through case. 

Now here is the same method after the patch (now included in Clojure 1.7):

{% gist 06008478ae7beeaced82 MultiFn-after.java %}

We now look up and alter the cache when the default branch is taken. This can make a dramatic 
difference in performance when you are using the :default case. Here is a simple test 
using a mixture of default and non-default values:

```clojure
;; build a cycle of strings, longs, and keywords of requested size
;; the multimethod handles strings and longs but not keywords
;; time each rep where the multimethod is mapped over the cycle
(defn perf [reps size]
  (let [data (take size (cycle ["abc" 5 :k]))]
    (dotimes [_ reps]
      (time (doall (map mm data))))))

;; Before patch:
user=> (perf 5 100000)
"Elapsed time: 1301.262 msecs"
"Elapsed time: 928.888 msecs"
"Elapsed time: 942.905 msecs"
"Elapsed time: 858.513 msecs"
"Elapsed time: 832.314 msecs"

;; After patch:
user=> (perf 5 100000)
"Elapsed time: 134.169 msecs"
"Elapsed time: 28.859 msecs"
"Elapsed time: 45.452 msecs"
"Elapsed time: 13.189 msecs"
"Elapsed time: 13.42 msecs"
```

The JIT warms up in both cases but you can see that there is a dramatic performance boost here.
This change was added in Clojure 1.7.0-alpha1.


