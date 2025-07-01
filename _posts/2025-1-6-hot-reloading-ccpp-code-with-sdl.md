---
layout: post
title: "Hot reloading C/C++ code with SDL"
description: "Using cross-platform APIs provided by SDL to hot reload C/C++ code."
tag: C/C++
---
Most of us are involved in hot reloading in some way or another. Unreal should be supporting this out of the box; Unity can achieve this with the help of several plugins, and even better, you integrated it into your in-house engine. It is a way to manipulate your game code while it is running on the executable program. Helps to achieve faster development cycles by avoiding restarting the game each time we make a change in the code.

I was lucky enough to research it within the C/C++ context by using cross-platform APIs that the [SDL](https://www.libsdl.org/) library provides. Let's take a look at the following footage where I am manipulating a very basic render call while the game is actually running.
![Hot reload footage](https://github.com/iozsaygi/sdl-hot-reload/raw/main/Showcase/render-call-change.gif)

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

## Loading game code instance
Loading the game code is just a single function call; however, we need to ensure that the path to the shared library is correct; otherwise, this call will fail.
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

First, we are unloading the existing game code instance. This is a must-do in order to open room for the newest version. During unloading the game code, we also mark it as `invalid` so our update loop will be safer.

After unloading the game code, we are triggering a new build (it just executes the `MSBuild` command under the hood) to produce a new version of our `.dll` file. Then it's just calling SDL APIs and some error handling.

We can bind this function to a specific key, or we can just execute it whenever a change is detected in the game code.
## Unloading the game code instance
This is pretty much straightforward. Just freeing some memory with the SDL API.
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

Notice how we are immediately setting `isValid` to false in order to prevent any mistake within the update loop.
## Update loop
Now I am pretty sure it can look like I dumped a huge piece of code suddenly, but I tried my best to produce a very simple update loop that actually triggers hot reloading with a single keybind. I even removed all of the unrelated logic, such as frame rate capping, collision checks, etc.
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

        if (gc->isValid) {
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

Just like most of the game's update loop, it starts the frame by reading input from the player. If specific input (the space key in this case) is received, it tries to update the instance of the game code.

Considering our game code consists of a very simple rendering call, it directly switches to rendering and calls the `onEngineRenderScene` function, which is available within the `game.dll` file. It also checks if the game code is valid before doing that so it won't try to iterate over a null game code instance and crash.

The game code receives the data it requires to work with as a parameter, which results in a preserved game state between unlimited hot reload calls. (Assuming memory layout didn't change for game code at all.)
## Conclusion
Hot reload is a great software engineering project on its own, and there are still related features that I want to experiment with. The form of hot reloading can also be applied to the game assets, such as textures, so I will definitely see what I can do. Also getting rid of the keybind trigger requires me to write some kind of watcher over the file notification system of the OS. Both sound amazingly fun.

_I would also like to share several resources that I followed while implementing it; check them out below:_
- [Hot Reload Gameplay Code: What, why, limitations and examples!](https://zylinski.se/posts/hot-reload-gameplay-code/)
- [A fun little test of a SDL2 platform layer for hot code reloading with rayfork](https://gist.github.com/chrisdill/291c938605c200d079a88d0a7855f31a)
- [Handmade Hero Day 021 - Loading Game Code Dynamically](https://www.youtube.com/watch?v=WMSBRk5WG58)

Thank you for reading this through; I wish you a happy new year!