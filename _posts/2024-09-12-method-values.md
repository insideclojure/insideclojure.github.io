---
layout: post
date: 2024-02-12
title: Method Values
---

Clojure [1.12.0-alpha6](https://clojure.org/news/2024/02/08/1-12-alpha6) introduced method values to Clojure and this post will explore them in a bit more depth.

## The olden days

Clojure functions are first-class values and can easily be passed around, but a common question seen in Clojure forums for years is how to map a static method (or instance method or constructor) over a collection. The idiomatic answer has been to wrap it in an anonymous function:

{% highlight clojure %}
;; static method
(map #(Long/toBinaryString %) (range 8))
;; ("0" "1" "10" "11" "100" "101" "110" "111")

;; instance method
(map #(.toUpperCase %) ["hi" "there"])
;; ("HI" "THERE")

;; constructor
(map #(java.util.Date. %) [1707771694522 1707771780922])
;; (#inst "2024-02-12T21:01:34.522-00:00" #inst "2024-02-12T21:03:00.922-00:00")
{% endhighlight %}

These all work, but it seems like we could do better, especially as Java increasingly allows you to treat methods and constructors as functions and pass them around as first-class values.

## Java methods are Clojure functions

Since alpha6, qualified method symbols are Clojure function values and can be used as if you had created the anonymous functions above:

{% highlight clojure %}
(map Long/toBinaryString (range 8))
(map ^[] String/toUpperCase ["hi" "there"])
(map ^[long] java.util.Date/new [1707771694522 1707771780922])
{% endhighlight %}

Qualified method symbols can now refer to static methods, instance methods (the first arg is the instance), and constructors (with `/new`). The Clojure compiler must find the single method the method refers to to create the function value that invokes it. If the method is not overloaded and has only one arity, this is unambiguous (as with the first example).

If there are multiple arities for the method name, it would be possible to make a function that supported all of the arities, but it is unusual to need this and it is not supported. Usually there is a single arity for which the method will be applied (often arity 1). In the second example here, the instance method has arities that take 0 and 1 arguments (remember that the instance itself is always an argument to instance methods). To differentiate, we use :param-tags, a new metadata tag that annotates a method symbol with the parameter types of the desired arity and overload. Because the desired arity takes no arguments, this is specified as an empty vector: `^[]`. If there is no overloading, use wildcard `_` for parameters as a wildcard - the number of wildcards must match the desired arity.

If there are multiple overloads for method name, specify the desired overload types in the param-tags instead of wildcards (as in the 3rd example above).

The emitted code is the same code you might have written at the beginning in the anonymous function. However, the generated function also knows exactly the method being invoked, and its parameter and return types. Due to that, there is no reflection (the original examples had reflection) and primitive type hints or coercions where they could be usefully applied (these were all missing in the original examples).

## Qualified methods in invocation position

For uniformity, Clojure also now supports qualified methods and constructors in the invocation position (in addition to the `.instanceMethod` and `AClass.` syntax commonly used):

{% highlight clojure %}
(Long/toBinaryString 10)
(^[] String/toUpperCase "hi")
(^[long] java.util.Date/new 1707771694522)
{% endhighlight %}

In general, qualified method symbols always require the selection of exactly one method/arity/overload. :param-tags metadata must be supplied any time there are multiple arities or overloads - no inference is done. This may be more verbose than the older syntax, but the tradeoff is specificity. You know there is no inference and no reflection.

One exception to this is the only pre-existing case where qualified methods were used prior - static methods in invocation position. Static methods in invocation position without :param-tags previously supported inference, and continues to do so for backwards compatibility.

## Error handling

Error messages were considered in a variety of failure cases and provide useful feedback when qualified methods and param-tags are insufficient.

If a method does not exist at all:

{% highlight clojure %}
user=> (^[_] String/fooBar)
Syntax error (IllegalArgumentException) compiling at (REPL:1:1).
Could not find method fooBar in class java.lang.String
{% endhighlight %}

If no param-tags is supplied and the qualified method was ambiguous by arity or overload, recommend using param-tags:

{% highlight clojure %}
user=> (String/new "a")
Syntax error (IllegalArgumentException) compiling at (REPL:1:1).
Multiple matches for constructor in class java.lang.String, use param-tags to specify
{% endhighlight %}

If :param-tags metadata could not select an overloaded method (either found none or more than one):

{% highlight clojure %}
user=> (^[_] Math/abs -1)
Syntax error (IllegalArgumentException) compiling at (REPL:1:1).
Expected to find 1 matching signature for method abs in class java.lang.Math but found 4 with param-tags [_]
{% endhighlight %}

A new kind of error mode when not using inference is that the :param-tags metadata might specify a method with a different arity than the number of arguments supplied (here in invocation position, but more likely in value position):

{% highlight clojure %}
user=> (^[long] Math/abs -1 -2)
Syntax error (IllegalArgumentException) compiling Math/abs at (REPL:1:1).
Invocation of method abs in class java.lang.Math expected 1 arguments, but received 2
{% endhighlight %}

## Static fields

Static fields are another kind of qualified symbol, `Aclass/staticField` that already existed in Clojure. Static fields evaluate to their value and do not need to be invoked with parens. Due to details of the implementation, surrounding static fields with parens does appear to have the same effect, but this has not ever been published as valid syntax and it breaks the general substitutability of Clojure expressions - a static field in parens should invoke the value of the field.

For backwards compatibility, this unintended behavior has been retained in Clojure 1.12, but clj-kondo will now warn about the use of `(AClass/staticField)` and we hope that eventually we can remove this provision in Clojure.

## Unsupported

Java allows you to pass methods from a particular instance object ("bound functions"), effectively closing over the "this" object. Clojure qualified method values do not support this - use an anonymous function that closes over the object, as before.

Java vararg methods are implemented in the JVM as methods that receive a typed array of values for the trailing argument, and Clojure method values match this behavior. It is not possible to supply individual values to var arg methods in either invocation or value position, although this may be considered in a future release.


