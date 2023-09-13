---
layout: post
title: Changing the behaviour of IL2CPP generated code with compiler arguments
description: List of useful compiler arguments to change the behavior of IL2CPP-generated code.
tags: Unity
---
Greetings everyone!

During the development of a game title where I worked as a DevOps/Build engineer, our gameplay team wanted to explore the capabilities of the IL2CPP compiler to squeeze out every bit of performance.

I immediately started to do research and wanted to share my research in this blog post since I didn't see any blog posts or content that shared a list of IL2CPP compiler arguments that we can use to modify IL2CPP-generated code.

There is an API in Unity's codebase that lets us pass arguments to the compiler during builds; let's start by exploring it first.

### PlayerSettings.SetAdditionalIl2CppArgs
The API is pretty straight forward:

``public static void SetAdditionalIl2CppArgs(string additionalArgs);`` 

It is a static method and just takes a string parameter to pass it on to the IL2CPP compiler.

Here's the example usage:
```cs
class MyCustomPreprocessBuild: IPreprocessBuildWithReport 
{
	public int callbackOrder { get { return 0; } }
	
	public void OnPreprocessBuild(BuildReport report)
	{
		PlayerSettings.SetAdditionalIl2CppArgs("{AdditionalCompilerArguments}"); 
	}
}
```

It will get called automatically by Unity's build pipeline during the builds, while respecting the callback order specified as a property.

There are many more arguments that we can pass to the IL2CPP compiler; let's explore those commands.

### Additional compiler arguments
Of course, Unity has many more arguments, but here are the main ones that piqued our interest:

``-configuration``

Used to specify build configuration, using ``-configuration Release`` configuration will optimize the IL2CPP-generated code for release builds.

``-emit-null-checks``

We thought that if we port our game's source code to fully support value-type objects instead of reference-type ones, this would also mean we could avoid null checks in our gameplay code. If we can avoid null checks in our gameplay code, we can also try to avoid null checks in IL2CPP-generated code to prevent branching. It can be set to ``true`` or ``false``.

``-disable-opt control-flow``

In rare cases (mostly for research), I needed to disable control flow optimizations to try to understand how IL2CPP-generated code is actually structured during conversion. This was not necessary for our case, but I wanted to take a look.

``-strip-bytecode=false``

In our build pipeline, we were automatically disabling the byte code strip process just to improve the debugging experience for our builds in the test environment.

``-disable-opt bounds-checks``

This argument disables the checks for array element accessing operations, which can increase performance since it removes all the array element accessing checks during runtime, but it comes with a risk of undefined behaviour. Should be used carefully.

### Conclusion
These were the top arguments that were interesting for us; there are more arguments to explore.

It was amazing to see how build pipeline changes can greatly impact the performance of our gameplay code. In some scenarios, we can achieve higher frame rates by structuring our game code and modifying our build arguments.

It was definitely an amazing topic to research, and I hope you'll find it useful and use it at some point during your development.

Thank you for reading it out!