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

_If I need to represent the workflow with some kind of graph, it would look like the following:_
<p align="center">
<img src="https://github.com/iozsaygi/iozsaygi.github.io/blob/main/assets/images/hot-reload-workflow.png?raw=true" />
</p>

**There is one thing really crucial:** memory of the game code needs to be managed by the engine so we won't lose the game state between our hot reload calls. Also, detecting the changes in the game code can be a bottleneck for the engine if we try to run it every frame; setting an interval value for it should be a valid move.

Before everything else, we need to define what we are going to hot reload, so let's see how we can represent the game code within the engine.

## Representing the game code
At least for the engine, the game code is pretty simple. It contains a **flag** that represents the availability of the game code, a **path** to the actual shared library on the disk, an **opaque pointer** to hold the code instance, and a **function signature** to target specific functions within the game code.

We will call game code only if the instance is valid and reliable.
```cpp
// The signature of game code that we will be loading from shared library and call within the engine's render loop.  
// This can be expanded with other functions that we would like to hot reload.  
typedef void (*Game_OnEngineRenderScene)(SDL_Renderer* renderer, SDL_Rect rect);

struct game_code {  
    bool isValid;  
    const char* path;  
    void* instance;  
    Game_OnEngineRenderScene onEngineRenderScene;  
};
```

This game code structure works pretty well if we only need to hot reload rendering logic. Needless to say, functions to hot reload are totally dependent on our workflow.

*For the sake of simplicity, our game code is just a single render call that draws a rectangle at given coordinates:*
```cpp
void Game_OnEngineRenderScene(SDL_Renderer* renderer, SDL_Rect rect) {  
    SDL_SetRenderDrawColor(renderer, 255, 255, 255, 255);  
    SDL_RenderFillRect(renderer, &rect);  
}
```

Please note that game code struct is defined within the engine's scope (will be built as an executable) and render calls are defined within the game code's (will be built as a shared library) scope.