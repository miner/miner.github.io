---
title: Tarantella Queens
layout: post
--- 

Mark Engelberg recently announced an update to his [Tarantella][1] library for Clojure.
Tarantella is an implementation of Knuth's [Dancing Links][7] algorithm for solving
[exact cover][2] problems.  Mark gave an excellent talk at the 2017 Clojure Conj entitled
["Solving Problems Declaratively"][3] which explains the algorithm and demonstrates how
Tarantella works.

[1]: https://github.com/Engelberg/tarantella

[7]: https://en.wikipedia.org/wiki/Dancing_Links

[2]: https://en.wikipedia.org/wiki/Exact_cover

[3]: https://youtu.be/TA9DBG8x-ys

Tarantella starts with a matrix of constraints.  Each row in the matrix has certain columns
set (marked 1) for a corresponding constraint.  Tarantella searches for a covering set of
rows such that each column is set once in the solution set.  For convenience, the constraint
matrix can be specified as a set of marked columns rather than listing all the ones and
zeroes explicitly.

In my previous [Eight Queens][4] implementation, I used the terms "row" and "column" to
label the positions on the board.  Chess players would probably prefer "rank" and
"file".  To avoid confusion, in this post I will refer to the positions as X-Y coordinates
on an N-by-N board.  See my [blog post][4] for an explanation of how the N-Queens problem is
essentially concerned with finding unique positions in terms of X, Y, and the two diagonals.

[4]: http://conjobble.velisco.com/2019/02/14/eight-queens.html

To work with Tarantella, we need to encode our constraints for the positions of the queens
as Tarantella columns.  X and Y each have `N` possible values.  The diagonal constraints
correspond to the sums and differences of X and Y at each square.  Thus, there are `2N-1`
possible values for each diagonal.  We have to designate each constraint as a separate
column.  That makes a total of `6N-2` columns.  It's convenient to encode X as the first N
columns (`0` to `N-1`), and Y as the next N columns (`N` to `2N-1`).  These constraint
columns must be set exactly once to get unique placements in the X and Y coordinates.  Each
diagonal constraint is assigned sequentially to an additional `2N-1` columns.  Note that the
diagnonal constraints are declared as optionals for Tarantella, which means they can be set
once or not at all for a solution.

The solutions returned by `dancing-links` are row numbers from the original constraints.  We
can convert back to Y coordinates using `(rem ROW N)`.  The final result follows the
convention used in my other N-Queens solutions.  Each solution is a vector of Y coordinates
for the queens.  The X coordinates are implied by the index order.

```clojure

(require '[tarantella.core :as t])

(defn queens-constraints [n]
  (for [x (range n)
        y (range n)]
    [x (+ n y) (+ n n x y) (+ (* 5 n) (- x y 2))]))

;; The rows in a solution aren't guaranteed in any particular order so we
;; need to sort first, then decode queen placements from the row numbers.
(defn solve-queens [n]
  (map (fn [sol] (mapv #(rem % n) (sort sol)))
       (t/dancing-links (queens-constraints n)
                        :optional-columns (range (* 2 n) (- (* 6 n) 2)))))

```


Execution time from [Criterium][5] for the Eight Queens puzzle benchmark on my oldish iMac.

[5]: https://github.com/hugoduncan/criterium/

-----

| Function       |    Time     |
| --------       |    -------: |
| solve-queens   |    931.9 µs |
| queens         |    928.8 µs |
| fastest-queens |    383.0 µs |

-----

So the Tarantella solution is on par with my previous `queens` solution, which I thought was
moderately clever.  The Tarentalla approach is also much simpler code than my bit-twiddling
`fastest-queens`.  I highly recommend considering Tarantella for your future puzzle solving
needs.  It's nice to have someone else do the hard work for you.

You can find my updated [Queens][6] sample code on GitHub.

[6]: https://github.com/miner/queens
