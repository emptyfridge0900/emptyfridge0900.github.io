+++
title="CLI? CIL? CLR?"
date=2025-07-09

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++

When preparing for interviews, questions like these always show up:

What is .NET?
What is CIL?
What is CLR?

I had ignored these questions for a while and focused only on syntax or architecture, but today I went down this side road.

If you start following the origin of these terms, there is no end. CIL leads to bytecode, machine code, compiled code, interpreted code, JIT, runtime, cross platform, and so on.

### Common Intermediate Language

The lowest-level code here is machine code: code a computer can execute directly.
Because machine code is hard for humans to read, assembly maps human-readable-ish instructions to machine code.
Each CPU has its own machine code and its own assembly language.
Assembly is not productive enough for most application development, so compilers appeared. If we write code in a high-level language such as C, the compiler converts it into machine code.

The historical distinction between compiled languages and interpreted languages is too complicated to reduce to "hard to develop" versus "easy to develop." The important point here is that .NET does not compile source code only directly into CPU machine code.

Today's topic, Intermediate Language, is literally an intermediate language.
When .NET languages such as C#, F#, or VB are compiled, the result is usually an assembly(.dll/.exe), and that assembly contains metadata and Intermediate Language. In .NET, this IL is also called CIL(Common Intermediate Language).

At runtime, the CLR usually converts this CIL into machine code with Just-In-Time(JIT) compilation. So it is not quite accurate to say ".NET just interprets compiled CIL." Modern .NET also has ahead-of-time options such as ReadyToRun and Native AOT.

.NET calls this CIL, while Java usually calls its equivalent bytecode.

### Common Language Runtime

Now that CIL makes more sense, what is the CLR?
Simply put, it is the runtime environment. It is the component that takes the already compiled intermediate language and turns it into machine code for execution.
For more detail, it is worth reading about [Just In Time](https://en.wikipedia.org/wiki/Just-in-time_compilation) and [Managed Code](https://en.wikipedia.org/wiki/Managed_code).

JIT is a compilation strategy. Managed code means **code whose execution is managed by the CLR**.

When code written in .NET languages such as C#, VB, or F# is compiled, it produces CIL. The CLR turns that CIL into machine code with JIT and runs it. The CLR "manages" that whole execution process.

Things the CLR provides for managed code:

- **Automatic memory management (GC)**: automatically frees objects that are no longer used
- **Type safety**: prevents invalid casts, invalid method calls, and memory-size mistakes
- **Security boundaries**
- **Exception handling**

Unmanaged code such as C or C++ makes the programmer handle these responsibilities directly. Managed code lets the CLR carry that burden.

CIL itself is managed code, but managed code is not the name of the intermediate language format. It is a broader concept describing code executed under CLR management.

### Common Language Infrastructure

If CIL is the compiled intermediate language and CLR is the execution engine that runs it, what is CLI?

Translated literally, Common Language Infrastructure does not feel very intuitive. It is easier to think of it as a technical specification.

CLI is a standard that says: "Implement a runtime, type system, metadata, and intermediate language according to these rules."

In the CLI specification, the runtime area is called the VES(Virtual Execution System), and Microsoft's implementation is the CLR. In modern .NET, CoreCLR fills that role.

.NET does not mean only the runtime. It is a development platform that includes the runtime, base libraries, SDK, compilers, and app stacks such as ASP.NET Core, Windows Forms, and WPF. CLR/CoreCLR is the core runtime responsible for execution.

### Three-line summary

1. I do not want to study this.
2. There is too much to organize. Just follow the links below.
3. I have done more than ten interviews, and nobody asked this anyway.

### Ref
https://blog.naver.com/ya3344/223125448918
https://en.wikipedia.org/wiki/Common_Intermediate_Language
https://en.wikipedia.org/wiki/Common_Language_Runtime
https://www.sysnet.pe.kr/2/0/1697
https://blog.naver.com/leusin/223697119291
https://namu.wiki/w/%EA%B8%B0%EA%B3%84%EC%96%B4
https://learn.microsoft.com/en-us/dotnet/core/introduction
https://learn.microsoft.com/en-us/dotnet/standard/managed-code
https://learn.microsoft.com/en-us/dotnet/standard/clr
https://learn.microsoft.com/en-us/dotnet/standard/assembly/
https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/language-specification/introduction
https://learn.microsoft.com/en-us/dotnet/core/deploying/native-aot/
