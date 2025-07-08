---
layout: post
title: Workaround for the Git LFS requirement when working with Firebase in Unity
description: Exploring a quick workaround to avoid the Git LFS requirement when working with Firebase SDK in Unity.
tags:
  - DevOps
---
When working on a repository with Git LFS, things can get messy pretty fast if we don't pay attention to our revisions. I find LFS requirements on the Git pretty useful considering they also encourage us to have an eye on what we push into the repository. Especially when working with Firebase in Unity, we have to commit large binary files into the repository. Luckily there's a simple workaround to avoid activating Git LFS by hosting the Firebase SDK in another repository as a submodule.

This method will work until each individual Firebase module doesn't exceed the LFS limit in terms of archived file size.

## Downloading individual Firebase modules
