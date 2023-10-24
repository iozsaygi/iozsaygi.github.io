---
layout: post
title: Minimal wrapper around addressables to implement subsystem hierarchy
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
        {            
            Instantiate(gameplayDebuggerCanvasPrefab, Vector3.zero, Quaternion.identity).transform.SetParent(transform, true);
        } 
    }
}
```
``GameplayDebuggerSubsystem`` basically contains a reference to a prefab(s) that it needs to function and creates its required prefabs during the initialization of the scene. It is the most basic subsystem that I can imagine right now; nothing fancy, just creating new game objects.

After setting up the initial code for the ``GameplayDebuggerSubsystem``, I created a prefab of it and marked it as addressable so we could actually load it into memory and start using it at runtime whenever we needed.

Now let's take a look at another minimal class that is actually placed in the scene hierarchy manually. It is just holding reference to the subsystem prefabs that are marked as addressables and the subsystem streaming controller class.

### Controller class to manage subsystems
Minimal subsystem manager that actually contains references to the addressable subsystem prefabs.
```cs
using UnityEngine;
using UnityEngine.AddressableAssets;

namespace AAA.Source.Framework.Subsystem.Runtime
{  
    [DisallowMultipleComponent]  
    public class SubsystemManagementController : MonoBehaviour  
    {
        [SerializeField] private AssetReferenceGameObject[] registeredSubsystemAddressables;  
        [SerializeField] private SubsystemStreamingController subsystemStreamingController;  
  
        private async void Start()  
        {            
            subsystemStreamingController = new SubsystemStreamingController();  
            foreach (var registeredSubsystemAssetReference in registeredSubsystemAddressables)  
                subsystemStreamingController.RequestAssetForInitialization(registeredSubsystemAssetReference);  
  
            await subsystemStreamingController.LoadQueuedSubsystemsAsync();  
            subsystemStreamingController.InstantiateLoadedSubsystems(transform);  
            subsystemStreamingController.InitializeInstantiatedSubsystems();  
        }    
    }
}
```
Let's highlight specific parts of it for further explanation.


```cs
[SerializeField] private AssetReferenceGameObject[] registeredSubsystemAddressables;
```
This is the actual array that holds references to the addressable subsystem prefabs that will be loaded into memory by the ``SubsystemStreamingController`` class.


```cs
subsystemStreamingController = new SubsystemStreamingController();  
            foreach (var registeredSubsystemAssetReference in registeredSubsystemAddressables)
```
Here, we are creating a new instance of the subsystem streaming controller and requesting load operation for each addressable subsystem reference we have in the ``registeredSubsystemAddressables`` array.