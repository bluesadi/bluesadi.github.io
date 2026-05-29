---
layout: post
title: "Where Are We with Rust Decompilation? Lessons from Oxidizer"
date: 2026-05-26
tags: [decompilation, rust, oxidizer]
---

In late 2023, I started working on a Rust decompiler that would eventually become Oxidizer. Two and a half years later, in March 2026, it was accepted at IEEE S&P. Getting into a top-tier security conference is exciting. That said, the project itself is still far from where I wanted it to be. This post is a look back at what I've learned along the way, and some thoughts on where Rust decompilation is going in the future.

<!-- more -->

# Decompilation Is a Best Effort

The most important lesson I learned is that decompilation is inherently a best-effort problem. Most high-level information is lost during compilation: variable names, types, and control-flow structures aren't stored in the binary, and many different programs compile to the same machine code, so there's no single correct way back. A decompiler can only reconstruct something functionally equivalent to the source, and how readable that output is varies widely.

That does not stop the tools from trying, and they are good at it. Have you ever wondered what is happening under the hood when you press F5 in IDA or any other modern decompiler? At a high level, the pipeline is well-worn. The figure below shows the main stages every mainstream tool follows.

![The decompilation pipeline, from raw bytes to pseudocode](/assets/img/posts/2026-05-26-rust-decompilation-oxidizer-retrospective/pipeline.svg)

Reading the diagram left to right, raw bytes are first disassembled into instructions. Those instructions are then grouped into a control-flow graph. The graph is lifted into an intermediate representation that does not depend on the underlying ISA. Variable recovery works out where variables live, what their types are, and what to name them. Structuring algorithms turn the graph into nested structures including loops and conditionals. The final stage prints something that reads like normal source code. This is also the pipeline Oxidizer uses. We did not invent a new pipeline for decompilation, and we did not need to. Most of these stages have been refined since the late 90s and they work well in practice.

Even for C, several of these stages are guesses more than computations. The binary does not record where one stack variable ends and the next begins, whether a value is a signed integer or a pointer, or what any of it was called. Variable recovery leans on heuristics to recover those locations, types, and names, and structuring has to pick one arrangement of loops and conditionals out of the many that produce the same control flow. These steps are tuned well enough that the output reads cleanly most of the time, but there is no single right answer underneath, only a best effort that usually lands close.

# Rust Makes It Harder

What changes with Rust is not the pipeline but what each stage was built for. Every mainstream decompiler in use today was developed and tuned against C, and to a lesser extent C++. The choices made inside each stage reflect that, including how memory layout is recovered, how calling conventions are read, how errors are recognized, and how generic code is matched back to its template. When the input binary came from a C-family compiler, the output is good. When it came from Rust, the same stages still run and still produce code, but that code often does not match what the original source looked like. The best-effort gaps that C decompilation already lives with get wider, and Rust adds several of its own.

The stages do not strain equally under Rust. Since there is no single correct answer to recover, it helps to sort the steps by how confidently we can recover one, not just by whether they are solved. For some steps the method is principled and the result is reliable. It is not always exact, but the errors are rare, sit at the instruction level, and are not specific to Rust, so we can treat the result as a solid baseline. For others there is no exact method, but a heuristic gets the common case right, with a known set of cases, often Rust-specific, where it breaks. And for the rest there is no reliable method yet. The information we want is either gone or too ambiguous to pin down.

The rest of this post walks Rust decompilation through these three buckets. That is the most honest way to answer where we are. Some parts are solid, some work well in practice as long as you know the failure modes, and some are still open.

# Solidly Recoverable

These are the parts where Rust behaves like any other compiled language. The methods here are not perfectly accurate. Disassembly, for one, can still get a handful of instructions wrong on a hard binary. But the accuracy is more than enough in practice and the errors are not specific to Rust, so this is the stable baseline everything else builds on. The section is short for that reason.

The clearest example is disassembly and control-flow recovery. Rust compiles through LLVM, so the machine code it emits looks like the machine code any LLVM-based C or C++ compiler emits. The same disassemblers and CFG builders run on it without changes, and they reach the same accuracy. Jump tables, non-returning functions, and the occasional code-versus-data ambiguity are as hard for Rust as they are for C, and no harder.

Calling conventions are mostly in this bucket too. Rust does not promise a stable ABI, but for any single binary the compiler still lowers calls onto the platform convention, so where arguments live and how values come back can be read the usual way. The places this gets murky, such as how small aggregates are packed into registers, are shared with C++ and are not unique to Rust.

Function boundaries round out the list. As long as symbols are present, or the usual prologue and call-target heuristics apply, splitting the binary into functions works as well as it does anywhere else. None of this is where Rust decompilation struggles, which is exactly why it belongs here.

# Good Heuristics, Known Failure Modes

Here there is no exact method, but a heuristic gets the common case right. For each item the structure is the same: what the standard pipeline expects to see, what Rust actually emits, when the heuristic holds, and when it breaks. This is where Oxidizer does most of its work.

