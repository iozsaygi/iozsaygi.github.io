---
layout: post
title: "Hot reloading C/C++ code with SDL"
description: "Using cross-platform APIs provided by SDL to hot reload C/C++ code."
tag: C/C++
---
Most of us experienced the hot reload within the engine that we used. Unreal should be supporting that out of the box, and you should be able to achieve this in Unity by using some kind of third-party plugin. However, things get really interesting when we try to take a peek behind the scenes. Hot reloading introduces a new angle to how we approach game development, and it is really incredible to see how this affects the speed of development iterations.

For the past few weeks, I was able to create a C/C++ environment to research hot reloading with greater detail. I decided to use SDL to handle the native platform layer, such as window creation, reading inputs, and basic rendering calls because SDL also has cross-platform APIs that really helped me a lot when implementing the hot reload.

There are a lot of small and big details that go into hot reloading, and I think it is a great software engineering project on its own. However, we will try to keep it simple as much as possible during this post, so let's try to understand the actual idea and how it really works.

### Concept of hot reloading
Let's imagine a very simple game development environment. You are not using any engine, just pure C/C++ code and maybe some help from your favorite third-party library. The development cycle for this scenario will probably look something like this within the flow diagram:
![Simple C/C++ development iteration]()