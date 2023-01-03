+++
title ="Hands-on Learning: Raft & Consensus Algorithms (Part 3 - Does this actually work?)"
draft = true
[taxonomies]
tags = ["distributed systems", "consensus", "raft" ]
+++

So I've been reading a lot lately about distributed systems and the building blocks used to build distributed systems from the ground up. This series of posts is going to focus on consensus algorithms. Or one algorithm in particular, the Raft consensus algorithm.

I will walk you through the Raft white paper and how to actually implement it.

My goals in attempting to write my own version of Raft are:

- Develop a deeper, more practical understanding on Raft & consensus algorithms. For me things tend to gel in my mind better when I do vs just raad.
- Learn Rust: I am choosing to implement this using Rust mostly because I've been intrigued by the Rust language and have been wanting to practice. Thsi exercise presents a non-trivial problem to solve with Rust.
- Share my learnings with anyone reading this

## What do we mean by "consensus"?
