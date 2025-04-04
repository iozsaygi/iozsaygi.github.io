---
layout: post
title: Disassembling your C# code to explore method inlining results
description: Using third-party tools to disassemble our C# code and explore the results of method inlining.
tags:
  - C#
---
If you spend enough time tackling optimization challenges, you’ll eventually come across the concept of method inlining. Over the past few weeks, I’ve had the chance to explore it in depth. Before diving deeper, let me briefly explain what it’s all about.

_Put simply, method inlining replaces a method call with the actual body of the method itself._

Why is this considered more performant than a regular function call? _Because it removes the overhead associated with calling a function — such as jumping to another memory location and managing the call stack._

That said, inlining isn’t something you can—or should—sprinkle throughout your codebase. In most cases, the compiler is actually better at deciding when and where inlining makes sense.

Here’s the catch: even though I attempted to manually inline some functions, I had no real way of knowing whether the compiler agreed with my efforts. I wanted to verify if my inlining choices were actually removing the `call` instructions in the generated assembly code. After some digging, I was fortunate enough to discover that [BenchmarkDotNet](https://benchmarkdotnet.org) can help uncover exactly that.

!!!TODO: When not to inline!!!