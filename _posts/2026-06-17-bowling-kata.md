---
title: Bowling Kata
layout: post
---

The **Bowling Game Kata** is an oldie, but a goodie.  The programming problem is to score a
game of bowling.  The more formal name of the game is [Tenpin Bowling][1] to distinguish it
from other related games.

[1]: https://en.wikipedia.org/wiki/Tenpin_bowling

Here's a link to the classic solution in Clojure by [Stuart Halloway][2].  The input for
that version was simply a sequence of numeric rolls with no special marks.

[2]: https://github.com/stuarthalloway/clojure-bowling

For a little extra fun, I prefer a version that follows the standard notation for
bowling, in which a game is represented as a string with an *X* for a strike, */* for a
spare, *-* for a gutter ball (no pins), and *1-9* for hitting so many pins.  The game is
played as ten frames, usually with two rolls per frame.  However, as a special case, taking
down all ten pins with the first ball results in a strike.  The frame score for a strike is
10 plus the next two balls.  If the first ball is less than 10, a second ball is rolled.  If
the second ball takes down the rest of the pins, you have a spare.  The frame with a spare
scores 10 plus the next ball.  Any other combination of two balls in a frame scores the sum
of the two balls.  If the player scores a strike or spare in the tenth frame, he is allowed
to roll extra balls as required to score the final frame.  (The extra balls do not count as a
new frame.)  The final score is the sum of the ten frame scores.

A perfect game is all strikes "XXXXXXXXXXXX" -- that's twelve strikes for ten frames plus
the two extra balls.  The final score is 300.

The string notation also allows spaces to be used for readability.  The spaces should be
ignored for scoring.  Here's a test that applies a `score` function.  (The reader is welcome
to adapt it to his favorite test framework.)

```clojure
(defn score-test [score]
   (assert (= (score "35 6/ 7/ X 45 X X X XXXX") 223))
   (assert (= (score "11 11 11 11 11 11 11 11 X 11") 30))
   (assert (= (score "11 11 11 11 11 11 11 11 11 X 11") 30))
   (assert (= (score "XXXXXXXXXXXX") 300))
   (assert (= (score "9-9-9-9-9-9-9-9-9-9-") 90))
   (assert (= (score "5/5/5/5/5/5/5/5/5/5/5") 150))
   (assert (= (score "12 12 12 12 12 12 12 12 12 0/X") 47))
   (assert (= (score "XX 3/ 4/ X 54 7/ X X X 3/")  198))
   (assert (= (score "XX 3/ 4/ X 54 7/ X X X 37")  198))
   true)
```

The problem statement does not require the detection of illegal inputs.  That would be a
good extension.  For example, you can never have a spare in the first ball of a frame, and
two balls in a frame cannot sum up to more than 10.  My solution does not handle those
errors.  I am assuming valid input strings.  (Famous last words.)

Most of the Clojure solutions I've seen, parse the string into a sequence of integers and
then do the appropriate calculations in a loop.  My solution is a little different in that I
am parsing the string into a vector of integers and using a special value to mark a spare
(-1).  The vector of balls representation allows me to use `reduce-kv` to drive the function
and have convenient access to other balls by offset from the current index.  I should note
that `reduce-kv` over a vector gives you the index as the *key*.  Clojure has some
optimizations that make `reduce-kv` fast.  In this case, it's especially handy to have an
index so that you can access nearby balls.

The mapping from a character digit into an integer is mostly done by converting to *long* and
subtracting `(long \0)` but with special cases for the *X* and *-*.  It is a happy accident
that */* maps into -1 in this scheme.  That makes it convenient to test for a spare with the
`neg?` function.

The other trick I'm using is to count down from 20 "half frames" with special handling of
strkes, which count as 2 half frames.  We need to count the frames in order to avoid
confusion with the possibility of extra balls for a mark in the final frame.  When `fc` is
even, it means we're looking at the first ball of a frame.  The *state* of the reduction
function is a vector of the half-frame count and the running score, `[fc sc]`.  We score a
frame only when it is complete (a strike or two balls).  Clojure purists might prefer to use
a map for the *state*.  For a small number of slots, positional values seem reasonable to
me.  At then end, we only need the score so we call `peek` on the result of the reduction.

Using the `reduce-kv` over a vector of balls gives better performance than the common
sequence-oriented loop-style solutions.  It is an eager approach.  For a small problem
like this, there's really no advantage to being lazy.  One other performance note:  using
`clojure.string/replace` to remove the spaces is faster than dealing with the spaces as
characters in a sequence.  In general, the string functions are faster than I naively
expected, probably thanks to Java optimizations under the hood.

Here is my bowling **score** implementation.

```clojure

(require '[clojure.string :as str])

(defn score [game]
  (let [bv (mapv (fn [c] (let [x (- (long c) (long \0))] (case x 40 10 -3 0 x)))
                 (str/replace game " " ""))]
    ;; note X maps to 40, \ to -1, - to -3.  The test for a spare mark is neg?
    (peek
     (reduce-kv (fn [[fc sc] i b]
                  (if (zero? fc)
                    (reduced [sc])
                    (if (even? fc)
                      ;; first ball of frame
                      (if (= b 10) ;strike
                        (let [b2 (bv (+ i 2))]
                          [(- fc 2)
                           (if (neg? b2) (+ sc 20) (+ sc 10 (bv (inc i)) b2))])
                        ;; don't score first ball, we will account for it on second ball
                        [(dec fc) sc])
                      ;; second ball of frame, recover the previous ball if necessary
                      [(dec fc)
                       (if (neg? b) (+ sc 10 (bv (inc i))) (+ sc (bv (dec i)) b))])))
                ;; init at 20 half-frames, and zero score
                [20 0]
                bv))))
```
