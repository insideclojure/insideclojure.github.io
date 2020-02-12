---
layout: post
date: 2020-02-02
title: Customizing your REPL in clj
---

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

{% highlight %}
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
