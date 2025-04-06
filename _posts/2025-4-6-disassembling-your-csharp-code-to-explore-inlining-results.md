---
layout: post
title: Disassembling your C# code to explore method inlining results
description: Using third-party tools to disassemble our C# code and explore the results of method inlining.
tags:
  - C#
---
If you spend enough time tackling optimization challenges, you’ll eventually come across the concept of method inlining. Over the past few weeks, I’ve had the chance to explore it in depth. Before diving deeper, let me briefly explain what it’s all about.

_Put simply, method inlining replaces a method call with the actual body of the method itself._

Why is this considered more performant than a regular function call? _Because it removes the overhead associated with calling a function — such as jumping to another memory location and managing the call stack._

That said, inlining isn’t something you can—or should—sprinkle throughout your codebase. In most cases, the compiler is actually better at deciding when and where inlining makes sense.

Here’s the catch: even though I attempted to manually inline some functions, I had no real way of knowing whether the compiler agreed with my efforts. I wanted to verify if my inlining choices were actually removing the `call` instructions in the generated assembly code. After some digging, I was fortunate enough to discover that [BenchmarkDotNet](https://benchmarkdotnet.org) can help uncover exactly that.

## Conditions for inlinable methods
Even if we try to force certain methods to be inlined, the JIT compiler has its own set of rules when deciding whether a method should actually be inlined.

For example:
- Virtual methods typically aren’t inlined, as their abstraction adds an extra layer that makes inlining difficult.
- Methods larger than 32 bytes of IL code are usually skipped—unless you explicitly use the [MethodImplOptions.AggressiveInlining](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.methodimploptions?view=net-9.0) attribute to suggest otherwise.
- Methods with complex or non-trivial bodies are also less likely to be inlined.

## Requirements
To run the example on your Windows machine, you'll need to install a few things:
- **.NET SDK** is required to build and run the project.  
    The easiest way is to install the **“.NET Desktop Development”** workload via the Visual Studio Installer.
- **BenchmarkDotNet** to disassemble and analyze the generated code.  
    You can install it by running: `dotnet add package BenchmarkDotNet`

## When there is no inlining
Let's consider the following C# code, a very basic one indeed:

```csharp
using System.Runtime.CompilerServices;
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Configs;
using BenchmarkDotNet.Diagnosers;
using BenchmarkDotNet.Running;

BenchmarkRunner.Run<BenchmarkHook>(ManualConfig.Create(DefaultConfig.Instance) 
    .AddDiagnoser(new DisassemblyDiagnoser(new DisassemblyDiagnoserConfig(printInstructionAddresses: true, maxDepth: 3))));

public class Collection
{
    private const int Effect = 10;

    [MethodImpl(MethodImplOptions.NoInlining)]
    public int Add(int number)
    {
	return number + Effect;
    }
}

public class BenchmarkHook
{
	private int _resultCache;

    [Benchmark]
    public void Execute()
    {
	var collection = new Collection();
        _resultCache = collection.Add(1);
    }
}
```

- We first start by running our benchmark. However, as you also noticed, we are setting up a configuration to disassemble our benchmarked C# code. There are several things we can adjust to manipulate the behavior of the disassembler; please check [here](https://benchmarkdotnet.org/articles/features/disassembler.html) for more information.
- Then we have a very basic class with an `Add` function that we marked with [MethodImpl(MethodImplOptions.NoInlining)](https://learn.microsoft.com/en-us/dotnet/api/system.runtime.compilerservices.methodimploptions?view=net-9.0). This tells JIT to not inline this function and possibly generate a code that has more IL byte size than the inlined version.
- Finally, our benchmark class that actually communicates with the BenchmarkDotNet. When we run the benchmarks, it automatically tracks the methods with the `[Benchmark]`attribute.

Let's open up our terminal and run this project to see how assembly is generated for this piece of code. Run the following command on the terminal window: (Within the directory of your `.csproj` file)
`dotnet run -c Release`

After the benchmarks are run, you should be able to see the `<benchmark-name>-asm.md` file within the `BenchmarkDotNet.Artifacts/results` directory. Let's open it up and review our generated assembly code.

```assembly
; BenchmarkHook.Execute()
       7FF9CC74FB50 push      rbx
       7FF9CC74FB51 sub       rsp,20
       7FF9CC74FB55 mov       rbx,rcx
       7FF9CC74FB58 mov       rcx,offset MT_Collection
       7FF9CC74FB62 call      CORINFO_HELP_NEWSFAST
       7FF9CC74FB67 mov       rcx,rax
       7FF9CC74FB6A mov       edx,1
       7FF9CC74FB6F call      qword ptr [7FF9CCA5E8C8]; Collection.Add(Int32)  
       7FF9CC74FB75 mov       [rbx+8],eax
       7FF9CC74FB78 add       rsp,20
       7FF9CC74FB7C pop       rbx
       7FF9CC74FB7D ret
; Total bytes of code 46

; Collection.Add(Int32)
       7FF9CC74FB30 lea       eax,[rdx+0A]
       7FF9CC74FB33 ret
; Total bytes of code 4
```

Let's point out critical things in this generated assembly code:

- There is a `call` instruction to `qword ptr [7FF9CCA8E7F0]; Collection. Add(Int32)`
- The generated code is 46 bytes.
- `Collection.Add(Int32)` was JIT-compiled into a separate function.

Now, what would happen if the compiler decided to inline our benchmarks? Let's see!

## When there is inlining
After adding `[MethodImpl(MethodImplOptions.AggressiveInlining)]`
attribute, the compiler starts to do some optimizations; let's see the assembly-generated code with inlining:

```assembly
; BenchmarkHook.Execute()
       7FF9CC76FB50 mov       dword ptr [rcx+8],0B
       7FF9CC76FB57 ret
; Total bytes of code 8
```

- There are no longer separate calls for `Collection.Add(Int32)`
- Total bytes of generated code is 8.
- Caching result of operation by using `mov` instruction.

The generated code is much more optimized, and instructions are minimized.

## Why would you need this?
This is a nice thing to learn, but why would you need this in any situation?

- You are trying to inline your functions, but you would like to see if the compiler actually listens to your efforts.
- You just want to see the assembly-generated version of your .NET code, and you are just curious about this.
- You are keen to explore optimizations that JIT does, such as loop unrolling, etc., by examining the assembly output.

## Conclusion
Software engineering is a giant ocean for me; feeling the need to constantly learn new things will probably not stop as long as I keep working in this area.

I was very interested in this topic for the past several weeks and learned a lot of things from the following resources:

- [.NET Methods Inlining and Loops](https://www.codeproject.com/Articles/1072041/NET-Methods-Inlining-and-Loops](https://www.codeproject.com/Articles/1072041/NET-Methods-Inlining-and-Loops)
- [When is a method eligible to be inlined by the CLR](https://stackoverflow.com/questions/4660004/when-is-a-method-eligible-to-be-inlined-by-the-clr](https://stackoverflow.com/questions/4660004/when-is-a-method-eligible-to-be-inlined-by-the-clr)
- [C# Aggressive inlining benefits][https://schwabencode.com/blog/2025/01/20/csharp_aggressive-inlining-benefit](https://schwabencode.com/blog/2025/01/20/csharp_aggressive-inlining-benefit

I hope I was able to introduce you to a new concept or at least enable another vision for you. Thank you so much for taking your time to read this through!