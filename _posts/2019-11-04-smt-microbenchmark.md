---
title: SMT Microbenchmark
layout: post
--- 

I recently came across a blog post by John Jacobsen with the title:
[*From Elegance to Speed*][1].  The author had a short function written in Clojure that he
ported to Common Lisp for an impressive performance improvement.  Along the way, he made
some changes in the approach for his Common Lisp implementation but didn't apply those ideas
to his Clojure code.  I thought it was worth a try to see what we could do with Clojure.

[1]: http://johnj.com/from-elegance-to-speed.html

First, any microbenchmarking experiment should be taken with a grain of salt.  The goal of
Clojure is to be a powerful tool for writing large programs to solve complex problems.  It's
not necessarily optimized for snippets of simple arithmetic.  For more information about
Clojure, please see the [Rationale][2].

[2]: https://clojure.org/about/rationale

With that disclaimer out of the way, let's look at the original Clojure function, called `smt-8`.

```clojure

(defn smt-8 [times]
  (->> times
       (partition 8 1)
       (map (juxt identity
                  (comp (partial apply -)
                        (juxt last first))))
       (filter (comp (partial > 1000) second))))

```

The input is a monotonically increasing sequence of long integers representing time data.
The function walks through the sequence looking for clusters (in a sliding window of size 8)
that are close in time (within a 1000 milliseconds).  The result is a sequence of the
desired clusters and their widths.  From domain knowledge, we expect that such clusters are
relatively rare in the input data set.

I think the original `smt-8` is reasonable.  It uses a few higher-order functions that might
not be recognized by non-Clojure programmers.  Here's a slightly modified Clojure
implementation that runs a bit faster for me.

```clojure
;; modest improvement using `keep`, about 20% faster
(defn smt-8a [times]
  (->> times
       (partition 8 1)
       (keep (fn [part]
               (let [diff (- (last part) (first part))]
                 (when (> 1000 diff)
                   [part diff]))))))
```

The `keep` function is a something like a combination of `map` and `filter` and I think it
makes the code a bit more readable, but that's naturally a matter of taste.

In another [post][3], the author explains how he was experimenting with laziness and
infinite sequences when he wrote the original `smt-8`.  Many of the core functions in
Clojure support lazy sequences which makes Clojure an excellent tool for this kind of
exploration.  However, for his comparison with Common Lisp, he decided to punt on the
question of laziness.  That's fine if you don't need laziness, but I want to point out to
casual readers that laziness can be a useful feature especially when working with large data
sets.  However, laziness does require a bit of overhead that you might prefer to avoid when
doing competitive benchmarking.  Common Lisp is not lazy by default so you would need to do
some extra work or find a good library if you wanted a lazy solution in Common Lisp.

[3]: http://johnj.com/lazy-physics.html

If we don't need a lazy solution, and we have a finite input sequence, we can make things
significantly faster using a vector as input.  In particular, we don't have to create
intermediate partitions anymore.  We can just check the values at the appropriate indices.
(Remember the input is monotonic so only the end points of a potential cluster matter.)
Assuming it's safe to convert the input to a vector, we can try something like the
following.

```clojure

;; for better performance, the original data should already by a vector
(defn smt-8for [times]
  (let [v (vec times)
        width 8]
    (for [i (range (- (count v) (dec width)))
          :let [diff (- (v (+ i (dec width))) (v i))]
          :when (> 1000 diff)]
      [(subvec v i (+ i width)) diff])))

```

In `smt-8for`, we're using a `for` comprehension to drive the calculations.  We're also
taking subvectors out of the original vector to make our clusters, but only if we have
already determined that the cluster belongs in the result.  We are eliminating the creation
of many partitions that would end up being filtered out.  The `subvec` function does not
copy, it just keeps an offset into the original vector.  It's fast but it prevents garbage
collecting the original vector as long as the subvectors are alive.  If that's not
acceptable, you can make a copy of the cluster instead.  In my tests, `smt-8for` is more
than 30 times as fast as the original `smt-8`.

