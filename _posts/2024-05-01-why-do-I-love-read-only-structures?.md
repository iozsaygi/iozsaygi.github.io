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
So a lot of times, when writing multithreaded code, we often worry about race conditions and data immutability. Take the following struct definition as an example; it is pretty much open to being modified from anywhere, everywhere.
```csharp
public struct NotSoSafe
{
    public int Data;
}
```

So if you are using this struct with a collection, you can eventually [lock](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/statements/lock) it and try to prevent it from being modified by other threads while you are working on that data in your specific thread. Which might work, but still doesn't achieve full immutability.

So consider following usage of the same struct, which is full readonly and locks itself from changing (after allocation) from other parts of code.
```csharp
public readonly struct Safe
{
    public readonly int Data;

    public Safe(int data)
    {
	Data = data;
    }
}
```

This is much safer to iterate on since you know it is not going to get modified by other parts of the code or threads. Of course, it can still be changed by reallocation, but it is a totally different case to look at.

However, there's a case where it can be hard to use readonly structs, and that case is serialization. I heard there are .NET libraries that can serialize or deserialize read-only structures, but I never gave them a try before.

### Impacts of stack allocations
So before I dive into this, there is one crucial thing that I want to mention. Structs will be allocated on the stack, but at some point, if you declare a struct instance **inside of a class**, it will be allocated on the heap with that class instance.

Take a look at the following usage example:
```csharp
// Created a basic struct that holds data as 'byte' type.  
public struct DataPackage
{
    public byte Data;

    public DataPackage(byte data)
    {
        Data = data;
    }
}

// Class that operates on the data package instance.
public class DataController
{
    public DataPackage DataPackage;
}
```

In the usage case above, both struct and class will be allocated on the heap since struct instance is declared as a member of heap allocated class.

However, I think one of the best advantages of the structure is definitely the [data locality](https://gameprogrammingpatterns.com/data-locality.html) it introduces. Consider a case where you have to work with large datasets, maybe an array of thousands of elements. Iterating over an **array of class instances** is much **slower** than iterating over an **array of struct instances**. (I wish I had some benchmark data to show, but lessons are learned for future blog posts.)

With the array that holds a class instance, you are pretty much chasing a pointer to that class instance that is located randomly around the RAM. It will definitely cause wasted cycles while the code is trying to receive that pointer from its random location. But in the case where you are storing instances of structs in the array, your logic will be able to run faster than its class version because all of the data it needs is allocated on the stack, and the CPU can actually reach it faster from its cache.

### Possible optimization opportunities for the compiler

### Resources
* [lock](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/statements/lock)
* [Data Locality](https://gameprogrammingpatterns.com/data-locality.html)
* [Better CPU Caching with Structs](https://www.jacksondunstan.com/articles/3399)