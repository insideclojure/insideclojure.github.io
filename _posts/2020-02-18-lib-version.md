---
layout: post
date: 2020-02-18
title: 1.0 all the things
---

I've been working on a write-up for the 2020 State of Clojure survey (sorry that's taking so long). One feedback comment stuck with me:

"A lot of libraries in the Clojure ecosystem (including in clojure.core) have version 0.y.z, indicating breaking changes could still be introduced. Yet some of these libraries have widespread adoption. The Clojure community emphasises the importance of backwards compatibility, but including some-lib 0.y.z in a large production system still makes me uncomfortable. What is preventing these libs, like data.csv, from becoming stable?"

The short answer to this is a) nothing and b) these libraries have already been stable for years.

The Clojure ecosystem in general is not big on the concept of semantic versioning (see Rich's [talk](https://github.com/matthiasn/talk-transcripts/blob/master/Hickey_Rich/Spec_ulation.md) about why semver is broken), and I personally do not see 0.x vs 1.x as having any special meaning. But clearly that is not the case for the commenter above, or for other comments I've seen of the same vibe. I suspect this is probably most strongly the case for those coming from the JavaScript/NPM ecosystem, or other language communities where semver is held in higher regard.

Semver tries to separate "major" changes (anything could have changed, and you may be broken by updating) from "minor" changes (additive or fixative changes and you will not be broken by updating). Rich's point is that breaking APIs is not the way to build a stable software ecosystem, and we should stop doing it. Also, you can turn a breaking change into a growing change by giving it a new name, rather than changing the meaning of the old name. If you puruse growth over breakage, then there is never a version that "breaks" users. tools.deps has its own version selection algorithm that aligns with this (always preferring newer over older versions).

Clojure and the contrib libraries have endeavored to follow this model of change for a long time. Consequently, we have lots of libraries in the Clojure ecosystem that have been around for many years, are widely used, have stable APIs, and yet are 0.x version. It's silly for that to be (falsely) indicating to people not to use them, so I have asked Clojure contrib library owners to more actively bump up their library versions. 

If you are a Clojure library maintainer and in this same position, I'd encourage you to bump as well!





There was a question on Slack this evening about getting a similar effect to the injections feature of Leiningen in clj. There is no built-in feature for this, but you don't really need one. The provided [clojure.main/repl](https://clojure.github.io/clojure/clojure.main-api.html#clojure.main/repl) function is highly customizable. For example, it provides an `:eval` hook, which defaults to just the `eval` function.

But that's a great place to inject your own code around every eval, you're almost there. At an existing repl, you can try a program like this to start a new nested repl with that function:

{% highlight clojure %}
(require '[clojure.main :as m])
(m/repl
  :init #(apply require m/repl-requires) 
  :eval (fn [form]
          (do
            (require '[clojure.pprint :refer [pp]])
            (eval form))))
{% endhighlight %}

The :init hook replicates what clojure.main uses as an initial binding set, so it's good to repeat that. The :eval hook just does an additional referral, making `pp` always available.

You can start this as your actual repl at a command line by just smooshing it all together:

{% highlight bash %}
clj -e "(require '[clojure.main :as m]) \
        (m/repl :init #(apply require m/repl-requires) \
        :eval (fn [form] (do (require '[clojure.pprint :refer [pp]]) \
        (eval form))))"
{% endhighlight %}

Or you can pack it into your deps.edn (using the Corfield comma as bash-safe whitespace):

{% highlight clojure %}
{:aliases {:repl {:main-opts ["-e" "(require,'[clojure.main,:as,m])(m/repl,:init,#(apply,require,m/repl-requires),:eval,(fn,[form],(do,(require,'[clojure.pprint,:refer,[pp]])(eval,form))))"]}}}
{% endhighlight %}

then just `clj -A:repl`

It would be pretty easy to wrap this tiny program into a tool with the injections as an argument for reuse but not sure it's worth that.
