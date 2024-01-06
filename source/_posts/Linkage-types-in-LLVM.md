---
title: Linkage Types in LLVM
date: 2024-01-05 00:08:08
tags:
---

As many other C/C++ compilers, LLVM also needs to tell the linker (ld, lld, etc.) how to merge the global values (global variables, functions, etc.) of different modules at link-time. In LLVM, global values can be divided into 10 types according to their linkage types. The linker will treat each type of global values differently. If the linkage types of global values are not correctly handled during write LLVM Pass (especially when you are planing to manipulate the global variables in a Module Pass), unexpected behaviors may occur. In this post, I will demystify the 10 linkage types in LLVM.

<!-- more -->

// TODO