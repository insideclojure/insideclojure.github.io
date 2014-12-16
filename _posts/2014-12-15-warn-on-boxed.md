---
layout: post
date: 2014-12-15
title: Boxed math warnings
---

In Java, there are two forms of numbers - primitive and boxed (Object) forms. 
This distinction may be resolved some day with [value types](http://cr.openjdk.java.net/~jrose/values/values-0.html), but for now that's just the reality of the JVM and something every JVM-based language deals with.
In Clojure, numbers are boxed by default and always passed as Objects in a var invocation.
Clojure supports type hints and other options for using unboxed primitives instead.

A very common performance issue in numeric Clojure code is the unintentional use of boxed numbers.
For example:

{% highlight clojure %}
(defn sum-squares [a b]
  (+ (* a a) (* b b)))
{% endhighlight %}

You can examine the bytecode for this using either the [no.disassemble](https://github.com/gtrak/no.disassemble) plugin or by AOT-compiling and using javap to disassemble.

Using no.disassemble, you can do something like this at the REPL: 

{% highlight clojure %}
(require '[no.disassemble :refer (disassemble)])
(println (disassemble (fn [a b] (+ (* a a) (* b b)))))
{% endhighlight %}

Below is the bytecode for just the invoke method of this function:

~~~
  public java.lang.Object invoke(java.lang.Object a, java.lang.Object b);
     0  aload_1 [a]
     1  aload_1 [a]
     2  aconst_null
     3  astore_1 [a]
     4  invokestatic clojure.lang.Numbers.multiply(java.lang.Object, java.lang.Object) : java.lang.Number [39]
     7  aload_2 [b]
     8  aload_2 [b]
     9  aconst_null
    10  astore_2 [b]
    11  invokestatic clojure.lang.Numbers.multiply(java.lang.Object, java.lang.Object) : java.lang.Number [39]
    14  invokestatic clojure.lang.Numbers.add(java.lang.Object, java.lang.Object) : java.lang.Number [42]
    17  areturn
~~~

Analysis of this bytecode:

- Method signature - inputs and output are all Objects (so boxing happens on input and output)
- line 0 - loads a from local variable 1 onto the stack ("this" is in 0, a is in 1, b is in 2)
- line 1 - loads a from local variable 1 onto the stack
- lines 2-3 - load a constant null and store it back to local variable 1. This is the pattern that "clears" the local variable, freeing it for GC. In this case, it doesn't matter, but it can if local 1 was the head of a sequence.
- line 4 - invokes the static method Numbers.multiply(Object, Object) : Number. Boxing! The a's on the stack are removed and replaced with the result.
- lines 7-11 - exactly the same thing as 0-4, but with b instead
- line 14 - picks up the a^2 and b^2 results and adds them via Numbers.add(Object, Object) : Number (Boxing!)
- line 17 - returns the object from the stack

Let's sprinkle some type hints on the function and see what changes:

{% highlight clojure %}
(defn sum-squares ^long [^long a ^long b]
  (+ (* a a) (* b b)))
{% endhighlight %}

~~~
  public final long invokePrim(long a, long arg1);
     0  lload_1 [a]
     1  lload_1 [a]
     2  invokestatic clojure.lang.Numbers.multiply(long, long) : long [40]
     5  lload_3 [arg1]
     6  lload_3 [arg1]
     7  invokestatic clojure.lang.Numbers.multiply(long, long) : long [40]
    10  invokestatic clojure.lang.Numbers.add(long, long) : long [43]
    13  lreturn
~~~

Now we have an invokePrim function (the prior invoke Object version still exists, just in case this gets called with something else!) that takes and returns longs. Note that we now do not need to clear locals, we don't call methods in Numbers with Object and we have a long return at the end. This is good, but we can actually do more.

The Numbers methods will do bounds checking and throw on overflow - this is the default Clojure behavior. We can get rid of bounds checking if we don't care and want better performance by setting the dynamic variable \*unchecked-math*. Let's see what that looks like...

{% highlight clojure %}
(set! *unchecked-math* true)
(defn sum-squares ^long [^long a ^long b]
  (+ (* a a) (* b b)))
{% endhighlight %}

~~~
  public final long invokePrim(long a, long arg1);
    0  lload_1 [a]
    1  lload_1 [a]
    2  lmul
    3  lload_3 [arg1]
    4  lload_3 [arg1]
    5  lmul
    6  ladd
    7  lreturn
~~~

The calls to Numbers.multiply() and .add() have been replaced with bytecodes that work directly on primitives. These are easier to JIT into hardware instructions as well.

Let's see the performance difference for running the function over 100 numbers (using [criterium](https://github.com/hugoduncan/criterium) for benchmarking which takes care of many common JVM measuring pitfalls):

{% highlight clojure %}
(require '[criterium.core :refer :all])
(def v (vec (range 100)))
(quick-bench (mapv #(sum-squares % %) v))
{% endhighlight %}

Version | Time 
----- | ----
Boxed | 7.35 µs
Checked primitives | 4.31 µs
Unchecked primitives | 3.35 µs

Having done this kind of thing a lot, it's kind of annoying to be disassembling the code all the time. Something that I worked on ([CLJ-1325](http://dev.clojure.org/jira/browse/CLJ-1325)) and which is now available is a new setting that will warn on many uses of boxed methods in the Numbers class (anything taking or returning an Object or Number). Here's how it works:

{% highlight clojure %}
user=> (set! *unchecked-math* :warn-on-boxed)
:warn-on-boxed
user=> (defn sum-squares [a b]
  #_=>   (+ (* a a) (* b b)))
Boxed math warning, /private/var/folders/7r/_1fj0f517rgcxwx79mn79mfc0000gn/T/form-init278329133936121132.clj:2:6 - call: public static java.lang.Number clojure.lang.Numbers.unchecked_multiply(java.lang.Object,java.lang.Object).
Boxed math warning, /private/var/folders/7r/_1fj0f517rgcxwx79mn79mfc0000gn/T/form-init278329133936121132.clj:2:14 - call: public static java.lang.Number clojure.lang.Numbers.unchecked_multiply(java.lang.Object,java.lang.Object).
Boxed math warning, /private/var/folders/7r/_1fj0f517rgcxwx79mn79mfc0000gn/T/form-init278329133936121132.clj:2:3 - call: public static java.lang.Number clojure.lang.Numbers.unchecked_add(java.lang.Object,java.lang.Object).
{% endhighlight %}

Since I'm doing this at the REPL, the line/col info is a bit hard to read---it's better with actual files. The warnings are pointing out all the places inside the function where boxed math is occurring (the two multiplys and the add). This feature was added in 1.7.0-alpha2 and tweaked a bit to the form above in alpha3. 

You probably noticed we didn't see the boxing occurring on the way in/out of the function. That's not actually boxed math, just function invocation, so it's a little harder to detect and warn on automatically. We do have another ticket that would catch boxing on the return paths ([CLJ-1585](http://dev.clojure.org/jira/browse/CLJ-1585)) but it also catches a lot of false positives so needs more work so it won't be in 1.7.

