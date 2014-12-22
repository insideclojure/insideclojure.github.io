---
layout: post
date: 2014-12-21
title: Creating a reducible repeat
---

One fun ticket ([CLJ-1603](http://dev.clojure.org/jira/browse/CLJ-1603)) that I worked on recently involved retrofitting some existing sequence "generator" functions to be directly reducible.
Specifically, the clojure.core functions cycle, iterate, and repeat.
I'll just talk about repeat in this post and maybe come back to the others in the future.

*Note: This ticket and patch have not yet been screened or integrated into Clojure and their implementation may change!*

Collections or sequences can mark themselves as being internally reducible by implementing either IReduceInit or IReduce. 
Prior to 1.7, IReduce covered both forms of reduce (with and without the initial value).
As of 1.7, these forms have been split apart - IReduceInit implements just the version with the initial value.
IReduce now extends IReduceInit and adds the no-init version.

The cycle, iterate, and repeat functions all return lazy seqs, created via the lazy-seq macro.
Making them also support IReduceInit meant either modifying the existing (Clojure) implementation or creating a new Java-based implementation.
I poked around at implementing a Clojure version but there are some problematic issues because of where these functions are defined in clojure.core (in particular, the late stage where deftype is defined while loading clojure.core).

Ultimately, I wrote the implementation in Java. Here's a partial view of the class (with some less important parts removed):

{% highlight java %}
public class Repeat extends ASeq implements IReduce {

private static final long INFINITE = -1;

private final long count;  // always INFINITE or >0
private final Object val;

private Repeat(long count, Object val){
    this.count = count;
    this.val = val;
}

public static Repeat create(Object val){
    return new Repeat(INFINITE, val);
}

public static ISeq create(long count, Object val){
    if(count <= 0)
        return PersistentList.EMPTY;
    return new Repeat(count, val);
}

public Object first(){
    return val;
}

public ISeq next(){
    if(count == INFINITE) {
        return this;
    } else if(count == 1) {
        return null;
    } else {
        return new Repeat(count-1, val);
    }
}

public Object reduce(IFn f, Object start){
    Object ret = start;
    if(count == INFINITE){
        while(true){
            ret = f.invoke(ret, val);
            if(ret instanceof Reduced)
                return ((IDeref)ret).deref();
        }
    } else {
        for(long i=0; i<count; i++){
            ret = f.invoke(ret, val);
            if(ret instanceof Reduced)
                return ((IDeref)ret).deref();
        }
        return ret;
    }
}
}
{% endhighlight %}

Most importantly, we extend ASeq which gives us much of the sequence and collection infrastructure. At that point, it's really a matter of implementing first/next in the right way, and of course out primary target, reduce.

There are really two cases for repeat - infinite and finite. Infinite repeats are easy - each is a single instance that returns the repeated value for first() and itself for next(). Reducing over an infinite repeat just involves repeatedly invoking f on the constant value until a reduced value is returned.

The finite repeat requires the use of a counter to know when the sequence should end. Each new value creates a new immutable instance of Repeat with a reduced counter. Reduction again involves just repeatedly invoking f on the constant value until either the counter is out or a reduced value is returned.

So, why bother doing this at all? 
The benefit is performance. 
Functions like repeat (cycle, iterate, and also range) form the basis of generated sequences of values. 
Reducing directly over them (in the reduce loop above) is more efficient than looping through a sequence.
 
A transducible process will make one pass through the input collection, or in this case, the generated sequence. 
The stack of composed functions will be applied to each input once.
Wen you start to combine reducible sources and multiple transducer functions, you see a big difference in performance.

Here's a few simple tests of the updated repeat function:

| expr | 1.6.0 | 1.7.0-alpha4 | 1.7.0-alpha4 + new repeat |
| ----- | ---- | ---- | ---- |
| (into [] (repeat 1000 1)) | 107 µs | 97 µs | 5 µs |
| (reduce + 0 (repeat 1000 1)) | 112 µs | 112 µs | 17 µs |
| (into [] (take 1000) (repeat 1)) | n/a | 75 µs | 33 µs | 

The first two tests demonstrate reduction over a finite repeat.
The final one is a transducer applied over an infinite repeat.

One implementation alternative I considered was splitting the finite and infinite cases into separate classes and that may still be a worthwhile change.
