---
layout: post
title: "Where Are We with Rust Decompilation? Lessons from Oxidizer"
date: 2026-05-26
tags: [decompilation, rust, oxidizer]
---

In late 2023, I started working on a Rust decompiler that would eventually become Oxidizer. Two and a half years later, in March 2026, it was accepted at IEEE S&P. Getting into a top-tier security conference is exciting. That said, the project itself is still far from where I wanted it to be. This post is a look back at what I've learned along the way, and some thoughts on where Rust decompilation is going in the future.

<!-- more -->

# Background: Decompilation

Have you ever wondered what is happening under the hood when you press F5 in IDA or any other modern decompilers? At a high level, the pipeline is well-worn. The figure below shows the main stages every mainstream tool follows.

![](/assets/img/posts/2026-05-26-rust-decompilation-oxidizer-retrospective/2026-05-28-14-44-18.png)

Reading the diagram left to right, raw bytes are first disassembled into instructions. Those instructions are then grouped into a control-flow graph. The graph is lifted into an intermediate representation that does not depend on the underlying ISA. Type recovery recovers variables and types. Structuring algorithms turn the graph into nested structures including loops and conditionals. The final stage prints something that reads like normal source code. This is also the pipeline Oxidizer uses. We did not invent a new pipeline for decompilation, and we did not need to. Most of these stages have been refined since the late 90s and they work well in practice.

What is different is what each stage was built for. Every mainstream decompiler in use today was developed and tuned against C, and to a lesser extent C++. The choices made inside each stage reflect that, including how memory layout is recovered, how calling conventions are read, how errors are recognized, and how generic code is matched back to its template. When the input binary came from a C-family compiler, the output is good. When it came from somewhere else, the same stages still run and still produce code, but that code often does not match what the original source looked like. This is the gap that matters here, and Rust is the case where it shows up most often in practice today. The rest of this post is about that.

# What are Closed Problems

# What are Open Problems