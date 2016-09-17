---
published: false
title: JIT code generation in DBKit
layout: post
---
When I started working on the query engine in [DBKit](https://github.com/mtanski/dbkit) I knew I wanted to build a really fast query engine that could run on the host CPU. Ideally, it would be also able to offload work to GPU or other CoProcessors like the Xeon Phi. In the columnar execution engine this essentially means SIMD or vectorized code. I knew this was going to be a big undertaking. I've previously worked with and hacked Google's [Supersonic](https://github.com/google/supersonic) which my project takes inspiration from and Supersonic it self had limited SIMD support. Supersonic implemented a few expression but it primarily on the C++ compiler generate good code using (using up to SSE4 support). It was also painfully aware that Rust's SIMD support is currently in flux... it's not non-existent but also there's not a story there yet. A little false optimizing upfront is not always a bad thing. So in the grand tradition of that  I started on basic plumbing (allocator, schema, collections) and getting first operation working. 

Learning Rust and getting the plumbing working provided enough distraction for a few weeks. Rust can have a steep learning curve (vertical line) to get going you need to use unsafe, nightly features when you're getting going.



 I was going to spending a lot time working on vectoring common operations like all the relational filter operations. I knew it was going be a big undertaking, esp. given Rust's influx SIMD support. 

