---
layout: post
title: Why do I love read-only structures?
description: My personal opinions on read-only structures in C#.
tags:
  - C#
---
Over the past one or two years, there has been a specific C# feature that I have been really obsessed with. I pretty much use it anywhere that I can.

Yep, that golden feature is read-only structs.

First, let's summarize general reasons to use them, and then take a detailed look at each reason with its own part.

1. Immutability and thread safety
2. Impacts of stack allocations
3. Possible optimization opportunities for the compiler

### Immutability and thread safety