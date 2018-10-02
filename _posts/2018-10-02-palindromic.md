---
title: Longest Palindromic Substring
layout: post
--- 

I watched the [Apropos Clojure #18][1] video cast the other day.  The group worked on an
example problem of finding the longest palindromic substring of a given string.  A palindrome
is a string of characters that's the same in reverse order, such as "abba".

[1]: https://www.youtube.com/watch?v=elF9BPa0Np4

The panelists started with a classic generate and test approach.  Their solution looked
something like this:

```clojure
(defn palindrome? [s]
  (= (seq s) (reverse s)))

(defn substrings [s]
  (let [mx (inc (count s))]
    (for [start (range mx)
          end (range (inc start) mx)]
      (subs s start end))))

(defn longest-palindrome [s]
  "Return the longest substring of s that is a palindrome"
  (apply max-key count (filter palindrome? (substrings s))))
```

I have a couple of suggestions for improvements.  The first idea is to build the
substrings in length order, longest first.  A slight rearrangement of the `for`
comprehension will do the trick.

```clojure
(defn substrings1 [s]
  (let [cnt (count s)]
    (for [len (range cnt 0 -1)
          start (range (inc (- cnt len)))
          :let [end (+ start len)]]
      (subs s start end))))

(defn longest-palindrome1 [s]
  "Return the longest substring of s that is a palindrome"
  (first (filter palindrome? (substrings1 s))))
```

With `longest-palindrome1`, the first palindrome found is known to be the longest as the
candidates are generated in length order.  The `filter` and `substrings1` functions are lazy
so the process terminates as soon the first palindromic substring is generated.  The smaller
substrings are not realized.  In my tests, `longest-palindrome1` was 2-3 times faster than
the original.

The second improvement is to use a faster palindrome test.  It's not obvious, but
`clojure.string/reverse` is pretty fast (due to Java interop) and definitley beats using
Clojure sequences of characters.  (I use "str" as a namespace alias for "clojure.string".)

```clojure
(defn palindrome2? [s]
  (= s (str/reverse s)))

(defn longest-palindrome2 [s]
  (first (filter palindrome2? (substrings1 s))))
```

The `longest-palindrome2` runs about 10 times faster than `longest-palindrome1`.  That's a
nice performance win for a small change.

The fastest implementation I came up with uses Java interop to access characters within the
original string without creating new strings.  Only the longest palindromic substring
needs to be realized.  The ^String type annotations help the Clojure compiler pick the
correct Java methods.

```clojure
(defn substr-pal? [^String s start end]
  (loop [front start back (dec end)]
    (or (>= front back)
        (and (= (.charAt s front) (.charAt s back))
             (recur (inc front) (dec back))))))

(defn longest-palindrome3 [^String s]
  (let [cnt (.length s)]
    (first (for [len (range cnt 0 -1)
                 start (range (inc (- cnt len)))
                 :let [end (+ start len)]
                 :when (substr-pal? s start end)]
             (subs s start end)))))
```

You can find my code in this [gist][2].  

[2]: https://gist.github.com/miner/0c8ccad10cb8018ab6e49e3de2825d54

I wrote a benchmark function `run-benchmarks` which calls Criterium to get timings.  Here
are the results on my machine:

* Testing miner.pal/longest-palindrome, Execution time mean : 633.145389 µs

* Testing miner.pal/longest-palindrome1, Execution time mean : 222.759123 µs

* Testing miner.pal/longest-palindrome2, Execution time mean : 22.415052 µs

* Testing miner.pal/longest-palindrome3, Execution time mean : 7.496435 µs
