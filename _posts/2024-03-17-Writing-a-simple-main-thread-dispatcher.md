---
layout: post
title: Writing a simple main thread dispatcher
description: Creating a very simple but useful main thread dispatcher in the Unity engine.
tags:
  - Unity
---
Sometimes, whenever you are trying to write multi-threaded code (either async or raw threaded code) in Unity, we often face warnings like ``"x can only be called from the main thread"`` because Unity's API is not thread-safe.

There are several types of solutions to this. One of the easiest in terms of implementation is tracking the other thread with a boolean, but it is not really good when we can just write a dispatcher to handle this.

In this post, we'll be writing a very simple main-thread dispatcher in Unity, so we can get rid of those errors.