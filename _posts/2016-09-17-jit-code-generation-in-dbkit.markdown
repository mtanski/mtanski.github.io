---
published: false
title: JIT code generation in DBKit (part 1)
layout: post
tags: [DBKit, Rust, SIMD, LLVM, JIT]
---
When I started working on the query engine in [DBKit](https://github.com/mtanski/dbkit) I knew I wanted to build a really fast query engine that could run on the host CPU. Ideally, it would be also able to offload work to GPU or other CoProcessors like the Xeon Phi. In the columnar execution engine this essentially means SIMD or vectorized code.

### Vectorizing the query engine

I knew this was going to be a big undertaking. I've previously worked with and hacked Google's [Supersonic](https://github.com/google/supersonic) which my project takes inspiration from and Supersonic it self had limited SIMD support. Supersonic implemented a few expression but it primarily on the C++ compiler generate good code using (using up to SSE4 support). It was also painfully aware that Rust's SIMD support is currently in flux... it's not non-existent but also there's not a good story there yet. 

A little false optimism upfront is not always a bad thing. So in the grand tradition of that  I started on basic plumbing (allocator, schema, collections) and getting first operation working. Learning Rust and getting the plumbing working provided enough distraction for a few weeks. After getting that working it was time to implement the FILTER operation and you can't really implement that without being able to supporting enough expressions that you can filter on.

Initially I played around with implementing some simple expressions like IsNull, NotNull, Equals, NotEquals, etc... using the raw SIMD instincts. But I abandoned due to a number of issues. First, I was reminded how painful writing SIMD code by hand is. Second, there needs to be an agreed baseline at compile time. The default for x86-64 is SSE2 (and bellow) and I would like to support  SSE2, AVX2, AVX512 and at some later point GPUs. I would like to support these without writing it many implementation.

#### JITing vectorized code

Confronted with that I took a step back and remember the body of academic research around JITing WHERE expression to avoid per row 2-level level conditional operation  (conditional about conditional). I decided to follow this thought further. The engine I'm building works by processing primarily in columnar fashion so I wouldn't be JITing the pre-row conditional filter, but if I could JIT vertical columar expressions to host CPU supported SIMD that would be it.

Enter LLVM. It's been used in numerous languages. It's used by the reference Rust compiler, it's been used by languages that feature JIT compilers (Javascript) and JIT languages with SIMD support like Julia. LLVM has been also used in other databases.

I have never used LLVM my self, but I have read about it's development over the years. I knew that other projects were using LLVM to compile to IR or Bitcode as a portability layer. In fact this is what the first OpenCL versions did (OpenCL is now transitioning to a different IR called [SPIR](https://www.khronos.org/news/press/khronos-releases-opencl-2.1-and-spir-v-1.0-specifications-for-heterogeneous)). I spent about a week learning about LLVM and it's IR, working & transforming IR. I thought to my self if can only generate the IR myself and have LLVM do a good enough job optimizing (vectorizing) for the host CPU I'd be in business.

### Writing LLVM IR is tedious

Turns out writing IR by hand that does something mildly interesting is pretty tedious task. So I cheated. I wrote very simple C code:

{% highlight C linenos %}
typedef unsigned char u8;

void bools_and(unsigned count, const u8 *rhs, const u8 *lhs, u8 *out) {
  for (unsigned off = 0; off < count; off++) {
    out[off] = rhs[off] & lhs[off];
  }
}
{% endhighlight %}

Then I used Clang to turn it into IR while targeting the SPIR arch. It turns out that unoptimized IR targeting SPIR is pretty generic and with some additional LLVM tools it can be mapped pretty well to OpenCL and a x86_64 host with lots of extensions.

So we can turn the above C code into IR like this:

{% highlight %}

{% endhightlight %} 

... And with a bit of right arguments to LLVM [opt](http://llvm.org/docs/CommandGuide/opt.html) you can have opt generate optimized IR that targets x86-64 and with more arguments you can have it spit out IR that when assembled compiles to pretty optimized host CPU code.

In the next part I'll cover getting LLVM working in Rust, JITing the generate IR and the battle to get it to generate decent code at runtime.

