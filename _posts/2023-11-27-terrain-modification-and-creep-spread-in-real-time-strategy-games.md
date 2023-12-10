---
layout: post
title: Terrain modification and creep spread in real-time strategy games
description: Reviewing the creep spread mechanism that is mostly used in real-time strategy games
tags:
  - Unity
---
I think the first strategy game I've ever played was [Age of Empires II: The Conquerors](https://en.wikipedia.org/wiki/Age_of_Empires_II:_The_Conquerors). Being able to experience battles in a historical simulator blew my mind. History was an interesting topic for me when I was a child, so there is no doubt I really enjoyed playing strategy games back in the day, I guess.

RTS games really evolved over the years; new games were introduced almost every year, and each good game improved the genre by adding big features like supporting thousands of units in the scene, new graphical enhancements to the overall environment, and active support for the competitive scene.

I've never been a competitive RTS player, but I was always a fan of the genre. If I need to list my top five RTS games, the list will probably look like this:

1. [Warcraft III: The Frozen Throne](https://en.wikipedia.org/wiki/Warcraft_III:_The_Frozen_Throne)
2. [The Lord of the Rings: War of the Ring](https://en.wikipedia.org/wiki/The_Lord_of_the_Rings:_War_of_the_Ring)
3. [Age of Empires II: The Conquerors](https://en.wikipedia.org/wiki/Age_of_Empires_II:_The_Conquerors)
4. [StarCraft II](https://en.wikipedia.org/wiki/StarCraft_II)
5. [Empire Earth](https://en.wikipedia.org/wiki/Empire_Earth)

Some of these games introduced a really cool feature called creep spread or corruption. Basically, specific races had the ability to corrupt the land to gain an advantage during battle or improve vision through map, which would be crucial to dominating the opponent during the match.

In this post, I will try my best to explain the implementation of this technique, but first, let's try to understand what corruption or creep spread is in RTS games.

### Understanding the creep mechanic
When we talk about the [Undead](https://liquipedia.net/warcraft/Undead) race from Warcraft III, we'll see that they are able to corrupt the land by constructing new buildings on the terrain.

Let's take a look at the screenshot below from Warcraft III.
![Warcraft III Undead Blight](https://github.com/iozsaygi/iozsaygi.github.io/blob/main/assets/imgs/tmacsirtsg/wc3_undead_blight.png?raw=true)

The corrupted terrain has a special texture that represents a special area for the Undead race. Undead units will gain additional positive effects (buffs) if they are on corrupted terrain.

Similar to Warcraft III, StarCraft also has the very same feature for the [Zerg](https://en.wikipedia.org/wiki/Zerg) race.
![StarCraft II Zerg Creep](https://github.com/iozsaygi/iozsaygi.github.io/blob/main/assets/imgs/tmacsirtsg/sc2_zerg_creep.png?raw=true)

Also, in most of the RTS games, creep can block opponents from constructing new buildings on terrain. For example, in StarCraft, [Terran](https://starcraft.fandom.com/wiki/Terran) players will not be able to construct a building on top of a terrain that is corrupted by Zerg players' creep.

Now let's get right into Unity and see how we can implement this feature for our own RTS game!

### Overview
To keep things simple for the sake of this blog post, we will be implementing the feature in a 3D isometric environment. Also, I am not planning to share any shader code since I am not that good with shader development. So I'm sorry for those who were looking for shader source code. We will mostly stick to gameplay logic.

I will be separating the implementation part into several topics to ease the reading process. You can find the specific topics below.

1. [Scene preparation](#scene-preparation)
2. [Node structure](#node-structure)
3. [Generating a node map](#generating-a-node-map)
4. Building placement
5. Implementing creep logic

### Scene preparation
A simple isometric camera with ``orthographic`` projection is pretty enough. I also added a plane object at ``(0.0f, 0.0f, 0.0f)`` to make it act as our ground.

Here's my current game view; simple as that.
![Scene Preparation](https://github.com/iozsaygi/iozsaygi.github.io/blob/main/assets/imgs/tmacsirtsg/scene_preparation.png?raw=true)

One thing to mention is that I will be using ``800x600`` as my reference resolution in game view to ensure consistency between screenshot sizes.

### Node structure
Nodes are the core building blocks of RTS games. They are used for pathfinding, map data generation, building placements, and many more. 

We will be representing nodes with a really simple read-only struct that contains information about the node's ID, position, and neighbors.

The first structure is NodeID, which just contains a unique byte value to represent our node's number in the node map. Nothing fancy.
```cs
public readonly struct NodeID
{
	// The value that we will be using to represent invalid nodes.  
	public const byte InvalidNodeID = 255;

    // We can get away with 'byte' here because we know we will not have that many nodes.
    public readonly byte Value;

    public NodeID(byte value)
    {
	    Value = value;
    }
}
```

The second structure is the node's itself. This one contains its ID value, the position of itself, and the ID values of its neighbors.
```cs
using UnityEngine;

public readonly struct Node
{
    public readonly NodeID NodeID;
    public readonly Vector3 Position;
    public readonly NodeID[] Neighbors;

    public Node(NodeID nodeID, Vector3 position, NodeID[] neighbors)  
    {
	NodeID = nodeID;
        Position = position;
        Neighbors = neighbors;
    }
}
```

Alright, these two structs should be enough to implement the rest of the logic. We first thought about the data we had and tried to represent it with a model. Let's jump into another fun part: generating node maps.

### Generating a node map
At first, we'll be adding the required serialized fields to our MonoBehaviour. ``MeshFilter`` will basically refer to the ground object that we created earlier, and the origin offset will be ``(0.5f, 0.0f, 0.5f)`` for the sake of this blog post.
```cs
using UnityEngine;

public class NodeMap : MonoBehaviour
{
    [SerializeField] private MeshFilter meshFilter;
    [SerializeField] private Vector3 originOffset;
}
```

We will use the vertices of the plane to divide it equally for creating nodes. Please refer to the official [Unity documentation](https://docs.unity3d.com/ScriptReference/MeshFilter-sharedMesh.html) for detailed information about mesh filters and shared meshes.
