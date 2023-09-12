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
		PlayerSettings.SetAdditionalIl2CppArgs("-emit-null-checks"); 
	}
}
```