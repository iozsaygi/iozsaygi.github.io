---
layout: post
title: "Hot reloading C/C++ code with SDL"
description: "Using cross-platform APIs provided by SDL to hot reload C/C++ code."
tag: C/C++
---
Most of us are involved with hot reload in one way or another. Unreal Engine supports it out of the box, and you can also achieve it by using several plugins in the Unity engine as well. If you ever wondered how things work under the hood, you will end up just like me and try to implement it into your workflow. I was lucky enough to spend some time researching it and even implement a very simple version of it by using cross-platform [SDL](https://www.libsdl.org/) APIs.

Before taking a look at some code, let's try to understand the idea of hot reloading.