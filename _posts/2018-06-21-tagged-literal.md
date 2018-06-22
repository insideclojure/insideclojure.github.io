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



The call to `add-lib` takes a lib and a coordinate, exactly what you'd find in a deps.edn file and supports all of the current coordinate types (Maven, git, or local). `add-lib` uses the resolver in tools.deps.alpha to determine the transitive set of dependencies, download them to your Maven cache (if needed) and add the urls of those libs to the highest level DynamicClassLoader in your current thread. `add-lib` returns true if it added any urls. You can then use namespaces from the dependencies without restarting your REPL.

This technique for expanding the libraries available to the current context will not work in all REPL situations. While some of those constraints are known, I'd be interested to learn more about where this does or doesn't work. It may be possible that changes in Clojure core could make it more readily accessible.

Additionally, this code is not just blindly adding dependencies you ask for but is actually considering them in the context of the dependencies already on your classpath. So if for example, you already had a version of org.clojure/data.priority-map above, it would do the dependency resolution in the context of that version and not resolve to or add a new version to the classpath.

The `tools.deps.alpha.libmap` namespace contains code to load the starting lib map and maintain the runtime version as libs and their dependencies are added. The lib map can be loaded by several means:

* System property `clojure.libfile` - referring to an edn file holding the lib map. `clj` will set this automatically for you and point to the cached lib map it's using.
* System property `clojure.libmap` - referring to an edn string holding the lib map data, useful for tools that construct the classpath via means other than `clj`.
* System property `java.class.path` - by analyzing the Java classpath and using standard location information about the Maven local repository and structure, the libmap can be inferred.

How all of this works and which options are available is likely to change. Eventually some of this may even move into Clojure itself in the future.

A great place to chat about this is in `#tools-deps` in [Clojurians slack](http://clojurians.net/).
