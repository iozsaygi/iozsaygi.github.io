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
* They divide space into four equal regions or rectangles.
* They can decrease the search/query space and optimize performance drastically.
* Each quadrant will subdivide upon reaching the data capacity.
* Whole data structure built with recursive operations.

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
* The **bounds** property determines the occupation area of the quadrant.
* **positionRegistryCapacity** determines how many points can quadrant store within its bounds.
* **positionRegistry** is the list of points (world coordinates) that are inside within the bounds of this quadrant.
* It also has four different references to possible child quadrants.
* And, finally, the **isSubdivided** flag to see if a specific quadrant is already partitioned into smaller chunks also prevents the algorithm from endless recursive steps.

After defining the base quadrant class, now we will talk about its logic. We will see how to implement `InsertPosition`, `GetPositionsNearby`, and `Subdivide` APIs.

### Subdividing a quadrant
We haven't implemented the `InsertPosition(Vector3);` API yet, but before doing that, we need to know how we can subdivide a quadrant into four equal pieces. We will divide the quadrants when a single piece reaches its maximum capacity for position registry.

Let's see how we can subdivide a quadrant into four equal pieces:
```csharp
private void Subdivide()
{
    // Calculate the size for each child quadrant. 
    var originalCenter = _bounds.center;
    var originalSize = _bounds.size;
    var newSize = new Vector3(originalSize.x / 2, originalSize.y / 2, originalSize.z);

    // Calculate half sizes for easier calculations.
    var halfHorizontalSize = newSize.x / 2;
    var halfVerticalSize = newSize.y / 2;
  
    // Calculate centers for the four child quadrants.  
    var northWestCenter = new Vector3(originalCenter.x - halfHorizontalSize,  
        originalCenter.y + halfVerticalSize, originalCenter.z);
    var northEastCenter = new Vector3(originalCenter.x + halfHorizontalSize,  
        originalCenter.y + halfVerticalSize, originalCenter.z);
    var southWestCenter = new Vector3(originalCenter.x - halfHorizontalSize,  
        originalCenter.y - halfVerticalSize, originalCenter.z);
    var southEastCenter = new Vector3(originalCenter.x + halfHorizontalSize,  
        originalCenter.y - halfVerticalSize, originalCenter.z);

    // Create bounds for each child quadrant.
    var northWestBounds = new Bounds(northWestCenter, newSize);
    var northEastBounds = new Bounds(northEastCenter, newSize);
    var southWestBounds = new Bounds(southWestCenter, newSize);
    var southEastBounds = new Bounds(southEastCenter, newSize);
  
    // Allocate every single child quadrant reference.
    _northWest = new Quadrant(northWestBounds, _positionRegistryCapacity);
    _northEast = new Quadrant(northEastBounds, _positionRegistryCapacity);
    _southWest = new Quadrant(southWestBounds, _positionRegistryCapacity);
    _southEast = new Quadrant(southEastBounds, _positionRegistryCapacity);

    _isSubdivided = true;
}
```

Here, we are basically calculating the new rectangular area of each child quadrant (north west, north east, south west, and south east) and then allocating each child quadrant with the calculated areas. There is no magic going on here, just simple math.

However, do not forget that I am working on a 2D top-down environment to implement quadtrees, so subdividing calculations might be different for the space that you are working on.

Now let's get to the actual part: inserting a new position into the quadtree.

### Inserting a position into the quadtree
This one is fairly simple; it will work as a recursive API, and of course, just like other recursive logic, we must have a failure condition to not recursively call this into exceptions.

Before taking a look at the code, let's explain the steps that we need to perform:
1. Check if the passed position parameter is inside this quadrant's bounds; don't do anything if this evaluates to `false`.
2. Now that we are sure that the given position is inside the bounds, check if the quadrant reached its capacity for the position registry.
3. If the quadrant has not reached its capacity yet, then just add the passed position to the position registry.
4. If the quadrant has reached its capacity, then subdivide it into four equal quadrants and try to add the given position to each child quadrant.

With these steps, we are ensuring that the given position is fed into the correct quadrant and that the required subdividing operations are performed as well.

