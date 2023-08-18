---
layout: post
title: "Caring about struct layouts and saving memory"
description: "Use of explicit struct memory layout management to improve memory usage."
tag: C#
---
Generally, when using structs, we are not sure if the size of the struct matches the size that we expect.

C# (probably other languages too) introduces a memory alignment practice called **"padding"**. Basically, the compiler will add extra bytes to the struct to align the data in memory. Well, we can prevent this by explicitly specifying the memory layout for our fields in the struct or caring about field declaration orders.

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
Let’s review two different cases to fix this issue.

### Read-only struct with ascending field declaration
Even our field declaration order matters when it comes to size. Notice how we declared fields in ascending order related to their sizes. When we try to check the size of this struct with the ``sizeof`` keyword, we will get an output of ``4 bytes``. Which is fascinating.
```csharp
public readonly struct DataWithAscendingOrder
{
    public readonly byte ID; // Byte
    public readonly byte Attachment; // Byte
    public readonly short Progress; // 2 Bytes
}
```

But what if we don't want to change the declaration order of our fields? ``StructLayout`` and ``FieldOffset`` attributes to the rescue!

### Read-only struct with explicit memory layout
Here's our explicitly aligned read-only struct. We still expect to get 4 bytes of size, and yes! Since we explicitly aligned the memory of our fields. The compiler will not add an extra 2 ghost bytes to our struct.
```csharp
[StructLayout(LayoutKind.Explicit, Size = 4)]
public readonly struct DataWithExplicitLayout
{  
    [FieldOffset(0)] public readonly byte ID; // Byte
    [FieldOffset(2)] public readonly short Progress; // 2 Bytes
    [FieldOffset(1)] public readonly byte Attachment; // Byte
}
```

Let's inspect the attributes we've used so far.

1. ``[StructLayout(LayoutKind.Explicit, Size = 4)]``
	
	With this option, we are telling the compiler that we will manage the memory layout of this struct. Preventing the compiler from adding extra ghost bytes to the struct.

2. ``[FieldOffset(0)], [FieldOffset(2)], [FieldOffset(1)]``
	
	By using this attribute, we are indicating the physical position of the fields in memory, which lets us have complete control over our structure's memory layout. In this case, _FieldOffset[2]_ and _FieldOffset[3]_ are reserved for our short field (which is 2 bytes) and FieldOffset[0] and FieldOffset[1] are reserved for our byte fields..

### Last words
Performance is crucial when it comes to game development, and this little trick can prove useful when we have to consider every single byte of memory during development. If we optimize our structure's memory layout, this also means that we can operate on even more data by effectively using our memory.

I hope you have enjoyed reading so far. Thank you.