---
layout: post
title: Writing a simple main thread dispatcher in Unity
description: Creating a very simple but useful main thread dispatcher in the Unity engine.
tags:
  - Unity
---
Sometimes, whenever you are trying to write multi-threaded code (either async or raw threaded code) in Unity, we often face warnings like ``"x can only be called from the main thread"`` because Unity's API is not thread-safe.

There are several types of solutions to this. One of the easiest in terms of implementation is tracking the other thread with a boolean, but it is not really good when we can just write a dispatcher to handle this.

In this post, we'll be writing a very simple main-thread dispatcher in Unity, so we can get rid of those errors.

### Implementing a wrapper for dispatchable tasks
First things first, we'll start by creating a custom value type that will contain references to tasks that we want to execute in the main thread. A simple read-only structure can achieve this. Since it is read-only, it also supports thread-safe programming.

```csharp
using System;

public readonly struct DispatchableTaskBundle
{
    public readonly Action Task;

    public DispatchableTaskBundle(Action task)
    {
        Task = task;
    }
}
```

Just contains reference to task; we are going to fill those tasks from another thread that we'll be creating later on.

### Implementing a queue-based dispatcher
Well, the main component of our system is the dispatcher itself. It will be receiving the bundles we created from other threads and executing the context of the bundle in Unity's main thread.

The whole concept looks like this:
```csharp
using System.Collections.Concurrent;
using UnityEngine;

[DisallowMultipleComponent]
public class MainThreadDispatcher : MonoBehaviour
{
    // Thread-safe queue for registering received task bundles.
    private readonly ConcurrentQueue<DispatchableTaskBundle> dispatchableTaskBundles = new();

    public void RegisterTaskBundleForDispatching(DispatchableTaskBundle dispatchableTaskBundle)
    {
    dispatchableTaskBundles.Enqueue(dispatchableTaskBundle);  
    }  
    private void Update()  
    {        
    if (dispatchableTaskBundles.IsEmpty) return;  
  
        dispatchableTaskBundles.TryDequeue(out var dispatchableTaskBundle);  
        dispatchableTaskBundle.Task.Invoke();  
    }
}
```