---
layout: post
title: "Caring about struct layouts and saving memory"
description: "Use of explicit struct memory layout management to improve memory usage."
tag: C#
---
Generally, when using structs, we are not sure if the size of the struct matches the size that we expect.

C# (probably other languages too) introduces a memory alignment practice called "padding". Basically, the compiler will add extra bytes to the struct to align the data in memory. Well, we can prevent this by explicitly specifying the memory layout for our fields in struct.