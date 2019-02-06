---
layout: post
date: 2019-02-07
title: Survey comments
---

Daniel Compton posted a [breakdown and summary of comments](https://danielcompton.net/2019/02/06/clojure-survey-2019) from the [Clojure survey](https://clojure.org/news/2019/02/04/state-of-clojure-2019) and I wanted to highlight a few things in response to the comments there.

Re *spec*, we are working on it full time right now and I expect that will be the case for weeks to come. Certainly this is our focus right now. You can follow my weekly progress on my journals here.

Re *docs*, there are tons of docs out there, and we added a lot of great resources on the main site this year. I would love to have "from the beginning" guides for how to get started on the major editors/platforms that could be hosted on clojure.org. And I think also tutorials of "how to make a simple web app" style are also hugely useful and would love to see someone contribute things like that. If you're looking to help, those would be fantastic contributions to the Clojure/ClojureScript sites (PRs welcome).

Re *startup time*, haven't done any visible work on this for a while, but Rich and I brainstorm about this every few months and we do have some ideas to work on when we get past spec. When people say "startup time", this is really a more nuanced problem than most people realize and raw startup time (for writing scripts or something) is not really a goal of Clojure (self-hosted CLJS or Graal provide some good alternatives). However, improving general loading time for namespaces, particularly source not AOT, is the primary reason that startup of big apps is slow and that's an important problem with a lot of upside across many use cases. I very much would like to do some work on this in the next Clojure.

Re *hiring*, another way to solve issues with hiring is to teach Clojure to more people, rather than only hiring people that already know Clojure (of course you should do that too!). [Amperity](https://www.youtube.com/watch?v=QBsjYyg9bLE) seems to have a great model program for onboarding and teaching.

Re *dev process*, we've talked about this a bit and when we get back to Clojure core work (from spec), we will be looking at some new ways to connect with the community and focus our efforts appropriately.

Re *"native" support*, I think in general people underestimate how much value we get from the JVM / JS engines and how much work would be necessary to do "native" Clojure. In any case, this is not something the core team plans to work on. If you want to work on it, please do! We have been trying to at least not get in the way of Graal and fixed a couple things in 1.10 that were obstacles for it, happy to keep doing that.

Re *clj on Windows*, we need your help! A lot of work has been done, but I do not have the bandwidth to take it over the line. See https://dev.clojure.org/jira/browse/TDEPS-67 and the #clj-on-windows room in Slack. If you want it, help us get there.

And finally, I will just say thanks to Daniel Compton for everything he does for the community - between the REPL, the podcast, Clojurists Together, clojars, deps, etc, he puts in a ton of time and care for us and I really appreciate everything he does. Please give him your thanks! 