## Struct and field layout

A C decompiler expects struct fields to sit in memory in the order they were declared. Rust's default representation makes no such promise. The compiler is free to reorder fields to pack them tightly, so the offsets the decompiler reads no longer line up with any source-level order. The heuristic that recovers struct shape from access patterns still works, and the recovered fields are correct, but their order and names are the compiler's, not the programmer's. When a type is declared `repr(C)`, the original order comes back and the output matches what a C decompiler would produce.

## Trait objects and dynamic dispatch

When Rust calls a method through a trait object, it uses a fat pointer. One half points at the data, the other half points at a vtable of function pointers. This pattern is regular enough to recognize. Reading the vtable recovers the set of methods a type implements and turns an indirect call back into a named one. The heuristic holds whenever the vtable is laid out in the standard way. It gets harder when the optimizer devirtualizes the call, inlines through it, or merges identical vtables across types, at which point the one-to-one mapping the recovery relies on is gone.

## Slices, strings, and other fat pointers

Slices and string slices are also fat pointers, a data pointer paired with a length. Once you know to look for the pattern, recovering a slice and its length is reliable, and it makes a large fraction of standard-library calls readable. The failure mode is the same as above. When the length is constant-folded away or the pair is split up by optimization, the pattern no longer presents itself cleanly.

# Still Open

These are the cases where no reliable method exists today. The information we need was either erased by the compiler or is too ambiguous to recover, and being honest about this is the point of the retrospective.

## Generics and monomorphization

Rust compiles a generic function by stamping out a separate copy for every concrete type it is used with. By the time the binary exists, there is no generic function left, only a handful of unrelated-looking concrete ones. Recovering the original generic means recognizing that several functions are instantiations of one template and abstracting the type back out, and there is no reliable method for that today. Demangled symbol names help when they survive, since they still spell out the type arguments, but stripped binaries take even that away.

## Iterators, closures, and state machines

A Rust iterator chain is a tower of small generic types that the optimizer fuses into a single loop. A closure becomes an anonymous struct holding its captured variables plus a call. An async function becomes an enum-shaped state machine. In every case the high-level construct is gone and what remains is ordinary loops, structs, and branches. The decompiler can recover those faithfully, but turning them back into the iterator chain or closure the programmer wrote is open.

## Error handling

Rust's error handling compiles into ordinary control flow. The `?` operator becomes a branch on a returned `Result`, and panics become calls into the unwinding machinery with landing pads scattered through the function. The branches and calls recover correctly, so the behavior is preserved, but the compact source-level shape, a single `?` where the binary shows a match and an early return, does not come back on its own.

## Enum layout and niche optimization

Rust does not always store an enum's discriminant as a separate field. When a variant has an unused bit pattern, the compiler reuses it to encode the discriminant, so `Option<&T>` is just a pointer that may be null, and a richer enum may hide its tag inside a field of one of its variants. With no discriminant field to find, recovering the enum and its variants is ambiguous.

## Ownership and lifetimes

Ownership, borrowing, and lifetimes are checked entirely at compile time and have no runtime representation at all. There is nothing in the binary to recover them from, by design. This one is not so much open as closed in the other direction. The information is provably not there.

# Where Are We, Then

Putting the three buckets side by side gives a rough map of where Rust decompilation stands today.

| Concern | Bucket | Why |
| --- | --- | --- |
| Disassembly, CFG, calling conventions | Solidly recoverable | LLVM output, no different from C |
| Struct and field layout | Good heuristic | Recovered from access patterns, order is the compiler's |
| Trait objects, dynamic dispatch | Good heuristic | Vtable pattern is recognizable until the optimizer hides it |
| Slices, strings, fat pointers | Good heuristic | Pointer-plus-length pattern is reliable when intact |
| Generics, monomorphization | Still open | No reliable way to merge instantiations back to a template |
| Iterators, closures, async | Still open | The high-level construct is fused away into plain loops and structs |
| Error-handling shape | Still open | Behavior is preserved, the compact `?` form is not |
| Enum niche optimization | Still open | No discriminant field to recover the variants from |
| Ownership, lifetimes | Gone by design | No runtime representation exists |

The shape of the map is the honest answer to the question in the title. The base of the pipeline is solid, a useful middle layer works as long as you know its failure modes, and the parts that make Rust feel like Rust, generics, iterators, and rich enums, are still mostly out of reach. That is the gap between getting into S&P and having the decompiler I actually wanted.

If I had to pick where the next gains are, it is the middle bucket more than the open one. The open problems are open for deep reasons, some of them provable, like ownership leaving no trace. The heuristic cases are where a Rust-aware tool can still move the needle, by encoding what the Rust compiler actually does instead of what a C compiler would have done. That is the bet Oxidizer is making, and it is where I think Rust decompilation goes next.
