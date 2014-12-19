---
layout: post
date: 2014-12-19
title: Improving compile times
---

Today's post is about a compiler change in 1.7.0-alpha4 that reduces compile times on some (not all) projects, as tracked in [CLJ-1529](http://dev.clojure.org/jira/browse/CLJ-1529). This ticket was filed by the perspicacious [Zach Tellman](http://ideolalia.com/). Zach was specifically seeing issues with the compile times for his Aleph project and did a bit of profiling to find a hot spot.

In particular this method in the compiler code for host expressions (version before patch seen here).
The purpose of this code is to determine whether a given form used in a host expression refers to a Class.
The stringOk flag indicates whether a String representing a class name is valid too.

{% highlight java %}
private static Class maybeClass(Object form, boolean stringOk) {
  if(form instanceof Class)
    return (Class) form;
  Class c = null;
  if(form instanceof Symbol) {
    Symbol sym = (Symbol) form;
    if(sym.ns == null) //if ns-qualified can't be classname
    {
      if(Util.equals(sym,COMPILE_STUB_SYM.get()))
        return (Class) COMPILE_STUB_CLASS.get();
      if(sym.name.indexOf('.') > 0 || sym.name.charAt(0) == '[')
        c = RT.classForName(sym.name);
      else {
        Object o = currentNS().getMapping(sym);
        if(o instanceof Class)
          c = (Class) o;
        else {
          try{
            c = RT.classForName(sym.name);
          } catch(Exception e){
            // aargh
            // leave c set to null -> return null
          }
        }
      }
    }
  }
  else if(stringOk && form instanceof String)
    c = RT.classForName((String) form);
  return c;
}
{% endhighlight %}

This method is used in several circumstances. A particularly problematic path occurs during macro-expansion, in trying to understand a host interop call like (.append sb (char c)). 
In this case, the question to be resolved is whether "sb" is a class or a reference to an instance.

The code above will fall all the way through to the RT.classForName() call. 
Asking a classloader to speculatively load the class just to determine whether it really is a class is expensive.

As it turns out, Rich Hickey had found this same hot spot earlier this year when working on investigating lazy var loading in his work on the fastload branch.
Both Zach and Rich solved the problem in different ways.
Zach proposed a modification to maybeClass to check whether the symbol in question was a symbol that had meaning in the local scope at the point of the check:

{% highlight java %}
  if(LOCAL_ENV.deref() != null && ((java.util.Map)LOCAL_ENV.deref()).containsKey(form)) {
    return null;	
  }
{% endhighlight %}

Zach inserted this right after casting form to sym. Unfortunately this had the side effect of altering the semantics of some (perhaps pathological) code like this:

{% highlight clojure %}
 (let [String "foo"] 
      (. String substring 0 1)) 
{% endhighlight %}

Before the patch, the String in the interop call would be treated as a class and after the patch it would be treated as a method invocation on the lexical instance. This change in semantics gave us some heartburn because while this situation is probably rare, we were unsure whether it would break existing code in the wild.

I decided to extract Rich's approach from the fastload branch and take a look at that as an alternative. 
Rich's approach was to create a cache around this maybeClass question and remember the answer.

Once I had two patches in hand, it was time to collect some data.
I went out and tested the time to build and run the tests in a varied set of open source Clojure projects.
I found that both patches helped but Zach's patch was definitely making a bigger impact, particularly on macro-heavy projects like aleph, riemann, and lamina. 

I went back to the patches and took another look at both of them.
I was able to find a few additional efficiences in the cache case, and also get a better handle on the set of cases where semantics were changing from Zach's patch.
I eventually found that it was possible to delay Zach's check till later in the logic, past the check for "."'s and still catch all of the same cases without the change in semantics.

With the combination of all these changes, I made two versions of the patch - one that was the original patch changes plus the semantic change and another that added the cache as well. In testing I found that the version with the cache did performly slightly better (it caught a slightly broader set of cases) but really virtually all of the improvement was due to the locals check. 

Based on that, we decided the complexity of the cache was not worth the slight benefit and only the simpler version was included in 1.7.0-alpha4.
As it turns out, aleph seemed to be the project with the greatest benefit, going from 25.4 sec to 14.8 sec to run "lein test", a really significant reduction in time.
A number of other projects have also reported a signficant reduction in compile time with the alpha4 release.
I'm very happy we were able to clean this up and get it included.
Thanks Zach for filing the ticket and the patch!