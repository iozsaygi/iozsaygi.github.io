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

### IComparable interface
When working with native.NET types like int, byte, and double. C# already knows how to compare them, but things get tricky when you want to also compare your custom types.

Let's say you have a struct called Item, and items have some level of priority; the higher the priority of the item, the higher the value of it. Inside your inventory class, let's imagine you want to sort your items somehow; since it is your custom type, C# doesn't know how to sort it actually. At this point, you can implement the IComparable interface to your struct and define custom guidelines for C# about how to sort your struct instances.

Let's see the basic implementation of the IComparable interface.