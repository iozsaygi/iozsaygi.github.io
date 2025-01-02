---
layout: post
title: "Hot reloading C/C++ code with SDL"
description: "Using cross-platform APIs provided by SDL to hot reload C/C++ code."
tag: C/C++
---
Most of us are involved in hot reloading in some way or another. Unreal should be supporting this out of the box; Unity can achieve this with the help of several plugins, and even better, you integrated it into your in-house engine. It is a way to manipulate your game code while it is running on the executable program. Helps to achieve faster development cycles by avoiding restarting the game each time we make a change in the code.

I was lucky enough to research it within the C/C++ context by using cross-platform APIs that the [SDL](https://www.libsdl.org/) library provides. Let's take a look at the following footage where I am manipulating a very basic render call while the game is actually running.
![Hot reload footage](https://github.com/iozsaygi/sdl-hot-reload/raw/main/Showcase/render-call-change.gif)

If you want to see detailed implementation, you can always check out the [repository](https://github.com/iozsaygi/sdl-hot-reload).

There are many aspects involved in the hot reload, but first we will try to understand the core idea behind it.
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

Please note that game code struct is defined within the engine's scope (will be built as an executable) and render calls are defined within the game code's scope (will be built as a shared library).

Now, let's see how we can manage the instance of game code at runtime.
## Managing game code instance
When it comes to loading shared libraries at runtime, luckily SDL has some cross-platform APIs that come to our aid.

_Take a look at the following code snippet, which loads a new version of game code by triggering a build for it beforehand:_
```cpp
int Engine_TryUpdateGameCodeInstance(struct game_code* gc) {  
    assert(gc != nullptr);  
  
    // Remove the existing game code instance before updating.  
    if (gc->instance != nullptr) Engine_FreeGameCodeInstance(gc);  
  
    if (Engine_TriggerGameBuild() != 0) return -1;  
  
    gc->instance = SDL_LoadObject(gc->path);  
    if (gc->instance == nullptr) {  
        printf("Failed to load game code, the reason was: %s\n", SDL_GetError());  
        return -1;  
    }  
  
    gc->onEngineRenderScene = Game_OnEngineRenderScene(SDL_LoadFunction(gc->instance, "Game_OnEngineRenderScene"));  
    if (gc->onEngineRenderScene == nullptr) {  
        printf("Failed to load game render callback from game code, the reason was: %s\n", SDL_GetError());  
        return -1;  
    }  
  
    gc->isValid = true;  
    printf("Successfully updated the game code instance\n");  
  
    return 0;  
}
```

We won't get into detail on the part where we are triggering a build for game code since it might be specific to your workflow, but at least for Windows it was enough for me to trigger an `MSBuild` on the game's solution file.

Also, please note that we need to **trigger builds in release mode** so we can prevent unnecessary hooks that are embedded into our shared libraries for debugging. This will prevent us from hot reloading the code because it is held by another process at the time.

*So the following steps are performed when loading a new version of game code:*
1. Free the current game code instance (if any)
2. Trigger a build for the game code instance (do not try to reload if the build fails)
3. Load the shared library
4. Load the target function from the shared library

Freeing the current instance of game code is much, much simpler; we are doing this before loading the new instance.

_Take a look at the following code snippet:_
```cpp
void Engine_FreeGameCodeInstance(struct game_code* gc) {  
    assert(gc != nullptr && gc->instance != nullptr);  
  
    SDL_UnloadObject(gc->instance);  
    gc->onEngineRenderScene = nullptr;  
    gc->instance = nullptr;  
    gc->isValid = false;  
  
    printf("Successfully freed the existing game code instance\n");  
}
```

After we are done with the essential part, we just need to construct an update loop in the engine to call our game code if it is valid. Please note that how we are passing required data into the game code within the engine, so the game state is preserved between hot reload calls.

_A very simple update loop that updates the game code instance based on keybind triggers:_
```cpp
void Engine_Update(const struct render_context* rCtx, struct game_code* gc) {  
    assert(rCtx != nullptr);  
    assert(gc != nullptr);  
  
    bool active = true;  
    SDL_Event event;  
  
    while (active) {  
        // Event handling.  
        while (SDL_PollEvent(&event)) {  
            switch (event.type) {  
                case SDL_QUIT:  
                    active = false;  
                    break;  
                case SDL_KEYDOWN:  
                    switch (event.key.keysym.sym) {  
                        case SDLK_ESCAPE:  
                            active = false;  
                            break;  
                        case SDLK_SPACE:  
                            Engine_TryUpdateGameCodeInstance(gc);  
                            break;  
                        default:;  
                    }  
                    break;  
                default:;  
            }  
        }  
  
        // Render scene.  
        SDL_SetRenderDrawColor(rCtx->renderer, 0, 0, 0, 255);  
        SDL_RenderClear(rCtx->renderer);  
  
        // Only call game code if it is valid.  
        if (gc->isValid) {  
            // Notice how we are passing data from engine in order to save game state between hot reloads.  
            SDL_Rect rect;  
            rect.x = 450;  
            rect.y = 350;  
            rect.w = 100;  
            rect.h = 100;  
  
            gc->onEngineRenderScene(rCtx->renderer, rect);  
        }  
  
        SDL_RenderPresent(rCtx->renderer);  
    }  
}
```

This simple example demonstrates how we can hot reload rendering calls of our game, but it didn't actually detect the changes made to the game's code during the update loop. This is something that I am still researching, but triggering hot reloads with key binds also works well for this case.
## Conclusion
_Hot reload is a great software engineering project on its own. I checked a lot of resources to understand it, and there are some amazing ones that I find really helpful. Check them below:_
- [Handmade Hero Day 021 - Loading Game Code Dynamically](https://www.youtube.com/watch?v=WMSBRk5WG58)
- [Hot Reload Gameplay Code: What, why, limitations and examples!](https://zylinski.se/posts/hot-reload-gameplay-code/)

You can also check the [repository](https://github.com/iozsaygi/sdl-hot-reload) where I am still researching this topic.

Thank you for spending some time to read my post; I wish you a happy new year!