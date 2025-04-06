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
    [Benchmark]
    public void Execute()
    {
	var collection = new Collection();
        collection.Add(1);
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
       7FF9CC77FB50 sub       rsp,28
       7FF9CC77FB54 mov       rcx,offset MT_Collection
       7FF9CC77FB5E call      CORINFO_HELP_NEWSFAST
       7FF9CC77FB63 mov       rcx,rax
       7FF9CC77FB66 mov       edx,1
       7FF9CC77FB6B call      qword ptr [7FF9CCA8E7F0]; Collection.Add(Int32)
       7FF9CC77FB71 nop
       7FF9CC77FB72 add       rsp,28
       7FF9CC77FB76 ret
; Total bytes of code 39

; Collection.Add(Int32)
       7FF9CC77FB30 lea       eax,[rdx+0A] 
       7FF9CC77FB33 ret
; Total bytes of code 4
```

Let's point out critical things in this generated assembly code:

- There is a `call` instruction to `qword ptr [7FF9CCA8E7F0]; Collection. Add(Int32)`
- The generated code is 39 bytes.
- `Collection.Add(Int32)` was JIT-compiled into a separate function.

Now, what would happen if the compiler decided to inline our benchmarks? Let's see!

## When there is inlining
