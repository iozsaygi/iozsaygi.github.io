---
layout: post
title: Workaround for using layer masks in the custom editor window
description: A workaround that I explored for using layer mask fields inside custom editor windows.
tags:
  - Unity
---
When creating an editor tool that heavily relies on raycasting, I wanted to implement raycasting for specific objects in specific layers for obvious reasons.

But somehow, the raycasts were not working at all; it was all set and correct in the editor window fields, but something was missing. After a few hours of hair-pulling, I finally explored a workaround that actually lets programmers use layer masks inside the custom editor window.

But keep in mind that the code that I am about to reveal is written with Unity version ``2022.3.13f1`` (which is the latest LTS version available at the time that I am writing this blog), so the API might differ based on your Unity version.

Let's start by inspecting very simple editor code that tries to cast a ray and create a primitive game object at the ray's intersection point.

### The first iteration of the editor window
Here's the initial code that I wrote: very simple raycasting and creating a primitive game object at the intersection point. I also want to mention that this code is always running in the editor; it doesn't even matter if the game is actually running or not.
```cs
using UnityEngine;
using UnityEditor;

public class LayerMaskInsideEditorWindow : EditorWindow  
{  
    private float rayLength;  
    private LayerMask targetLayer;  
  
    [MenuItem("Unity Playground/Layer Mask Inside Editor Window")]  
    private static void Display() => GetWindow<LayerMaskInsideEditorWindow>();  
  
    private void OnGUI()  
    {        
	rayLength = EditorGUILayout.FloatField("Ray Length", rayLength);  
        targetLayer = EditorGUILayout.LayerField("Target Layer", targetLayer);  
  
        if (GUILayout.Button("Execute"))  
        {            
	    var ray = new Ray(new Vector3(0.0f, 50.0f, 0.0f), Vector3.down);  
            Physics.Raycast(ray, out var raycastHit, rayLength, targetLayer);  
  
            if (raycastHit.collider == null) return;  
            GameObject.CreatePrimitive(PrimitiveType.Cube).transform.position = raycastHit.point;  
        }    
    }
}
```
No matter what I tried, raycasting was not even working. So I decided to take a look at the amazing world of the internet and see how other people actually fixed this issue. After jumping around discussion threads a bit, I found an answer that uses the ``UnityEditorInternal`` namespace.

Of course, Unity is not providing this API for us to use; as its name suggests, it is for Unity's internal API only. So we are definitely taking a risk here because we don't know when this API is actually going to change. Oh, but on the bright side, we learned a new way to handle this specific feature.

Let's take a look at how we can utilize this namespace to handle layer masks in our custom editor windows.

### Utilizing UnityEditorInternal namespace
The first change we are going to introduce is temporary layer mask caching during the OnGUI function.
```cs
var temporaryLayerMask = EditorGUILayout.MaskField(  
    InternalEditorUtility.LayerMaskToConcatenatedLayersMask(targetLayer), InternalEditorUtility.layers);
```
Then we'll set the field of the editor window to a temporary layer mask we cached by using internal API of Unity.
```cs
targetLayer = InternalEditorUtility.ConcatenatedLayersMaskToLayerMask(temporaryLayerMask);
```
So the latest state of our editor window class will be something like this:
```cs
using UnityEditor;  
using UnityEditorInternal;  
using UnityEngine;  

public class LayerMaskInsideEditorWindow : EditorWindow  
{  
    private float rayLength;  
    private LayerMask targetLayer;  
  
    [MenuItem("Unity Playground/Layer Mask Inside Editor Window")]  
    private static void Display() => GetWindow<LayerMaskInsideEditorWindow>();  
  
    private void OnGUI()  
    {        
	rayLength = EditorGUILayout.FloatField("Ray Length", rayLength);  
  
        var temporaryLayerMask = EditorGUILayout.MaskField(  
            InternalEditorUtility.LayerMaskToConcatenatedLayersMask(targetLayer), InternalEditorUtility.layers);  
        targetLayer = InternalEditorUtility.ConcatenatedLayersMaskToLayerMask(temporaryLayerMask);  
  
        if (GUILayout.Button("Execute"))  
        {            
        var ray = new Ray(new Vector3(0.0f, 50.0f, 0.0f), Vector3.down);  
            Physics.Raycast(ray, out var raycastHit, rayLength, targetLayer);  
  
            if (raycastHit.collider == null) return;  
            GameObject.CreatePrimitive(PrimitiveType.Cube).transform.position = raycastHit.point;  
        }    }}
```


I am still trying to figure out why and how we need to use this type of workaround to make layer masks actually work in custom editor windows.