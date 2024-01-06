---
title: Linkage Types in LLVM
date: 2024-01-05 00:08:08
tags:
---

As many other C/C++ compilers, LLVM also needs to tell the linker (ld, lld, etc.) how to merge the global values (global variables, functions, etc.) of different modules at link-time. In LLVM, global values can be divided into 10 types according to their linkage types. The linker will treat each global value differently based on their linkage types. Not considering the linkage types of global values when writing LLVM Pass may result in unexpected behaviors, especially when you are manipulating the global variables by a Module Pass. In this post, I will demystify the 10 linkage types in LLVM to help you write more robust LLVM Passes.

<!-- more -->

// TODO