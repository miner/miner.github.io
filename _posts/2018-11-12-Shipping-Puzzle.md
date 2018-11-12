---
title: Shipping Puzzle
layout: post
--- 

Kevin Lynagh posted a couple of solutions to the "Shipping Puzzle" described in his
[Exploring a shipping puzzle][1] blog post and a [follow up post][2].  I enjoyed working on
my own Clojure version, which you can find [here on Github][3].

[1]: https://kevinlynagh.com/notes/shipping-puzzle/
[2]: https://kevinlynagh.com/notes/shipping-puzzle/part-2/
[3]: https://github.com/miner/shipping

I took a little bit different approach by keeping the `legs` partitioned by `:dow` and
sorted by `:start`.  Similarly, the `paths` are partitioned by the last `:dow` and sorted by
last `:end`.  Knowing that they're sorted simplifies the matching process so it requires
less searching for a connection.  I wasn't sure if the overhead of re-sorting would be a
performance problem, but my `shipping` function was still competitive with other Clojure
solutions.

We can improve the performance if we encode the data differently.  Sorting
on integers is faster than strings.  Instead of using string-valued attributes, we can
convert to integer codes for the "day of week", "start" and "end" cities.  The data were
originally in maps so it was simple to add a few new attributes to the `legs` as they were
loaded from the file.  I called the new integer attributes `:day`, `:depart`, and `:arrive`
-- corresponding to the original `:dow`, `:start` and `:end` keys.  With a few small
changes, my solution `shipping1` runs about 30% faster.
