---
layout: post
title: Usage of the IComparable interface to implement comparison operations for our custom types
description: Small talk about the IComparable interface and how we can use it.
tags:
  - C#
---
Recently, I was learning more about [SortedSet](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.sortedset-1?view=net-8.0) collections in .NET, simply because I needed a way to sort existing data before doing some gameplay operations over them. Luckily, at the end of the day, I've found a way that helps me to avoid the sorting requirement.

But the things I've learned stayed with me. In this post, I want to talk about how we can use sorted sets with our custom data types.

First things first, let's take a quick look at the [IComparable](https://learn.microsoft.com/en-us/dotnet/api/system.icomparable?view=net-8.0) interface.