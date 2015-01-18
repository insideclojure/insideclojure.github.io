---
layout: post
date: 2015-01-18
title: Reducible sequence generators
---

I've [written in the past](/2014/12/21/reducible-repeat) about making functions in core that generate sequences (specifically `cycle`, `iterate`, and `repeat`) directly reducible in [CLJ-1603](http://dev.clojure.org/jira/browse/CLJ-1603). The approach described there was to implement a new class (in Java) that directly implemented both the ISeq and IReduceInit interfaces, making something that can act as both.

When Stu Halloway screened the ticket he expressed some concerns about this approach as it introduced a lot of new Java code. Admittedly, this gave me pause as well. Rich suggested that maybe I was working too hard on creating something that was both a seq and reducible and instead creating something that was *seqable* and reducible. 

Based on that suggestion, I created a new version of the patch that used deftype to create something that was Seqable (and returned the same seq implementation used in the past) and IReduceInit for these functions. This worked great and introduces a small amount of Clojure rather than a large amount of Java, also pleasing. 

Here's an example for cycle with a few annotations:

{% highlight clojure %}
;; This is the original definition of cycle, renamed as cycle-seq
(defn- cycle-seq
  [coll] (lazy-seq
           (when-let [s (seq coll)]
             (concat s (cycle-seq s)))))

;; New deftype implementing Seqable and IReduceInit
(deftype Cycle [coll]
  clojure.lang.Seqable
  (seq [_] (cycle-seq coll))  ;; just call the old impl

  clojure.lang.IReduceInit
  (reduce [_ rf init]
    (loop [ret init
           s coll]
      (let [rr (rf ret (first s))]
        (if (reduced? rr)
          @rr
          (let [s (next s)]
            (recur rr (if (nil? s) coll s)))))))

  ;; declaring as Sequential will cause the seq to be used for nth, etc
  clojure.lang.Sequential)

;; New cycle implementation typically constructs a Cycle,
;; except for the special case of an empty collection to cycle
(defn cycle
  "Returns a lazy (infinite!) sequence of repetitions of the items in coll."
  {:added "1.0"
   :static true}
  [coll] (if-let [c (seq coll)]
           (->Cycle c)
           ()))
{% endhighlight %}

One thing that needed to be handled was that deftype is not defined until relatively late in clojure.core. The definition of cycle, iterate, and repeat had to be moved much later, till after deftype is defined. 

For cycle and iterate this was not a big deal, but repeat is used in a number of places making that unworkable by itself. Instead, I used the same technique used for reduce by supplying an internal version of repeat (repeat1) that is used until deftype is defined and the real version of repeat can be created.

One tradeoff with the deftype approach is that while reduce has comparable performance gains, seq does not gain any of the benefits of the all-Java approach, so performance is about the same as in prior releases when walking as a sequence.

*Note: This patch has been screened but not yet included in a release and may still undergo changes.*