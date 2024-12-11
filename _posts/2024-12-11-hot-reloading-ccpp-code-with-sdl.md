---
layout: post
title: "Hot reloading C/C++ code with SDL"
description: "Using cross-platform APIs provided by SDL to hot reload C/C++ code."
tag: C/C++
---
Most of us experienced the hot reload within the engine that we used. Unreal should be supporting that out of the box, and you should be able to achieve this in Unity by using some kind of third-party plugin. However, things get really interesting when we try to take a peek behind the scenes. Hot reloading introduces a new angle to how we approach game development, and it is really incredible to see how this affects the speed of development iterations.

For the past few weeks, I was able to create a C/C++ environment to research hot reloading with greater detail. I decided to use SDL to handle the native platform layer, such as window creation, reading inputs, and basic rendering calls because SDL also has cross-platform APIs that really helped me a lot when implementing the hot reload.

First, let's try to understand the idea behind it and how it actually works.