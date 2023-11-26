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