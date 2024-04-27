---
title: VariableRecovery Internals in angr
date: 2024-04-27 11:32:30
tags:
---

VariableRecovery is an important component in decompilation pipeline. In this blog, we will dive into the implementation of VariableRecoveryFast in angr.

<!-- more -->

## The Basics
### Decompilation Pipeline
Let's take a look at the simplified decompilation pipeline of angr. Given a binary, angr first loads it properly according to its format (e.g., ELF, PE, or even Intel Hex), gaining some necessary information for decompilation, such as segmentation information. For code segment, the Instruction Set Architecture (ISA) is identified by angr to lift the platform-dependent assembly code to platform-independent VEX IR. Based on VEX IR, a control flow graph is recovered by CFG Generation algorithms. The recovered CFG is program-wide and then be converted to a set of function-wide CFGs, and at the same time, VEX IR is converted to a higher-level IR - angr Intermediate Language (AIL), for future decompilation. Each AIL graph will be handled with a sequence of simplification passes. Then Variable Recovery comes into play, traversing each AIL statement and expression in the function and assigning variables and type constraints to them. Based on the variables recovered and type constraints collected by Variable Recovery, Type Recovery then infers and assigns a type for each variable by solving type constraints. Eventually, AIL graph will be structured into a sequence of high-level program structures (e.g., for loop, if-else) and structured pseudo-code is outputed to decompiler users.
![](VariableRecovery-Internals-in-angr/decompilation_pipeline.png)

### Forward Analysis
Forward analysis is a program analysis term, which is also heavily used in angr. Imagine you are traversing a control flow graph from the entry block, how would you do that? You would probably use a Broad First Search (BFS) strategy, starting with the entry block, then adding its successors into the queue, getting a new block from the queue and repeating the process. Forward analysis is a similar but more sophisticated strategy, where a block can be put into the queue multiple times until it reaches a limitation of iterations or all of its precessors don't produce anything new. 

## Pre-analysis


## 