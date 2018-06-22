---
layout: post
date: 2018-06-21
title: Default data reader function and tagged literals
---

Clojure has always avoided including reader macros with the goal of making all code readable by all users of Clojure, without requiring optional code. It's a small constraint, but on balance I think has been a good one.

However, Clojure 1.4 took one small step in allowing users to define reader literals, custom data structures built as compositions of other readable data, marked by a tag. For example, there are a couple of built-in reader literals for #uuid and #instant.

Clojure 1.5 made some additional enhancements to add more support for a default data reader function in the dynamic var `*default-data-reader-fn*`. This is a function that will be invoked if no data reader function is installed for a tag, sort of a "tag missing" function. By default, it's not set and most people don't use it.

However, it's pretty useful to have a system that can generically handle (and pass on) any reader literal it happens to encounter. While you may not know what to do with it, perhaps the next process down the line does.

Clojure 1.7 added support for a generic TaggedLiteral data type, embodied in the functions `tagged-literal` and `tagged-literal?` (it's really just a simple deftype). As it turns out, the signature of `tagged-literal` is exactly the same as the function expected for the `*default-data-reader-fn*` (it takes a tag and form and returns an object).

To get this ability to read, print, and pass along any tagged literal (even if not registered), you just need to connect these two pieces:

`(set! *default-data-reader-fn* tagged-literal)`

The object produced in this case supports the ILookup interface for keyword lookup with the keys :tag and :form, so you can interrogate the unknown tag as well. TaggedLiteral has built-in print support.

Here's a bit longer example:

{% highlight clojure %}
;; Clojure can't read tagged literals without a registered reader:
user=> #object[clojure.lang.Namespace 0x23bff419 "user"]
RuntimeException No reader function for tag object  
clojure.lang.LispReader$CtorReader.readTagged (LispReader.java:1430)

;; Set tagged-literal to be the default tagged value reader:
user=> (set! *default-data-reader-fn* tagged-literal)

;; Try again
user=> #object[clojure.lang.Namespace 0x23bff419 "user"]
#object [clojure.lang.Namespace 599782425 "user"]

;; Now it works, and reads to a TaggedLiteral object, which
;; supports ILookup on :tag and :form keys
user=> [(:tag *1) (:form *1)]
[object [clojure.lang.Namespace 599782425 "user"]]
{% endhighlight %}

You might ask, wouldn't this be a useful default default data reader function? Indeed it might....
