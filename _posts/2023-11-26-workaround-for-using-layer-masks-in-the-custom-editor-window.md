---
layout: post
title: Workaround for using layer masks in the custom editor window
description: A workaround that I explored for using layer mask fields inside custom editor windows.
tags:
  - Unity
---
When creating an editor tool that heavily relies on raycasting, I wanted to implement raycasting for specific objects in specific layers for obvious reasons.

But somehow, the raycats were not working at all; it was all set and correct in the editor window fields, but something was missing. After a few hours of hair pulling, I finally explored a workaround that actually lets programmers use layer masks inside the custom editor window.

But keep in mind that the code that I am about to reveal is written with Unity version ``2022.3.13f1`` (which is the latest LTS version available), so the API might differ based on your Unity version.