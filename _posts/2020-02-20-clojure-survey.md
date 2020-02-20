---
layout: post
date: 2020-02-20
title: Clojure survey feedback
---

I just published the results from the [State of Clojure 2020 Survey](https://clojure.org/news/2020/02/20/state-of-clojure-2020), but I also wanted to post some thoughts on the open feedback section at the end. I read through all the responses there (about 645 responses from the 2500+ people that completed the survey). As usual, a large percentage (about 65%) were expressions of either thanks or general happiness, so thanks for those, they are fun to read and feel appreciated.

I'll also shout out some people in the community that came up many times - Sean Corfield for his tireless library work and help for beginners, Daniel Compton for his work with Clojurists Together (and his many other efforts), and Michiel Borkent for his work on scripting and tooling around graal, clj-kondo, sci, etc. Kudos to them, and to anyone else I missed in the responses!

I also tried to categorize and sort through the top requests that I saw and here's the rough tally I came up with (subject to a lot of interpretation of course) for Clojure:

* 20+ mentions
** Spec 2
** More or better beginner or getting started docs
** More jobs or more people to hire for jobs
* 10+ 
** Best practices docs on specific topics - spec, core.async, code organization, etc
** Error messages
** Graal native image support
* 3<x<10
** Startup performance (specifically more for apps, related to namespace loading, not scripting cases)
** Reference docs or docstrings/examples
** Runtime performance
** Data science
** More involvement by community in road map
** Static analysis tools
** Java interop, specifically around Java 8+ functional APIs
** Web framework
** General lib stagnation/abandonment
* Fewer - stack traces, debuggeer, various tools.deps improvements, etc

So that's a lot of stuff, but the good thing is, there's a high degree of overlap between this list and what we consider priorities for the core team and direction. Spec 2 has not had a lot of active commits, but has been in active design off and on and I want nothing more than to get back to it and finish it, which should happen this year.

For the next Clojure version (presumably 1.11), we plan to fix the Graal locking issue and have also talked about several other possible features including Java interop for functional APIs, startup performance, etc. There are a few small improvments for error messages queued up, but I think we're likely to see the best changes come out of spec and new directions towards function specs.

In the community, we also see active work around data science/machine learning/AI with things like libpython-clj, Neanderthal, and the Clojure data science online meetup. I don't think the core team actively needs to drive any of that and we are supportive and in touch with people as needed. Michiel Borkent's clj-kondo is a great static analysis tool that has also pushed the state of Graal with Clojure further, and again this is mostly community-led (with the exception of the issue mentioned above).

For docs, I definitely feel the need there, particularly for beginners, and there are certainly things to do, just a matter of balancing it with all the things above. I definitely appreciate everyone writing or building things in the community to help new Clojurists. I've also written a big chunk of core.async docs, but they have enough holes that I haven't gotten them out there yet.

Hiring/getting hired in a Clojure job is a perennial issue. Jobs exist, and people exist, but the distribution of those in geography or experience are often out of whack. Probably the best place to make those connections are at in-person conferences, and I'm happy that we've seen so many new conferences spring up over the last couple years in multiple continents.

If you want to dive more deeply into this feedback and see what you can do to help, the response can be downloadeed as part of the [full results](https://www.surveymonkey.com/results/SM-CDBF7CYT7/)..
