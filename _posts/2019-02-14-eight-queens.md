---
title: Eight Queens
layout: post
--- 

The [Eight Queens puzzle][1] is a classic problem for computer programmers.  The goal is to
find all the ways to place eight queens on an 8-by-8 chess board such that no two queens
threaten each other.  The problem can be generalized to N-Queens on an N-by-N board.  The
Wikipedia article has a good discussion and lots of interesting links.  If you're working on
a homework assignment, you should start there before you borrow my code.

[1]: https://en.wikipedia.org/wiki/Eight_queens_puzzle

I'm a Clojure programmer so this article is going to explore some Clojure solutions.  You
can find examples in many languages at the [Rosetta Code][2] website, including a simple one
written in Clojure.  Here's a slightly modified version:

[2]: https://rosettacode.org/wiki/N-queens_problem#Short_Version

```clojure

(require '[clojure.math.combinatorics :as c])

(defn rosetta-queens [n]
  (filter (fn [x] (every? #(apply distinct? (map-indexed % x)) [+ -]))
          (c/permutations (range n))))
```

Typically, the chess board is represented as a two-dimensional array of positions in rows
and columns.  A brute force seach of every possible arrangement of 8 queens on 64 possible
positions is intractable.  The search space can be constrained by restricting the possible
arrangements such that no two queens can occupy the same row.  That reduces the problem to
finding a column for each queen, with the intuitive understanding that there will be one
queen per row.  Instead of explicitly listing the two-dimensional coordinates, we can
specify a vector of column numbers, with the index of each element implying the row number.
For example, a 4-queens solution might be `([0 1] [1 3] [2 0] [3 2])` in two-dimensional
notation.  The equivalent state is represented in one dimension as `[1 3 0 2]`.  The
`rosetta-queens` and most other solutions return the single vector result, and we will
follow that convention.

Let's take a minute to review the `rosetta-queens` function.  We want to arrange the queens
so that no two share the same row, column or either of the diagonals.  In other words, a
solution will have distinct assignments for each of those aspects.  The rows are naturally
distinct because they are simply the indices of the vector.  The columns are obviously
distinct as they are generated from permutations of the range expression.  Thus,
`rosetta-queens` generates candidates that are safe in terms of rows and columns, but it
still has to check if the diagonals cause conflicts.

It turns out that the sums and differences of the two-dimensional coordinates correspond to
the two diagonals.  Notice that two-dimensonal coordinates `[1, 2]` and `[4, 5]` are on the same
diagonal -- their differences are equal.  Similarly, `[0, 4]` and `[2, 2]` threaten each other
and their sums are the same.  Another way of thinking about diagonals is to say that the
slope between the positions must have an absolute value of 1.  Points `[a, b]` and `[c, d]`
threaten each other if `(= (Math/abs (- a c)) (Math/abs (- b d)))`.

The `rosetta-queens` solutions uses the `map-indexed` expression to get the row (as the
index) and the column from the candidate vector.  The `every?` expression checks both `+`
and `-` of those coordinates to check the two possible diagonals.  If all the diagonal
calculations are distinct for the queens, then we have a solution.

The `rosetta-queens` function is nice and short.  It uses built-in Clojure functions, plus
just one imported function `permutations` from the standard contribution library.  I think
this is a pretty good solution.  Its performance is acceptable and it's not too hard to
comprehend what it's doing once you see how the math works.

Naturally, there are a number of more efficient approaches.  The first possible improvement
that comes to mind is to try for a more incremental generation of positions.  After you
place a queen in one row, many positions in other rows can be ruled out.  If you add queens
sequentially to partial solutions, you can generate a set of solutions more efficiently than
evaluating all the permuations.

After doing some experiments, I came up with my own solution in Clojure.  I doubt there's
really anything new here as people have been working on the Eight Queens puzzle for over a
century.  In any case, I had fun trying out a few ideas and I thought I might share my
results here.

As we build up the solution by making assignments in row order, we need to evaluate the
possible positions in the next row.  For each position of row and column, we want a fast way
to check for conflicts with previously assigned queens.  The basic idea is to encode the
column and diagonal markers for each position using bits in a long.  The current state of
the board is just the `bit-or` of the previous state with the new queen's position suitably
encoded.  Conflicts can be found with a bitwise check.  Clojure is not known as a
bit-twiddling language, but all the bit functions you need are available.

Of course, we need to have non-overlapping codes for the diagonals and columns so we can set
the bits independently.  My little trick is to do the sums and differences with an extra argument
of the dimension.  An N-dimensional board has `2N - 1` diagonals in each direction.
With a 64-bit long, there are enough bits to hold the state of a 13 x 13 board -- that's 13
column bits + 25 bits for one diagonal + 25 bits for the other.  By the way, the `bit-set`
with a negative index works the way you would want it to -- `#(mod % 64)`.

First, let's see the code, and then we'll add a bit of explanation.

```clojure

(defn queens [^long dimension]
  {:pre [(<= 0 dimension 13)]}
  (let [add-queens (fn [qvs ^long row]
                     (for [qv qvs :let [conflicts (long (peek qv))]
                           col (range dimension)
                           :let [col (long col)
                                 rc (bit-or (bit-shift-left 1 col)
                                            (bit-shift-left 1 (+ row col dimension))
                                            (bit-shift-left 1 (- row col dimension)))]
                           :when (zero? (bit-and conflicts rc))]
                       (conj (pop qv) col (bit-or conflicts rc))))]
    
    (map pop (reduce add-queens [[0]] (range dimension)))))

```

