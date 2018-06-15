---
layout: post
title:  "Understanding LLVM"
description: Understanding LLVM, LLVM IR, clang etc ... 
img:
date: 2018-06-14  +1630
---

# What is LLVM?
+ Stands for Low-Level Virtual Machine but actually represents the whole "umbrella" of low-level toolchain components (assemblers, compilers, debuggers, LLVM IR, its implementation of the C++ Standard Library(!) etc.)
+ It is a set of modular and reusable libraries with well defined interfaces.
+ Motivation was that GCC did not allow to use parts of the compiler like parser etc individually. The runtime for GCC is a **monolithic** lump of code that is either included or excluded.

## Compiler Design
+ A typical compiler consists of a front-end, the optimizations section and the back-end.
+ The output of the front-end and the input of the back-end is usually an Intermediate Language which is typically standardized so that the M (number of languages) * N (number of architectures) problem is converted into a M + N problem.

## Existing models of compilers
1. The first model is that of Java and .NET VM's.
   + There is a JIT compiler and a bytecode format.
   + Multiple languages (like Java, Scala, Kotlin ... in case of JVM) all compile to the same bytecode and thus use the same JIT.
   
2. The second method is to compile to C code and thus make use of years of GCC optimizations.
   + This method might cause problems for exception handling and debugging.
   
3. The third method is something like what GCC does.
   + It provides support for multiple front-ends (C/C++/Objective-C/Fortran) and back-ends in the same compiler.
   + Unfortunately, GCC is a giant monolithic piece of code and we cannot use only portions of it.
   
All three methods are monolithic beasts.

# LLVM IR
+ LLVM has come up with its own IR language called LLVM IR.
+ LLVM IR is a low-level RISC instruction set.
+ It has its own type system which is supposed to be very rich.
+ What makes it different is that any front-end can independantly output code in LLVM IR format and pipe it to the optimizations step.
+ **Front-end**:
  + Responsible for parsing, validating and diagnosing errors and translating to LLVM IR code.

# LLVM
+ LLVM is designed as a **collection of libraries**.
+ The optimizer is designed as a collection of passes.
+ The end user can choose which of the passes he/she requires and thus does not need to unneccessaritly subject code to unrequired passes.
+ The LLVM core contains mostly many optimizers and some backends.

## What is Clang and where does it fit in all of this?
+ Clang is a C/C++/Objective-C compiler (**front end**) and is a part of LLVM.
+ Other front-ends include for Haskell, Rust, Swift etc.

## What is libclang and how is it useful?
+ libclang is a C API to clang which provides features for producing the AST and traversing it.
+ I think it might be useful for meta-level applications like an IDE completion engine or code-checking tools etc.

# References
+ A beautiful explanation of LLVM [Architecture of Open Source Applications](http://www.aosabook.org/en/llvm.html)
+ [A blog post with some code for parsing a C++ program using libclang](https://shaharmike.com/cpp/libclang/)
