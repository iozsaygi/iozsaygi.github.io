---
layout: post
title: "Caring about struct layouts and saving memory"
description: "Use of explicit struct memory layout management to improve memory usage."
tag: C#
---
Generally, when using structs, we are not sure if the size of the struct matches the size that we expect.

C# (probably other languages too) introduces a memory alignment practice called **"padding"**. Basically, the compiler will add extra bytes to the struct to align the data in memory. Well, we can prevent this by explicitly specifying the memory layout for our fields in struct.

Let's take a look at some examples.

### Read-only struct with compiler-aligned memory layout
Here's the good old read-only struct; we expect to get a total of ``4 bytes``, right?

Unfortunately, no. The compiler will add an extra 2 bytes to align memory, causing much more memory usage.
```csharp
public readonly struct Data  
{  
    public readonly byte ID; // Byte  
    public readonly short Progress; // 2 Bytes  
    public readonly byte Attachment; // Byte  
}
```

If we run the code below, we'll see that the output is actually 6 bytes. (Notice how we added the ``unsafe`` keyword to be able to use the ``sizeof`` operation.)
```csharp
internal abstract unsafe class Program  
{  
    public static void Main(string[] args)  
    {        
		Console.WriteLine($"Size of {nameof(Data)} is {sizeof(Data)}.");
    }
}
```
Let's fix this by explicitly specifying memory layouts in our struct.
