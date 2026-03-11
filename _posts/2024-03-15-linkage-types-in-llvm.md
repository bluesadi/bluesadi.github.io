---
layout: post
title: "Linkage Types in LLVM"
date: 2024-03-15
tags: [LLVM, Compilers]
---

This is a placeholder for the blog post about linkage types in LLVM. You can replace this content with your original post.

## Overview

LLVM defines several linkage types that control how symbols are resolved during linking. Understanding these types is essential for anyone working with LLVM IR or building compiler-based tools.

## Common Linkage Types

- **External**: The default linkage type. The symbol is visible to other modules.
- **Internal**: The symbol is only visible within the current module.
- **Private**: Similar to internal, but the symbol is not present in the object file's symbol table.
- **Weak**: The symbol can be overridden by another module.
- **LinkOnce**: Similar to weak, but the linker can discard the symbol if it's not used.
