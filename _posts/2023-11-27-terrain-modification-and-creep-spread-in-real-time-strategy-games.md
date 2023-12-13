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
To keep things simple for the sake of this blog post, we will be implementing the feature in a 3D isometric environment. Also, I am not planning to share any shader code since I am not that good with shader development. So I'm sorry for those who were looking for shader source code. We will mostly stick to gameplay logic. Also note that I will be using Unity ``2022.3.14f1`` for this blog post, so the API might be different based on the time you are reading this post.

I will be separating the implementation part into several topics to ease the reading process. You can find the specific topics below.

1. [Scene preparation](#scene-preparation)
2. [Node structure](#node-structure)
3. [Generating a node map](#generating-a-node-map)
4. [Building placement](#building-placement)
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
    // Assuming each of our nodes has (1.0f, 1.0f, 1.0f) size.  
    public static readonly Vector3 Size = Vector3.one;
    
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
    
    // The actual prefab that we will use to visualize the node map. Change it however you like.
    [SerializeField] private LineRenderer lineRendererPrefab;

    // The actual array that we'll be generating nodes on.
    private Nodes[] nodes;

    // Size of the generated node map.
    private Vector2 nodeMapSize;
}
```

We will use the vertices of the plane to divide it equally for creating nodes. Please refer to the official [Unity documentation](https://docs.unity3d.com/ScriptReference/MeshFilter-sharedMesh.html) for detailed information about mesh filters and shared meshes.

Now we will implement the actual node generation. This is a relatively big function, so please bear with me. I tried to add comments to each section of it.
```cs
private void Initialize()
{
    // Register corner points of mesh.
    var meshCornerPoints = new Vector3[3];
    var vertices = meshFilter.sharedMesh.vertices;

    // 'vertices[120]' is the origin for us. (Closest corner to the camera)
    meshCornerPoints[0] = meshFilter.transform.TransformPoint(vertices[120]);
    meshCornerPoints[1] = meshFilter.transform.TransformPoint(vertices[10]); 
    meshCornerPoints[2] = meshFilter.transform.TransformPoint(vertices[0]);
  
    // Calculate how many nodes will be available for each row and column.  
    var horizontalLength = Vector3.Distance(meshCornerPoints[0], meshCornerPoints[1]); 
    var verticalLength = Vector3.Distance(meshCornerPoints[1], meshCornerPoints[2]);
    nodes = new Node[(int)horizontalLength * (int)verticalLength];
    nodeMapSize = new Vector2(horizontalLength, verticalLength);

    // Define variables to handle generation in single for pass.
    var origin = meshCornerPoints[0] + originOffset;
    var iteration = 0;
    var depthOffset = 0; 

    for (byte i = 0; i < nodes.Length; i++)
    {        
        // Calculated position of the node.
        var placement = new Vector3(origin.x + iteration, 0.0f, origin.z + depthOffset);

        // Neighbor calculations for each node
        var rowLength = (int)horizontalLength;
        var forwardNeighborID = i + rowLength < nodes.Length ? new NodeID((byte)(i + rowLength)) : new NodeID(NodeID.InvalidNodeID);
        var backwardNeighborID = i - rowLength >= 0 ? new NodeID((byte)(i - rowLength)) : new NodeID(NodeID.InvalidNodeID);
        var rightNeighborID = (i + 1) % rowLength != 0 && i + 1 < nodes.Length ? new NodeID((byte)(i + 1)) : new NodeID(NodeID.InvalidNodeID);
        var leftNeighborID = i % rowLength != 0 ? new NodeID((byte)(i - 1)) : new NodeID(NodeID.InvalidNodeID);

        var neighbors = new NodeID[4];
        neighbors[0] = forwardNeighborID;
        neighbors[1] = backwardNeighborID;
        neighbors[2] = rightNeighborID;
        neighbors[3] = leftNeighborID;

        // Generate the actual node in array.
        nodes[i] = new Node(new NodeID(i), placement, neighbors);

        if (!(iteration >= horizontalLength)) continue;
        iteration++;
        iteration = 0;
        depthOffset += 1;  
    }
}
```

We can call this function at the ``Start`` or ``Awake`` callback of our MonoBehavior; it doesn't really matter for our case since we don't care about race conditions currently.

So it would also be so cool for us to see Node ID values in Gizmos mode. Let's go ahead and also implement the Gizmos function to render ID values for each node we created.
```cs
#if UNITY_EDITOR  
private void OnDrawGizmosSelected()  
{        
    if (nodes == null) return;  
  
    var guiStyle = new GUIStyle();  
    for (var i = 0; i < nodes.Length; i++)  
    {            
        var renderPositionWithOffset = nodes[i].Position + new Vector3(0.0f, 0.5f, 0.0f);  
        guiStyle.normal.textColor = Color.magenta;  
        Handles.Label(renderPositionWithOffset, nodes[i].NodeID.Value.ToString(), guiStyle);  
    }    
}
#endif // UNITY_EDITOR
```

One thing to mention is that you need to include ``UnityEditor`` to be able to render labels with the [Handles](https://docs.unity3d.com/ScriptReference/Handles.html) API.

After implementing the Gizmos rendering, our scene should look something like this:
![Node IDs with Gizmos](https://github.com/iozsaygi/iozsaygi.github.io/blob/main/assets/imgs/tmacsirtsg/node_ids_with_gizmos.png?raw=true)

Before getting into line renderers, we need to convert our world positions to nodes. To do this, we'll be adding a very simple function to the node map class. Basically, the function creates bounds and checks if a given world point is inside those bounds. It also returns a node that matches the given world position. See the function below.
```cs
public void FetchNodeFromWorldPoint(Vector3 worldPoint, out Node node)
{
    node = default;

    for (byte i = 0; i < nodes.Length; i++)
    { 
        var bounds = new Bounds(nodes[i].Position, Node.Size);
        if (!bounds.Contains(worldPoint)) continue;
        node = nodes[i];
        break;
    }
}
```

Well, maybe this is not the best way to achieve this. One way to optimize this would be to store nodes in a dictionary by using their IDs as keys and positions as values.

Now let's also add line renderers to visually divide our plane into groups of nodes.
```csharp
private void CreateLineRenderers()  
{  
    // Origin point that line renderers will be generated from.  
    var origin = new Vector3(-5.0f, 0.0f, -5.0f);  
    
    // Variables to manipulate actual positions, they can also be exposed/serialized to the inspector.
    const float verticalOffset = 0.025f;  
    const byte lineLength = 10;  
    const byte spaceBetweenLines = 1;  
    
    // Vertical line creation.  
    for (var i = 0; i < nodeMapSize.y + 1; i++)  
    {        
        var horizontalLineRendererGameObject = Instantiate(lineRendererPrefab, transform.position, Quaternion.identity);  
        horizontalLineRendererGameObject.transform.SetParent(transform, false);  
        var lineRenderer = horizontalLineRendererGameObject.GetComponent<LineRenderer>();  
        lineRenderer.positionCount = 2;  
        lineRenderer.SetPosition(0, new Vector3(origin.x, origin.y + verticalOffset, origin.z + (i * spaceBetweenLines)));  
        lineRenderer.SetPosition(1, new Vector3(origin.x + lineLength, origin.y + verticalOffset, origin.z + (i * spaceBetweenLines)));  
    } 
     
    // Horizontal line creation.  
    for (var i = 0; i < nodeMapSize.x + 1; i++)  
    {        
        var verticalLineRendererGameObject = Instantiate(lineRendererPrefab, transform.position, Quaternion.identity);  
        verticalLineRendererGameObject.transform.SetParent(transform, false);  
        var lineRenderer = verticalLineRendererGameObject.GetComponent<LineRenderer>();  
        var firstPosition = new Vector3(origin.x + (i * spaceBetweenLines), origin.y + verticalOffset, -origin.z);  
        var secondPosition = new Vector3(origin.x + (i * spaceBetweenLines), origin.y + verticalOffset, origin.z);  
        lineRenderer.SetPosition(0, firstPosition);  
        lineRenderer.SetPosition(1, secondPosition);  
    }
}
```

This is how it looks after adding line renderers; now we are able to visually separate nodes from each other at runtime.
![Line Renderers](https://github.com/iozsaygi/iozsaygi.github.io/blob/main/assets/imgs/tmacsirtsg/line_renderers.png?raw=true)

Now that we have node generation and a way to visualize it at runtime, let's take a look at how we can place buildings on nodes by using raycasting.

### Building placement
We will be creating a controller that uses raycasting to place buildings on top of nodes that we just created. We will not handle cases like not placing buildings on occupied nodes to keep this blog post simple.
```cs
using UnityEngine;  
  
public class IsometricController : MonoBehaviour  
{  
    // Reference of camera that we will be using.  
    [SerializeField] private Camera mainCamera;  
  
    // Length of the ray.  
    [SerializeField] private float rayDistance;  
  
    // The layer that we want to cast rays against.  
    [SerializeField] private LayerMask targetLayer;  
  
    // Reference of node map class to calculate node positions.  
    [SerializeField] private NodeMap nodeMap;  
  
    // The actual building prefab that will be placed by raycasting.  
    [SerializeField] private GameObject buildingPrefab;  
  
    private void Update()  
    {        
    // Wait for 'LMB' to trigger.  
        if (!Input.GetMouseButtonDown(0)) return;  
  
        // Construct the actual ray and perform a single raycast.  
        var ray = mainCamera.ScreenPointToRay(Input.mousePosition);  
        if (!Physics.Raycast(ray, out var raycastHit, rayDistance, targetLayer)) return;  
  
        // Convert world position to actual node.  
        var interactionPoint = raycastHit.point;  
        nodeMap.FetchNodeFromWorldPoint(interactionPoint, out var node);  
  
        // Spawn building at the node we just converted.  
        Instantiate(buildingPrefab, node.Position + new Vector3(0.0f, 0.5f, 0.0f), Quaternion.identity);  
    }
}
```

I tried my best to explain the logic with comments, but if I need to create a summary, we are waiting for the left mouse button (at least for PC) to be triggered and then casting rays from the camera into our world. Then we are detecting which node we are interacting with by using the ``FetchNodeFromWorldPoint`` function that we added to the node map class. Finally, we are spawning the actual building object on the node.

Now that we are able to place buildings with mouse clicks, we can finally start to work on the actual gameplay implementation of creep spread.