---
title: Generating Generators
layout: post
---

## Presented at Clojure/conj 2014

Property-based testing provides good test coverage and automatic shrinking of failure
cases. However, coding universal properties with the *test.check* generator combinators is
somewhat of an art. In many cases, it's easier to start from a declarative description of
the test data. The *Herbert* library automatically generates test.check generators from EDN
schemas. Learn how schemas can offer simplified testing, easier maintenance and better
documentation for your Clojure code.

[video](https://www.youtube.com/watch?v=4JGu33WF0Us)

[slides](https://speakerdeck.com/miner/generating-generators)

[Herbert](https://github.com/miner/herbert)