Now, let's take a look at the code. The function itself is pretty short, but it works recursively.
```csharp
public void InsertPosition(Vector3 position)
{
    // Check if given position is inside the bounds of the quadrant.
    if (!_bounds.Contains(position)) return;

    // Check if we have enough space in registry to save/add given position.
    if (_positionRegistry.Count < _positionRegistryCapacity)
    {  
	_positionRegistry.Add(position);
    }
    else
    {
        // We don't have enough space in registry, it is time to subdivide the quadrant.
        // But we need to check if it is subdivided before.
        if (!_isSubdivided) Subdivide();  

        // Try to add given position to each quadrant.
        _northWest.InsertPosition(position);
        _northEast.InsertPosition(position);
        _southWest.InsertPosition(position);
        _southEast.InsertPosition(position);
    }
}
```

Well, this was the core API for a quadtree, but we have one more thing to implement, and that is way more important than this. Next, we'll figure out how we can query the positions in a quadrant just by passing the origin or reference position to it.

### Implementing an API to query nearby positions
Well, let's dive into the most important part of quadtree: position querying. We implemented all of this functionality so we can query less data whenever needed, resulting in a decreased search space and improved performance.

The signature of the API is pretty simple: `public IReadOnlyList<Vector3> GetPositionsNearby(Vector3 origin);`

The `origin` parameter is basically the position in world coordinates. We will try to find a related quadrant based on it and eventually return all of the positions inside that quadrant.

Let's see the actual implementation, and then we will review it step by step.
```csharp
public IReadOnlyList<Vector3> GetPositionsNearby(Vector3 origin)
{
    // Check if given 'origin' is inside the bounds of current quadrant.
    if (!_bounds.Contains(origin)) return new List<Vector3>().AsReadOnly();

    // Check if current quadrant subdivided before.
    if (!_isSubdivided) return _positionRegistry.AsReadOnly();

    // The quadrant is subdivided, query each child quadrant it has.
    var nearbyPositions = new List<Vector3>();

    nearbyPositions.AddRange(_northWest.GetPositionsNearby(origin));
    nearbyPositions.AddRange(_northEast.GetPositionsNearby(origin));
    nearbyPositions.AddRange(_southWest.GetPositionsNearby(origin));
    nearbyPositions.AddRange(_southEast.GetPositionsNearby(origin)); 

    return nearbyPositions.AsReadOnly();
}
```

Just like I promised, let's run it down step by step:
1. First, we have to check if the given `origin` position is inside the bounds. This is also a termination condition for the recursive calls, making them safer to use.
2. Then we are checking if the current quadrant is subdivided; if not, we will just return positions inside the current quadrant.
3. Now this is the recursive part; if the current quadrant is subdivided, we are eventually visiting each child quadrant and then continuing the previous steps until we reach a quadrant that meets the conditions. While doing that, we are also keeping track of the positions we find.
4. Eventually, after recursive calls, we are just returning the total read-only list of positions that we calculated.

Yep, so with this API, we are actually able to decrease the search space that we are working on, hopefully leading to better performance.

### When to use quadtrees
I would like to create a compilation list that I think is a good reason to implement quadtrees into your game development workflow:
- When you want to handle collisions for thousands of entities.
- When you want to implement AoE spells but have lots of entities in the scene.
- When you want to stream the assets of the environment in your open-world game.
- When you want to decrease the amount of data you are working with, So your operations will work on smaller input sizes.

### Conclusion
What a year! I think we just celebrated the first year of my blog with quadtrees! I can't even believe that it has been a year since I started blogging. Time just flies away.

I strongly think that it was a great entry for the anniversary blog post. Hopefully, it will give some ideas to someone out there and ease the workflow for my fellow developers.

Oh! I almost forgot. You can view my [repository](https://github.com/iozsaygi/unity-quadtree) for the full quadtree implementation, fork it, and try to make your own adjustments if you want. I am sure there is room for improvement.

Thank you so much for sticking with me until the end. Let's see which type of blog posts await me in the next few years!