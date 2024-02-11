---
layout: post
title: DevOps in game development
description: My opinions about game development with and without DevOps culture.
tags:
  - DevOps
---
I've been working in the games industry for almost four years now, and I was always a generalist programmer. Worked on several mid-core-scaled titles for different platforms like PC, mobile, and virtual reality.

I wore different hats during the production of each game that I've worked on. Implemented gameplay features, created engine tools for other members of the development team, and handled build and asset streaming pipelines.

Frankly, dealing with the tools and build pipelines was more interesting to me than writing pure gameplay code. I think I just loved the feeling of being a supportive member of the team by creating tools and solid deployment pipelines.

Throughout those four years, I've had the chance to work on several game studios that have implemented or not implemented the DevOps culture, and I think I've got something to share.

**Remove later: START**

Planned sections:
1. The role of DevOps in game development
2. Effects of DevOps in crunch time
3. Is DevOps really a responsibility of a single person or team?

**Remove later: END**

### The role of DevOps in game development
DevOps is a crucial aspect for every development team and production lifecycle; however, I think it gets even more important in game development because of its effects on iteration, testing, and crunch time (I will talk more about this in a later section).

In game development, we usually rely on quick iterations over the work that we've done. I am sure that no one will like the idea that they have to wait an hour to test their work before merging into the main branch, and no one will like the idea that they introduced the crash into the game because automated tests were broken. So for a DevOps engineer or DevOps team, there is a lot to cover for ensuring a stable development pipeline through the studio.

I think it would be great if I made a list of responsibilities that I experienced throughout my career. I hope it will give a broad overview of the role of DevOps in game development.

- **Ensuring build stability**
	- I cannot stress this one enough; I think it is the most important responsibility for a DevOps engineer. Always provide the development teams with stable builds, ensuring they are able to iterate on the game without friction. No one in the studio or team should be able to say, _"Hey, I cannot work right now because my builds are failing somehow."_ People need to iterate, detect bugs, and identify crashes. So ensuring that the builds are stable takes the number one priority.
- **Optimizing hardware usage and maintenance**
	- Sometimes we have a lot of things to build, and we have fewer machines or hardware. Being able to take advantage of multiple build machines to reduce build times is crucial. While a single machine is building the actual executable, it would be perfect to have another machine build and deploy assets to content delivery networks at the same time, for example. This will decrease the duration of builds, which will lead to faster iterations and save everyone's time.
	- The maintenance part is also important; ensuring there is always enough space for upcoming builds in hardware and that the tools and software are always up-to-date is achievable by creating automated scripts at the OS level.
- **Automated tests and branch protection**
	- Content