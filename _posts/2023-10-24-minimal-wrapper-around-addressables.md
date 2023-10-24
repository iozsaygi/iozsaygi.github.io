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

#### Fields
```cs
[SerializeField] private AssetReferenceGameObject[] registeredSubsystemAddressables;
```
This is the actual array that holds references to the addressable subsystem prefabs that will be loaded into memory by the ``SubsystemStreamingController`` class.

#### Creating instance and requesting load operations for subsystems
```cs
subsystemStreamingController = new SubsystemStreamingController();  
foreach (var registeredSubsystemAssetReference in registeredSubsystemAddressables)
    subsystemStreamingController.RequestSubsystemForLoading(registeredSubsystemAssetReference);  
```
Here, we are creating a new instance of the subsystem streaming controller and requesting load operation for each addressable subsystem reference we have in the ``registeredSubsystemAddressables`` array.


#### Loading queued subsystems
```cs
await subsystemStreamingController.LoadQueuedSubsystemsAsync();
```
We are trying to load the queued subsystems into memory and wait until all load operations are complete. I think it is also good practice to not open the gameplay scene even if a single-load operation fails for subsystems. Because gameplay systems will not be able to function with broken subsystems.


#### Creating clones of loaded subsystems in scene
```cs
subsystemStreamingController.InstantiateLoadedSubsystems(transform);
```
After load operations are complete, actual instances of subsystems are created in the scene at runtime.


#### Initialization of created subsystem clones
```cs
subsystemStreamingController.InitializeInstantiatedSubsystems();
```
Remember the ``OnSceneInitialize()`` callback? This is the place where it gets called. After creating instances of subsystems, we are initializing them one by one.

Now, let's get to the fun part. Subsystem streaming class that actually handles the inner operations like loading, creation, and initialization.

### Subsystem streaming controller
This class contains all the fun! Actual loading and initialization are handled here. Let's inspect each function that we used in the earlier sections.


#### Request subsystem for loading
```cs
public void RequestSubsystemForLoading(AssetReferenceGameObject assetReferenceGameObject)  
{  
    if (subsystemLoadingQueue.Count == subsystemLoadingQueueCapacity)  
    {        Debugger.Log(LogLevel.Warning, $"{nameof(SubsystemStreamingController)}",  
            "Subsystem loading queue already reached its capacity and still receiving requests to load new subsystems");  
  
        return;  
    }  
    subsystemLoadingQueue.Enqueue(assetReferenceGameObject);  
    Debugger.Log(LogLevel.Trace, $"{nameof(SubsystemStreamingController)}",  
        "Queued new subsystem for loading");  
}
```
Basically, it checks if we have reached the load operation capacity before queueing the subsystem for loading. I implemented this just to feel **"safe"** I guess. New load requests will not be handled if the streamer has already reached its capacity for load operations.


#### Load queued subsystems async
Yep, that's a long one, I agree.
```cs
public async Task LoadQueuedSubsystemsAsync()  
{  
    var assetLoadTasks = new Task<GameObject>[subsystemLoadingQueue.Count];  
    byte index = 0;  
  
    while (subsystemLoadingQueue.Count > 0)  
    {        
	var assetReference = subsystemLoadingQueue.Dequeue();  
        var asyncOperationHandle = assetReference.LoadAssetAsync<GameObject>();  
        assetLoadTasks[index] = asyncOperationHandle.Task.ContinueWith(loadedTask =>  
        {  
            if (!loadedTask.IsFaulted && !loadedTask.IsCanceled)  
                return asyncOperationHandle.Result;  
  
            Debugger.Log(LogLevel.Critical, $"{nameof(SubsystemStreamingController)}",  
                $"Failed to load subsystem into memory, the reason was: {loadedTask.Exception?.Message}");  
  
            return null;  
        });  
        index++;    
    }  
    
    subsystemLoadingQueue.Clear();  
    await Task.WhenAll(assetLoadTasks);  
  
    foreach (var assetLoadTask in assetLoadTasks)  
    {        
	    if (!assetLoadTask.IsCompleted || assetLoadTask.Result.gameObject == null)  
            continue;  
  
        Debugger.Log(LogLevel.Trace, $"{nameof(SubsystemStreamingController)}",  
            $"Loaded {assetLoadTask.Result.gameObject.name} into memory");  
  
        if (!loadedSubsystems.Contains(assetLoadTask.Result.gameObject))  
            loadedSubsystems.Add(assetLoadTask.Result.gameObject);  
    }
}
```
Basically, it iterates on the queue that holds references to the addressable subsystem objects. Each time, it removes a subsystem from the queue and starts loading it into memory.

After iterating over the queue, it just logs the name of the loaded subsystem and stores it in the list. That list will be used to create instances of subsystems in the scene.


#### Instantiate loaded subsystems
```cs
public void InstantiateLoadedSubsystems(Transform parent)  
{  
    foreach (var loadedGameObject in loadedSubsystems)  
    {        
    var instantiatedGameObjectReference =  
            Object.Instantiate(loadedGameObject, Vector3.zero, Quaternion.identity);  
  
        instantiatedGameObjectReference.transform.SetParent(parent, true);  
        instantiatedSubsystems.Add(instantiatedGameObjectReference);  
  
        Debugger.Log(LogLevel.Trace, nameof(SubsystemStreamingController),  
            $"Instantiated {instantiatedGameObjectReference.name} into the scene");  
    }
}
```
Minimal function that creates the clones of loaded subsystems in the scene; nothing really fancy.