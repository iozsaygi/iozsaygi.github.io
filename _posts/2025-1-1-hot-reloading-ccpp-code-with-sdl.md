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
Unlike our traditional game development environment, where you have to close your game, change code, and build to see effects on the screen, hot reloading enables us to modify game code on the fly.

Let's visualize it a bit and see how it would look within the engine loop.
<p align="center">  
<img src="https://github.com/iozsaygi/iozsaygi.github.io/blob/main/assets/images/hot-reload-workflow.png?raw=true" />  
</p>

*There are several things that we need to consider when implementing hot reload:*
1. Memory should be managed outside of game code in order to preserve game state between reloads.
2. Our game code needs to be built as a shared library (`.dll` on Windows) in **release mode** so we won't be affected by unnecessary debug hooks on the process.
3. Memory layout changes within a game can require additional effort to keep hot reload stable.
4. This isn't a feature that we need to ship players with our game, so ensuring the hot reload will work only during development is essential.

In case of my own implementation, I decided to build the engine as an executable program that hot reloads the game code, and it is also responsible for the update loop for the overall system.

Now we will inspect some C/C++ code to implement a very basic version of hot reload that works dependent on keybinds. To achieve this, we first need to define our game code so that the engine will be able to manage it.
## Defining the game code
Game code is pretty simple: a **flag** that represents the availability of the game code, a **path** to the actual shared library on the disk, an **opaque pointer** to hold the code instance, and a **function instance**.
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

The important thing is that we can expand the number of functions that we want to hot reload; in this case we are only going to reload simple rendering calls.

We also need some way to load and free the instance of game code; luckily, the SDL library provides us with the cross-platform APIs to achieve this.

_Before going further, check out the following APIs if you want to explore a bit more detail: (These are the core APIs we are going to rely on.)_
- [SDL_LoadObject](https://wiki.libsdl.org/SDL3/SDL_LoadObject)
- [SDL_LoadFunction](https://wiki.libsdl.org/SDL3/SDL_LoadFunction)
- [SDL_UnloadObject](https://wiki.libsdl.org/SDL3/SDL_UnloadObject)
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

Thank you for spending some time to read my post; I wish you a happy new year!