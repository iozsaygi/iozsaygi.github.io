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
	- I don't have an idea about other software development industries, but automated testing plays a huge role in game development. Being able to apply manual test cases to automated pipelines reduces the burden on development teams. Imagine a monkey test where it spawns input on screen to determine user interface and interaction bugs. It is priceless; basically, any ideal test can be automated to have faster iteration speeds during development.
	- There is also branch protection; usually, the main branch is the ultimate version of the current game that has to be chosen over anything else. So protecting that branch from bugs, crashes, and errors is critical. There are different types of automated protection checks that are executed to ensure the health of the main branch. For example, we might want to run a build for specific branches just before merging them into the main to see if it breaks something. We also might want to create a pipeline to ensure game-specific tests for each pull request that is opened. So if some of the tests fail, we simply don't even merge the pull request before it passes all of the tests required.
- **Building and deploying assets**
	- Imagine building an open-world game where you cannot simply load everything into memory at once. This is where the asset streaming concept comes in. Or think about another case: if you want to upload a small executable to the store to increase the download count of your game, players will probably doubt downloading if you have really large download sizes. Asset streaming can solve those problems; basically, upload your assets into a content delivery network and let the game (client) download assets as it needs them. This way, you are going to have smaller download sizes and better memory usage. So for a DevOps engineer, we might want to create an automated pipeline that detects changes on assets and uploads the changed assets to the servers. This way, gameplay code can rely on the latest assets that are downloaded from the server.
- **Versioning**
	- Keeping track of old versions of the game, or the last stable version (which we often call the **"stage"**), is also crucial to clearly overview development. We always want to see a good overview of past releases so we can schedule new ones or address urgent issues and bug fixes accordingly.
	- As always, specific automation pipelines can also be applied to the versioning. Think of a Jenkins pipeline that automatically uploads ``.aab`` files to the Google Play Store to avoid some of the manual work required. Or even better, automating the signing, archiving, and uploading processes of ``.ipa`` is also a perfect thing.

This was the list of responsibilities that I experienced as a DevOps engineer who worked in games. I am sure it will differ based on the industry, but it was a good general overview, I think.

Now there is also another topic that I really want to talk about. The crunch time is a reality of game development, and I think DevOps might be the factor that actually reduces it.

### Effects of DevOps on crunch time
So crunch time is a harsh truth about the world of game development: we work extra hours, sacrificing from our daily routines to ship our beloved game. However, DevOps might be the solution for this.

The speed of our iterations is one of the key factors that determines whether we are going to meet our deadline for the upcoming release. Think of a CI pipeline where everyone has to wait for hours to get their builds in their hands and then finally test them. Crunch is inevitable in this case. 

DevOps principles play a great role in this by ensuring the lowest friction possible for development teams and making sure builds are constantly delivered.

Also, automated test pipelines can lift some of the burden from QA teams to ensure QA time is used wisely. For instance, if we just want to test input-related bugs, we can automate this by creating a monkey test pipeline instead of wasting QA's time. This will create some space for the QA team so they can focus on other, more important test cases and hopefully speed up development and avoid a crunch.

Well, sometimes crunch is inevitable, but I think we have a lot to do as DevOps engineers to help people avoid it.

### Applying DevOps principles from scratch
So I got the chance to work on a team that already implemented DevOps principles into their workflow, and I also got the chance to work on a team that didn't implement DevOps principles into their workflow.

I think I had even more fun when I was the person who implemented DevOps principles into the team's workflow. Because I get to choose which CI tool to use, I get to choose how the build pipeline is going to be built. Deciding on those big pieces was exciting for me. Of course, seeing how it affects development and teamwork is priceless.