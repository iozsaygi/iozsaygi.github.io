---
layout: post
title: SerializeReference attribute in Unity
description: Inspecting the useful 'SerializeReference' attribute to serialize objects as reference types instead of value types.
tags:
  - Unity
---
It is incredible that I just learned about this attribute after more than five years of development with the Unity engine.

Recently, I implemented a gameplay feature that required me to store different types of class instances in an array, with these classes sharing their base classes as common. I realized that I cannot do that with the good old '[SerializeField](https://docs.unity3d.com/ScriptReference/SerializeField.html)', and after my research, I learned about the '[SerializeReference](https://docs.unity3d.com/ScriptReference/SerializeReference.html)' attribute.

In this post, I will try to explain the difference between them.