---
layout: post
date: 2020-09-30
title: An example of clj execution
---

Last week we [released](https://clojure.org/releases/tools#v1.10.1.697) a new version of the Clojure tools with support for directly [executing](https://clojure.org/reference/deps_and_cli#_executing_a_function) a Clojure function that takes a map.

There is one good example of this sort of program in Clojure itself - `clojure.core.server/start-server`, which can be used to [start a socket server](https://clojure.github.io/clojure/clojure.core-api.html#clojure.core.server/start-server). Clojure users are probably most familiar with using the socket server to run a remote repl (but it can really be used to run any kind of socket server).

Looking closer at this function's doc:

```
(doc clojure.core.server/start-server)
-------------------------
clojure.core.server/start-server
([opts])
  Start a socket server given the specified opts:
    :address Host or address, string, defaults to loopback address
    :port Port, integer, required
    :name Name, required
    :accept Namespaced symbol of the accept function to invoke, required
    :args Vector of args to pass to accept function
    :bind-err Bind *err* to socket out stream?, defaults to true
    :server-daemon Is server thread a daemon?, defaults to true
    :client-daemon Are client threads daemons?, defaults to true
   Returns server socket.
```

The Clojure tools now support the invocation of an ad hoc function using the -X parameter, and then a series of key value parameters. To invoke this function, it's necessary to pass all of the required arguments (`:name`, `:port`, `:accept` function) plus we need to set `:server-daemon` to false to prevent the JVM from exiting:

```
clojure -X clojure.core.server/start-server :name '"server"' :port 5555 :accept clojure.core.server/repl :server-daemon false
```

Once the repl server has been started we can verify it's working by connecting with netcat in a separate terminal:

```
nc localhost 5555
user=> (+ 1 1)
2
```

Of course, that's a bit wordy on the command line and we mostly want all of those options every time so instead we can move those arguments into the deps.edn file:

{% highlight clojure %}
{:aliases
 {:repl-server
  {:exec-fn clojure.core.server/start-server
   :exec-args {:name "server"
               :port 5555
               :accept clojure.core.server/repl
               :server-daemon false}}}}
{% endhighlight %}

And then we can just invoke the function via the args in the alias:

```
clojure -X:repl-server
```

The execution mode also allows command line overrides of those parameters stored in the exec-args:

```
clojure -X:repl-server :port 5678
```

That's it! This just seemed like a good quick example to understand how -X works...