You can get a bit more performance if you're careful about boxed math.  Normally, you
shouldn't care so please don't get carried away with this next tip.  You can read the
documentation on [type hints][4], _\*warn-on-reflection\*_ and _\*unchecked-math\*_ for more
information.

[4]: https://clojure.org/reference/java_interop#typehints

```clojure

(set! *unchecked-math* :warn-on-boxed)

(defn smt-8forh [v]
  (let [width 8]
    (for [^long i (range (- (count v) (dec width)))
          :let [diff (- ^long (v (+ i (dec width))) ^long (v i))]
          :when (> 1000 diff)]
      (list (subvec v i (+ i width)) diff))))

```

Adding the type hints squeezes a bit more performance out of the same code.  I measured
better than 20% increase in performance, but remember this code is doing simple arithmetic
and almost no "business logic" so we're mostly measuring what would normally be considered
overhead costs.

Clojure has another feature called [Transducers][5] that can improve performance for chains
of operations over input sequences.  Here's a slightly complicated implementation using
transducers.  To get extra speed, this version builds a *list* of results, which is slightly
faster than appending to a vector.  It runs the `range` calculation in reverse to compensate
for the natural addition at the head of the output list.  I have to admit that the slight
performance improvement isn't worth increasing the obscurity of the code as I've done in
this case.  However, the general point is that transducers are often a win for performance
and clarity.  In my tests, this transducer version is marginally faster than the
previous solution.

[5]: https://clojure.org/reference/transducers


```clojure

(defn smt-8x [v]
  (let [width 8]
    (into ()
          (keep (fn [i]
                  (let [d (- ^long (v (+ (dec width) ^long i)) ^long (v i))]
                    (when (> 1000 d)
                      (list (subvec v i (+ ^long i width)) d)))))
          (range (- (count v) (inc width)) -1 -1))))

```

Now, if you're desperate for better performance and you have control over your input data,
you can use Java arrays, which have built-in support in Clojure.  For appropriate
situations, they are faster than the more refined Clojure collections, especially for direct
access to data that is not shared.  Of course, you can cause bugs with wanton mutations so
you have to be careful.  Frankly, I would rarely make this trade-off in real code.  On the
other hand, array access is a useful tool when playing the microbenchmark game.  If you have
a Clojure collection, you can populate a Java array of longs with the Clojure function
`long-array`.  Elements are accessed with `aget`.  Our new `smt-8arr` is similar to
`smt-8forh` but adapted for the Java array input.


```clojure

;; use Java array for speed
(defn smt-8arr [^longs larr]
  (let [width 8]
    (for [^long i (range (- (alength larr) (dec width)))
          :let [diff (- (aget larr (+ i (dec width))) (aget larr i))]
          :when (> 1000 diff)]
      (list (for [j (range i (+ i width))] (aget larr j)) diff))))

```

On my machine, I get about a 200x improvement over the original code.  Of course, I'm
assuming the data can be stored in the appropriate `long-array` in advance.  In real world
applications that do more than arithmetic, the business logic usually dominates so the
overhead of laziness and persistent data structures is minimal by comparison.  As I remarked
earlier, I would not normally resort to Java arrays unless I had to inter-operate with Java
code.

The big win in this case was avoiding object creation for partitions that were immediately
discarded.  I think most people would prefer the `for` comprehension as shown in my
`smt-8for` example.  That version gave respectable peformance from clean code (IMHO).


Execution times from [Criterium][8] on my oldish iMac.

[8]: https://github.com/hugoduncan/criterium/

-----

| Function  | Time (ms) |
| -----     |  -----: |
| smt-8     |    1572 |
| smt-8a    |    1341 |
| smt-8for  |      47 |               
| smt-8forh |      31 |
| smt-8x    |      25 |
| smt-8arr  |       7 |

-----


The source code for this post is available as a [gist][7].

[7]: https://gist.github.com/miner/bf8b8221a8e31e3438c8620a81fbc16c

