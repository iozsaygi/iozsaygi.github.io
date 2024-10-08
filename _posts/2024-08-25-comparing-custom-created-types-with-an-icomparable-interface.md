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
When working with native .NET types like int, byte, and double. C# already knows how to compare them, but things get tricky when you want to also compare your custom types.

Let's say you have a struct called Item, and items have some level of priority; the higher the priority of the item, the higher the value of it. Inside your inventory class, let's imagine you want to sort your items somehow; since it is your custom type, C# doesn't know how to sort it actually. At this point, you can implement the IComparable interface to your struct and define custom guidelines for C# about how to sort your struct instances.

Let's see the basic implementation of the IComparable interface.
```cs
public readonly struct Item : IComparable<Item>
{
    public int CompareTo(Item other)
    {
	throw new NotImplementedException();
    }
}
```

Just forces us to implement a new method called `CompareTo`. This is where we are going to define our sorting logic for C#.

### Defining custom sorting guidelines
We talked about item priorities; let's populate our struct with dummy data, including the priority. Then we'll be using that priority to define custom sorting guidelines for C#.
```cs
public readonly struct Item : IComparable<Item>
{
    // Dummy ID value that will be representing the unique number for item in database.
    public readonly byte DatabaseID;

    // The priority of the item, will be used for sorting guideline.
    public readonly byte Priority;

    // Dummy data to represent item's price at the vendor.
    public readonly double VendorPrice;

    public Item(byte databaseID, byte priority, double vendorPrice)
    {
	DatabaseID = databaseID;  
	Priority = priority;  
	VendorPrice = vendorPrice;  
    }

    public int CompareTo(Item other)
    {
	throw new NotImplementedException();
    }
}
```

Now let's actually implement the `CompareTo` method; it is pretty simple:
```cs
public int CompareTo(Item other)
{
    return Priority.CompareTo(other.Priority);
}
```

Since we will be using priority as a way to compare our items, this is pretty easy. We are just creating a wrapper around byte's IComparable implementation.

### Using sorted sets with our custom type
I am not really a fan of sorted sets, but let's use one to automatically sort our custom type based on the item priorities.
```cs
public class Inventory
{  
    // Items in the inventory. (Sorted)  
    private readonly SortedSet<Item> _items = new()
    {        
        new Item(0, 3, 50),  
        new Item(1, 1, 100),  
        new Item(2, 7, 150),  
        new Item(3, 4, 200),  
        new Item(4, 2, 250)  
    };

    public void Print()
    {
        foreach (var item in _items)
        {
            Debug.Log($"Item information:\n Database ID: {item.DatabaseID}\n" + $"Priority: {item.Priority}\n" +  $"Vendor Price: {item.VendorPrice}");
        }
    }
}
```

Basic inventory class defines the items with a custom order, not respecting the priorities. Also has print function to ease debugging process.

Even when we declare items without any sorted order when we call the print function, the result is pretty much sorted based on item priorities. Here's the result of the print function with the current item set:
```
Item information: Database ID: 1 Priority: 1 Vendor Price: 100
Item information: Database ID: 4 Priority: 2 Vendor Price: 250
Item information: Database ID: 0 Priority: 3 Vendor Price: 50
Item information: Database ID: 3 Priority: 4 Vendor Price: 200
Item information: Database ID: 2 Priority: 7 Vendor Price: 150
```

It is totally up to us to define how C# is going to sort our types; we can combine several properties and implement even more complicated sorting guidelines.

### Conclusion
I wrote this article while I was chasing a gameplay bug for so long (5-6 hours), and I kind of wanted to change my area of focus with some kind of blog post. Also, the IComparable interface was something that I really wanted to dive deep into, and I think this post made a cool entry for it.

I will definitely use this interface more but not really with sorted sets. I feel like sorted sets are the perfect way to waste CPU cycles since they are most likely doing item shifting under the hood very often. So I think it is cool to know about this interface, but sorted set usage is something to be carefull on.

Thank you for reading, and see you next time!