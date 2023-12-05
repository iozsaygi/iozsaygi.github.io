---
layout: post
title: Terrain modification and creep spread in real-time strategy games
description: Reviewing the creep spread mechanism that is mostly used in real-time strategy games
tags:
  - Unity
---
I think the first strategy game I've ever played was [Age of Empires II: The Conquerors](https://en.wikipedia.org/wiki/Age_of_Empires_II:_The_Conquerors). Being able to experience battles in a historical simulator blew my mind. History was an interesting topic for me when I was a child, so there is no doubt I really enjoyed playing strategy games back in the day, I guess.

RTS games really evolved over the years; new games were introduced almost every year, and each good game improved the genre by adding big features like supporting thousands of units in the scene, new graphical enhancements to the overall environment, and active support for the competitive scene.

I've never been a competitive RTS player, but I was always a fan of the genre. If I need to list my top five RTS games, the list will probably look like this:

1. [Warcraft III: The Frozen Throne](https://en.wikipedia.org/wiki/Warcraft_III:_The_Frozen_Throne)
2. [The Lord of the Rings: War of the Ring](https://en.wikipedia.org/wiki/The_Lord_of_the_Rings:_War_of_the_Ring)
3. [Age of Empires II: The Conquerors](https://en.wikipedia.org/wiki/Age_of_Empires_II:_The_Conquerors)
4. [StarCraft II](https://en.wikipedia.org/wiki/StarCraft_II)
5. [Empire Earth](https://en.wikipedia.org/wiki/Empire_Earth)

Some of these games introduced a really cool feature called creep spread or corruption. Basically, specific races had the ability to corrupt the land to gain an advantage during battle or improve vision through map, which would be crucial to dominating the opponent during the match.

In this post, I will try my best to explain the implementation of this technique, but first, let's try to understand what corruption or creep spread is in RTS games.

### Understanding the creep mechanic
When we talk about the [Undead](https://liquipedia.net/warcraft/Undead) race from Warcraft III, we'll see that they are able to corrupt the land by constructing new buildings on the terrain.

Let's take a look at the screenshot below from Warcraft III.
![Warcraft III Undead Blight](https://github.com/iozsaygi/iozsaygi.github.io/blob/main/assets/imgs/tmacsirtsg/wc3_undead_blight.png?raw=true)

The corrupted terrain has a special texture that represents a special area for the Undead race. Undead units will gain additional positive effects (buffs) if they are on corrupted terrain.

Similar to Warcraft III, StarCraft also has the very same feature for the [Zerg](https://en.wikipedia.org/wiki/Zerg) race.
![StarCraft II Zerg Creep](https://github.com/iozsaygi/iozsaygi.github.io/blob/main/assets/imgs/tmacsirtsg/sc2_zerg_creep.png?raw=true)

Also, in most of the RTS games, creep can block opponents from constructing new buildings on terrain. For example, in StarCraft, [Terran](https://starcraft.fandom.com/wiki/Terran) players will not be able to construct a building on top of a terrain that is corrupted by Zerg players' creep.

Now let's get right into Unity and see how we can implement this feature for our own RTS game!

### Overview
To keep things simple for the sake of this blog post, I will be implementing the feature in a top-down 2D environment. Also, I am not planning to share any shader code since I am not that good with shader development. So I'm sorry for those who were looking for shader source code. We will mostly stick to gameplay logic.

I will be separating the implementation part into several topics to ease the reading process. You can find the specific topics below.

1. Scene preparation
2. Node structure
3. Generating a node map
4. Building placement
5. Implementing creep logic
