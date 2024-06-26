---
layout: post
title: Implementing quadtree data structure in Unity
description: How quadtrees can affect optimization by reducing the search space.
tags:
  - Unity
---
I always loved participating in game jams when I actually had time for them. For one of the Ludum Dare events (four or five years ago), I decided to apply as a compo participant, which meant I had three days to make a game from scratch by myself. No team-mates were allowed in the compo category, so it was quite a serious challenge.

To add to that challenge, I decided to not use a game engine and build a game by making my own engine in C++ using SDL. Was that a mistake? Of course not! The amount of things that I learned from that simple game was amazing. I learned how to structure entity component systems from the ground up, how scenes work, the basics of 2D rendering, and, of course, collision detection with axis-aligned bounding boxes.

My aim was to create a 2D top-down game where you have to run away from a horde of enemies—by horde, I mean thousands of enemies. Basically, if an enemy contacts you, the game is over.

The game initially had really low performance because I was checking the collisions with brute force; I was literally checking collisions against every enemy in the game (even if they were distant to the player or out of screen), and as you can imagine, frame rate suffered a lot.

At that time, I wasn't really aware that there was a specific data structure that could be really helpful in this case. But here we are now. I am about to teach a lesson to myself. Let's try to understand quadtrees and see how they can improve performance by decreasing our search space.

Oh, by the way, if you ever wonder about the game I am talking about, It was [Underlands](https://ldjam.com/events/ludum-dare/47/underlands), a game that initially aimed to have thousands of entities handle each frame but ended up being a classic 2D top-down shooter.

### What is a quadtree?
Quadtrees are tree data structures where each node of tree exactly has four children. Each node of a quadtree is called a **quadrant**, and each quadrant has its own bounds (spatial area) and capacity for those bounds. If data inside the quadrant's bounds exceeds its capacity, the quadrant will be divided into four different quadrants.

Assuming we are working on a 2D top-down environment, quadrants will have names as **north west**, **north east**, **south west**, and **south east**. Each representing different bounds inside a big quadrant.

See the image below. A quadtree has one root quadrant, and it has four different quadrants that divide the space into four equal areas. (The colorful dots are the entities available inside the quadtree.)
![Quadrants](https://raw.githubusercontent.com/iozsaygi/iozsaygi.github.io/main/assets/imgs/iaqiue/quadrants.png)

Here's a quick list to overview general properties of quadtrees:
* They divide space into four equal regions
* They can decrease the search/query space and optimize performance drastically
* Each quadrant will subdivide upon reaching the data capacity
* Whole data structure built with recursive operations

Now, let's see how we can implement quadtree in C#. Also, we will be using the Unity engine to visualize and debug our implementation.

### Definition of quadrants
Just like we talked earlier, quadrants are basically children of another quadrant, and they divide space into four equal regions. In this section, we'll talk about the general definition of quadrant class; we will not dive into actual implementations since they will be explained in different sections separately.

Let's see how we can define a quadrant with basic C# classes.
```csharp
public class Quadrant
{
    // The bounds of the quadrant, its volume in space.
    private readonly Bounds _bounds;

    // How many points can be stored inside this quadrant at maximum?
    private readonly byte _positionRegistryCapacity;

    // Positions that are inside of this quadrant's boundaries.
    private readonly List<Vector3> _positionRegistry; 

    // Reference to each child quadrant, they will be allocated during subdivision operation.
    private Quadrant _northWest;  
    private Quadrant _northEast;  
    private Quadrant _southWest;  
    private Quadrant _southEast;

    // Basic flag to see if the quadrant is already subdivided.
    private bool _isSubdivided;
}
```

Quadrants have some properties:
* The **bounds** property determines the occupation area of the quadrant
* **positionRegistryCapacity** determines how many points can quadrant store within its bounds
* **positionRegistry** is the list of points (world coordinates) that are inside within the bounds of this quadrant
* It also has four different references to possible child quadrants
* And, finally, the **isSubdivided** flag to see if a specific quadrant is already partitioned into smaller chunks also prevents the algorithm from endless recursive steps

After defining the base quadrant class, now we will talk about its logic. We will see how to implement `Construct`, `InsertPosition`, and `GetPositionsNearby` APIs.