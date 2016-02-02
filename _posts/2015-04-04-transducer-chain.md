---
title: Transducer Chain
layout: post
---

[Transducers](http://clojure.org/transducers) are a new feature included in recent Clojure
1.7 alpha releases.  I've been playing with them a bit.  My code is on github:
[Transmuters](https://github.com/miner/transmuters).

As the documentation says, "Transducers are composable algorithmic transformations."  They
are functions of a particular form that end up being useful in a variety of contexts.  The
implementer needs to fullfill certain requirements in order to play nicely with the system.
The payoffs are conceptual simplicity and good performance.  The user doesn't need to
understand all the magic of the implementation.  He can simply compose the built-in
transducer functions to get the desired result with minimal conceptual overhead.

Composability is an important part of the design.  Clojure transducers compose using the
same `comp` function that we've had in Clojure since the beginning.

Let's take a simple example:

```clojure
(def xs (interpose 0 (range 10)))
;=> (0 0 1 0 2 0 3 0 4 0 5 0 6 0 7 0 8 0 9)

;; traditional nested collection function calls
(map inc (remove zero? (take 10 xs)))
;=> (2 3 4 5)

;; when nesting gets deep, the ->> "thread-last" macro is a convenient alternative
(->> xs (take 10) (remove zero?) (map inc))
;=> (2 3 4 5)

;; using a transducer
(sequence (comp (take 10) (remove zero?) (map inc)) xs)
;=> (2 3 4 5)

(transduce (comp (take 10) (remove zero?) (map inc)) * 1 xs)
;=> 120
```

It occurred to me that instead of running just one transducer across the whole collection
(or stream of inputs), it might be useful to run different transducers on portions of the
input.  The idea was to make a chain of transducers.  When one transducer terminates (as
with `take`), the next one can pick up where it left off.  Naturally, if a transducer
doesn't terminate, the other transducers in the chain will never see any input.

```clojure
(def process (comp (take 10) (remove zero?) (map inc)))

(sequence process xs)
;=> (2 3 4 5)

(sequence (chain process (map -)) xs)
;=> (2 3 4 5 -5 0 -6 0 -7 0 -8 0 -9)

(transduce (chain process (map -)) + 0 xs)
;=> -21

;; filter handles all the input		
(sequence (chain (filter odd?) (map -)) xs)
;=> (1 3 5 7 9)

;; chaining composed transducers
(sequence (chain (comp (take 7) (map (partial * 10)))
                 (comp (drop 5) (take 3) (map inc)))
          xs)
;=> (0 0 10 0 20 0 30 7 1 8)

;; traditional form using concat for the parts
(concat (->> xs (take 7) (map (partial * 10)))	  
        (->> xs (drop (+ 5 7)) (take 3) (map inc)))
;=> (0 0 10 0 20 0 30 7 1 8)
		
```

Notice that the traditional collection functions have to concat multiple forms over the
same collection, making sure that the `drop` accounts for all input used by the first
form.  In contrast, the chained form is nicely localized.

The naturally terminating transducers are `take` and `take-while`.  In other cases, the
*reducing function* in a `transduce` call might use `reduced` to terminate.  This can get a
little tricky regarding the handling of the input that caused termination.  With `take`, the
process knows exactly when it has taken the proper amount, but `take-while` needs to see one
extra input to decide when to terminate the process.  Notice in the following example what
happens with the 5.

```clojure
(sequence (chain (take 10) (filter odd?)) xs)
;=> (0 0 1 0 2 0 3 0 4 0 5 7 9)

(sequence (chain (take-while #(< % 5)) (filter odd?)) xs)
;=> (0 0 1 0 2 0 3 0 4 0 7 9)
```

In the above example, the `take` transducer takes the first 10 inputs and leaves the rest
for the `filter` to consider.  The `take-while` version is subtly different.  The
`take-while` terminated when it saw the 5, but that 5 was not seen by the `(filter odd?)`
transducer.  Once it was consumed by the `take-while`, it was lost.  Most people probably
expected both examples to produced the same result.

So that's a problem for the `chain` combinator.  Some terminating transducers consume one
extra input in order to know when to stop.  Makes sense, and normally it doesn't matter
because the process is going to ignore the rest of the input anyway.  However, this
reasonable behavior can make it appear that the `chain` transducer is skipping an input.

My solution was to hack what I'm calling a *pseudo-transducer*.  I named it `pushback`.  Add
that to your chain of transducers after one that consumes an extra input, and that input
will be given to the next transducer as its first input.  Of course, `pushback` is not
really a transducer, but it superficially looks like one when it's in a `chain`.  In
reality, it serves as a marker to tell the `chain` to give that burned input to the next
transducer, which ends up giving us the desired result.

```clojure
(sequence (chain (take-while #(< % 5)) pushback (filter odd?)) xs)
;=> (0 0 1 0 2 0 3 0 4 0 5 7 9)
```

It's not exactly pretty but it was the best idea I could come up with to handle the
situation.

The source for `chain` and related bits are on github:
[Transmuters](https://github.com/miner/transmuters).  I'll write up some more details when I
get the chance.
