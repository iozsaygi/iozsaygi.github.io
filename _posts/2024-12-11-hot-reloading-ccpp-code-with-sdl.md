---
layout: post
title: "Hot reloading C/C++ code with SDL"
description: "Using cross-platform APIs provided by SDL to hot reload C/C++ code."
tag: C/C++
---
Most of us are involved with hot reload in one way or another. Unreal Engine supports it out of the box, and you can also achieve it by using several plugins in the Unity engine as well. If you ever wondered how things work under the hood, you will end up just like me and try to implement it into your workflow. I was lucky enough to spend some time researching it and even implement a very simple version of it by using cross-platform [SDL](https://www.libsdl.org/) APIs.

_Here's a little footage to demonstrate hot reloading of basic SDL rendering calls:_
![Hot reload footage](https://github.com/iozsaygi/sdl-hot-reload/raw/main/Showcase/render-call-change.gif)

Before taking a look at some code, let's try to understand the idea of hot reloading.

## Idea of hot reload
In a very basic game development environment, we are usually building our game code directly into the executable of the platform that we are targeting. This approach works pretty well, but if we want to take advantage of hot reload, we need to force ourselves to think a bit differently.

Instead of directly building our game code into the executable, now we need to treat it as a separate layer and build it as a shared library. Of course we can't just get away by making our game code a shared library; we also have to write a separate application (which I really like to call `Engine` for this case) that is responsible for managing an instance of our game code at runtime. Whenever a change is detected within the game code, we will go ahead and refresh the instance of it within the engine.

_If I need to represent it with some kind of graph, it would look like the following:_
