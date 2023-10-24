---
layout: post
title: Minimal wrapper around addressables
description: Minimal C# addressable wrapper that I implemented for one of my prototypes.
tags:
  - Unity
---
During the development of some game prototypes, I often found myself tackling a lot with [Addressables](https://docs.unity3d.com/Manual/com.unity.addressables.html) in Unity. I wanted to create a queue-based initialization system for the gameplay subsystems.

But what exactly is a subsystem? In my case, a subsystem is basically a set of codes and prefabs that are part of a bigger subsystem or system.

Let's actually start by inspecting the base subsystem class and an example child object of it.

### Base Subsystem class and example usage
Here's the minimal definition of a base subsystem class that I used: it only has an abstraction for the initialization callback, which is called from a class that actually handles subsystem prefabs as addressables. (We'll take a look at that class too in the later parts of this blog.)
```cs
using UnityEngine;

namespace AAA.Source.Framework.Subsystem.Runtime
{
    [DisallowMultipleComponent]
    public abstract class BaseSubsystem : MonoBehaviour
    {
        public abstract void OnSceneInitialize();
    }
}
```
Now here is a basic example of a gameplay subsystem that is a child of the ``BaseSubsystem`` class and implements the ``OnSceneInitialize`` method to create prefabs that it requires to function.
```cs
using AAA.Source.Framework.Subsystem.Runtime;
using UnityEngine;

namespace AAA.Source.Gameplay.Debugger.Runtime
{
    public class GameplayDebuggerSubsystem : BaseSubsystem
    {
        [SerializeField] private GameObject gameplayDebuggerCanvasPrefab;
        
        public override void OnSceneInitialize()
        {            Instantiate(gameplayDebuggerCanvasPrefab, Vector3.zero, Quaternion.identity).transform
                .SetParent(transform, true);
        }    
    }
}
```