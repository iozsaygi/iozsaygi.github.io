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

Let's run down the class quickly:
- We are defining a concurrent queue (which is pretty much thread-safe) to contain registered bundles for dispatching. Visit [here](https://learn.microsoft.com/en-us/dotnet/api/system.collections.concurrent.concurrentqueue-1?view=net-8.0) for learning more about concurrent queues.
- We have defined a very basic function to register dispatchable task bundles that are created from other threads into our queue.
- In the update function (called every frame), we are checking if there are any tasks to dispatch on the main thread. If yes, then we are popping a task from the queue and invoking it on the main thread.

Well, this is the very basic version of dispatcher, instead of using updates, maybe we can try to directly dispatch tasks as soon as they start to arrive in our queue. It is pretty much subject to debate.

Now that we have our dispatcher, let's take a look at the dummy class that works on another thread.

### Implementing the worker thread
For the sake of this post, let's assume we want to generate randomly positioned and colored squares on the screen every three seconds. It will be using raw threads to work with; let's see the whole class.
```csharp
using System.Threading;  
using UnityEngine;  
  
public class ThreadedSpriteGenerator  
{  
    // Execution interval of the thread.  
    private readonly byte interval;  
    private readonly Sprite sprite;  
    private readonly MainThreadDispatcher mainThreadDispatcher;  
  
    // Actual reference to a thread that will be querying tasks on main thread.  
    private Thread thread;  
  
    // Flag to see if our thread is still active.  
    private bool isThreadRunning;  
  
    public ThreadedSpriteGenerator(byte interval, Sprite sprite, MainThreadDispatcher mainThreadDispatcher)  
    {
        this.interval = interval;  
        this.sprite = sprite;  
        this.mainThreadDispatcher = mainThreadDispatcher;  
        isThreadRunning = false;  
    }
    
    public void Begin()  
    {
        thread = new Thread(GenerateSpriteOnMainThread);  
        isThreadRunning = true;  
        thread.Start();  
    }
    
    public void Abort()  
    {
        thread.Abort();  
        isThreadRunning = false;  
    }
    
    private void GenerateSpriteOnMainThread()  
    {
        while (isThreadRunning)  
        {
            // Sleep the thread by an 'interval' amount.  
            Thread.Sleep(interval * 1000);  
  
            // Create the actual task bundle that will be queued.  
            var dispatchableTaskBundle = new DispatchableTaskBundle(() =>  
            {  
                // Create the game object.  
                var gameObject = new GameObject(nameof(ThreadedSpriteGenerator))  
                {     
                    transform =  
                    {         
                        // Randomize the position of game object.  
                        position = new Vector2(Random.Range(-5.0f, 5.0f), Random.Range(-5.0f, 5.0f))  
                    }          
                };  

                // Add sprite renderer component and update the texture.  
                var spriteRenderer = gameObject.AddComponent<SpriteRenderer>();  
                spriteRenderer.sprite = sprite;  
  
                // Randomize color for fun.  
                spriteRenderer.color =  
                    new Color(Random.value, Random.value, Random.value, 1.0f);  
            });  

            // Queue the created task in main thread.  
            mainThreadDispatcher.RegisterTaskBundleForDispatching(dispatchableTaskBundle);  
        }    
    }
}
```

Yep, it will probably look overwhelming, but it is quite simple.

We are simply creating a new game object and adding a sprite renderer to it at every defined interval. This completely happens on another thread, but the actual logic gets executed on the main thread because of the dispatcher we recently implemented.

Now let's connect these two main components by creating a proxy class.

### Implementing a proxy class
It will be pretty basic, just enabling and disabling threaded operations and injecting references around.
```csharp
using UnityEngine;  
  
[DisallowMultipleComponent]  
public class SceneProxy : MonoBehaviour  
{  
    [SerializeField] private MainThreadDispatcher mainThreadDispatcher;  
    [SerializeField] private Sprite sprite;  
  
    private ThreadedSpriteGenerator threadedSpriteGenerator;  
  
    private void OnEnable()  
    {   
        threadedSpriteGenerator = new ThreadedSpriteGenerator(3, sprite, mainThreadDispatcher);  
        threadedSpriteGenerator.Begin();  
    }  
    
    private void OnDisable()  
    {
        threadedSpriteGenerator.Abort();  
    }
}
```

I don't think this even needs a rundown, to be honest; it just enables and disables the worker thread class and shares the references around.

### Wrapping up
The complete implementation can be found in this [repository](https://github.com/iozsaygi/unity-main-thread-dispatcher).

Recently, I implemented a very basic metronome application to study multi-threaded and dependency injection (also to help me while drumming), and that application actually gave me the idea for this blog post. You can find the application repository [here](https://github.com/iozsaygi/tempo-pal).

It was a funny topic to talk about, and I hope you also liked it. See you next time!