In the `queens` function, we have a local `rc` that holds a long with three bits set
corresponding to the column and two diagonals markers.  The test for conflicts requires just
a single `bit-and` calculation and a `zero?` test.  We're using `reduce` to drive the
search, row by row, as it calls `add-queens`.  The `qvs` "state" is a collection of partial
assignments, where each assignment is a vector of queen columns plus an extra element, which
is a long integer holding the board state for the current assignments.  We can efficiently
`peek` and `pop` the last element of the vector, making it a convenient way to carry extra
bookkeeping data.  On each assignment, we `pop` off the old board state, then `conj` the new
queen column and the new board state.  Partial solutions that cannot be extended safely are
effectively dropped.  The initial value of `[[0]]` is a collection of a single vector
denoting the empty board with no conflicts.  When we get to the `reduce` result, we need to
`pop` off the bookkeeping data to return the desired final result.  There are a couple of
type hints and casts to let the code run with `*unchecked-math*` as a performance boost.

Although there's a limit on the dimension, the `queens` function certainly works well for
**Eight Queens**, which was our original target.  It's much faster than the simpler
`rosetta-queens` mainly because it does less work, but also because the encoding allows for
more efficient conflict checking.

If you read the Wikipedia article, you will find a link to a fast solution by
[Martin Richards (2009)][3].  (Credit is also given to Zongyan Qiu for publishing same
solution in 2002.)  After coming up with my solution, I went back to look at the Richards
approach.

[3]: https://www.cl.cam.ac.uk/~mr10/backtrk.pdf

The clever part of the Richards algorithm is how it keeps track of diagonal conflicts by
using bit-shifts.  When a queen is placed in a column, it threatens diagonally the columns
in the next row that are one less and one more.  If you combine the diagonal conflicts for
the assigned queens and shift once per row, you have all the conflicts for the next row.
That means you need three integers to keep track of the current columns and each of the
diagonals, which are designated "right" and "left" in the sense that they descend to the
right or left of the position of a queen.  Some bitwise logic reveals the open positions for
a queen in the next row.

I implemented a reasonable facsimile of the Richards algorithm in Clojure, but it was about
the same speed as my `queens` and somewhat more complicated.  However, I could not help
feeling that there must be a better way.  I doubt Donald Knuth would approve, but I got a
little carried away with some experiments focused on performance.

Perhaps I should be embarrassed by my `fastest-queeens`, but I'm also a little bit proud
that it shows a pretty good performance improvement.  There's a lot of code to handle bit
twiddling along the lines of the Richards approach.  I'm basically encoding the Richards
bookkeeping into a single long and shifting the bits as necessary.  (If you're thinking
*this guy should just give up and use C*, you're probably right!)  I'm limiting my solutions
to dimensions less than 14, so I can fit all the bookkeeping bits in one long.  I'm not
going to explain all the details.  Tweet [@miner][5] if you have questions.  Suffice it to
say, it's roughly twice as fast as my previous `queens` solution.

[5]: https://twitter.com/miner

```clojure
	
(defn fastest-queens [^long dimension]
  {:pre [(<= 0 dimension 14)]}
  (let [mask (dec (bit-shift-left 1 dimension))
        mask16 (bit-shift-left mask 16)
        mask32 (bit-shift-left mask 32)
        mask48 (bit-shift-left mask 48)

        shift-conflicts (fn [^long conflicts]
                          (let [rdiag (bit-and mask16 (unsigned-bit-shift-right conflicts 1))
                                ldiag (bit-and mask32 (bit-shift-left conflicts 1))
                                qcols (bit-and mask48 conflicts)]
                            (bit-or rdiag ldiag qcols
                                    (unsigned-bit-shift-right rdiag 16)
                                    (unsigned-bit-shift-right ldiag 32)
                                    (unsigned-bit-shift-right qcols 48))))
        
        add-queens
        (fn [qvs]
          (loop [qvs qvs xvs ()]
            (if-let [qv (first qvs)]
              (recur (rest qvs)
                     (let [^long conflicts (peek qv)]
                       (loop [vs xvs n (bit-and-not mask conflicts)]
                         (if (zero? n)
                           vs
                           (let [q (Long/numberOfTrailingZeros n)
                                 colmask (bit-shift-left 0x0001000100010001 q)]
                             (recur (conj vs (conj (pop qv) q
                                                   (shift-conflicts (bit-or conflicts colmask))))
                                    (bit-and-not n colmask)))))))
              
              xvs))) ]

    (mapv pop (nth (iterate add-queens [[0]]) dimension))))

```


Execution time from [Criterium][4] for the Eight Queens puzzle benchmark on my oldish iMac.

[4]: https://github.com/hugoduncan/criterium/

| Function       |    Time     |
| --------       |    -------: |
| rosetta-queens |     61.3 ms |
| queens         |    906.9 µs |
| fastest-queens |    374.0 µs |

-----

You can find my [Queens][6] project on GitHub.

[6]: https://github.com/miner/queens



