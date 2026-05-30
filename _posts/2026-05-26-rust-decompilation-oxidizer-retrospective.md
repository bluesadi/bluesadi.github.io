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

What changes with Rust is not the pipeline but what each stage was built for. Every mainstream decompiler in use today was developed and tuned against C, and to a lesser extent C++. The choices made inside each stage reflect that, including how user-defined types are recovered, how calling conventions are recovered, and how errors are recognized. When the input binary came from a C-family compiler, the output is good. When it came from Rust, the same stages still run and still produce code, but that code often does not match what the original source looked like. The best-effort gaps that C decompilation already has with get wider, and Rust adds several of its own.

The stages do not strain equally under Rust. Since there is no single correct answer to recover, it helps to sort the steps by the kind of method that can recover them. Some are **deterministic**: a fixed algorithm reads the binary and produces a definite answer, and while it is not always exact, the errors are rare, sit at the instruction level, and are not specific to Rust. Some are **deterministic given the types**: once the types are known, a fixed rule recovers them with no further guessing, which pushes all the uncertainty onto recovering the types in the first place. Part of that recovery is a **deterministic heuristic**: a fixed rule that gets the common case right and always returns the same answer, with a known set of cases, often Rust-specific, where it is wrong, like reading which functions came from the standard library or how a function passes its arguments. And the hardest part, including the user-defined types that everything downstream leans on, is only **probabilistic**: no fixed rule recovers it, because the information is either gone or maps back to more than one source, so the best anyone can do is pick the most likely reading.

The rest of this post walks Rust decompilation through these four kinds, from the most certain to the least. Some parts are deterministic and solid, some are deterministic once the types are in hand, the scaffolding those types hang on comes from a heuristic that works in practice as long as you know where it breaks, and the rest, including the types themselves, is probabilistic at best.

# Deterministic

These are the parts a fixed algorithm recovers, where Rust behaves like any other compiled language. The methods here are not perfectly accurate. Disassembly, for one, can still get a handful of instructions wrong on a hard binary. But the accuracy is more than enough in practice and the errors are not specific to Rust, so this is the stable baseline everything else builds on. The section is short for that reason.

The clearest example is disassembly and control-flow graph recovery. Rust compiles through LLVM, so the machine code it emits looks like the machine code any LLVM-based C or C++ compiler emits. The same disassemblers and CFG recovery run on it without changes, and they reach the same accuracy. It also applies to function boundary identification.

# Deterministic given types

A surprising amount of Rust-specific recovery is fully deterministic, on one condition: you have to know the types first. Each step below applies a fixed rule to the types it is handed and adds no guessing of its own. That defers an obvious question, where the types come from, and the two sections after this are the answer, one reliable and one not. For now, assume the types are in hand and watch how much follows mechanically.

## Struct and enum initialization

Once the type of a struct or enum is known, recovering how an instance was built is mechanical. The compiler turns a struct literal into a sequence of field writes at fixed offsets, and Oxidizer groups those writes back into a single initializer by matching each one to a field of the known type. Enums fold back the same way, with the discriminant write and the variant's field writes collapsing into one variant construction. There is no choice to make here. Given the type and the offsets, the grouping is forced. Get the type wrong upstream and the writes group into the wrong shape, but that error belongs to the type step, not this one.

## Pattern matching and the `?` operator

A `match` on an enum compiles into a read of the discriminant and a chain of branches, which a C decompiler shows as raw integer reads and `if`-`else`. Given the recovered enum type, turning that chain back into a `match` or `if let` is deterministic, because the discriminant values map one-to-one onto the named variants. Error handling rides on the same step, since a `match` on a `Result` that returns early on the `Err` variant collapses back into the `?` operator by a fixed rule. None of this guesses, as long as the enum type and its discriminant were recovered.

## Deref coercion

Rust converts between related reference types on its own. A `&String` passed where a `&str` is expected becomes a `Deref::deref` call, which the compiler often inlines down to a raw pointer-and-length access. Given the pair of types involved, folding that access back to the implicit coercion is a fixed rewrite, so the recovery is deterministic once the types are known. The limit is coverage rather than correctness. The rewrite has to be specified per type pair, and Oxidizer ships the common ones such as `&String` to `&str` and `&Vec` to a slice, so a coercion between types it carries no rule for stays in its lowered form.

# Deterministic Heuristic

Everything in the previous section assumed the types. Recovering them is the step that does not get to be deterministic, and it comes apart into two halves. The reliable half is here: which functions came from the standard library, and how each function passes its arguments and returns. Both are read by a fixed rule that is usually right. The other half, the actual user-defined types that everything downstream depends on, is where the precision falls apart, and it sits in the section after this one.

## Rust standard library functions

Rust links its standard library statically, so in a stripped binary a call to `Vec::push` looks like a call to any other function. The recovery is signature matching. Oxidizer first identifies the compiler version with FLIRT, then matches library functions against a signature set built for that exact version, and applies a database of known standard-library struct and function types on top. When the signatures match, a large amount of otherwise opaque code gets named and typed at once. It breaks when the binary was built with a compiler version you do not have signatures for, when a function was inlined away so there is no separate body to match, or when a function is too simple to be unique. FLIRT masks out the call addresses in a signature, so a function that is just one or two calls looks like every other one under that mask.

## Calling conventions

Reading how a function passes its arguments and hands back its result is a fixed rule, because Rust lowers calls onto the platform ABI even though it does not promise a stable one. Oxidizer reads the prototype this way, including the case where a small struct comes back in registers such as `rax` and `rdx` rather than through a caller-allocated buffer, which it recognizes as a multi-register return. This recovers the shape of the interface, how many arguments a function takes and where its result lives, and it does so reliably. What it does not tell you is what those arguments and results actually are. That is the type question, and it is a different kind of problem, the subject of the next section.

# Probabilistic

These are the cases where no method recovers a reliable answer, because the information was either erased by the compiler or maps back to more than one source. Some of them still run a fixed procedure, but it can do no better than return a likely answer, and some have no handle to grab at all. Either way the honest label is a probabilistic guess, the kind of likelihood a learned model might assign, and saying so is the point of the retrospective.

## Struct and enum types

The deterministic-given-types section needed the types, and this is where they were supposed to come from. The honest framing is that recovering a type is a probabilistic problem wearing a deterministic disguise. Rust monomorphizes and erases enough that many distinct types compile to the same layout, so a type is not a fact to read off the binary but a judgment about which type is most likely given the evidence, exactly what a probabilistic method is for. Oxidizer does not use one. It runs inter-procedural prototype inference and a constraint-based solver, a deterministic heuristic, because that is the tractable thing to build, and it still gets the best type recovery among the tools measured. But a fixed rule can only approximate a distribution, and the numbers are the size of the gap: struct precision in the single digits, recall around a third, with enum types, which Oxidizer alone recovers at all, in the same low range. Field order widens it further, since Rust may reorder fields for packing and a wrong guess groups the writes into the wrong shape. The deterministic method is not failing at a deterministic problem, it is standing in for a probabilistic one. That is why types belong here, and it is the floor the whole deterministic-given-types layer rests on.

## Generics

Rust compiles a generic function by stamping out a separate copy for every concrete type it is instantiated with. By the time the binary exists there is no generic left, only a set of concrete functions that happen to share a shape. `core::ptr::drop_in_place::<T>` alone can appear in dozens of versions, one per `T`. Recovering the generic means recognizing that these copies came from one definition and abstracting the type back out, and Oxidizer does not attempt it. It recovers the concrete functions, and for standard-library generics it can lean on signatures, but reconstructing the generic abstraction is left open. Demangled symbols help when they survive, since the names still spell out the type arguments, and stripped binaries take even that away.

## Traits and dynamic dispatch

A call through a trait object goes indirectly, through a vtable of function pointers reached from the second half of a fat pointer. The information needed to name the trait and bind the call back to a concrete method is spread across the vtable, the type that owns it, and the call site, and the optimizer is free to devirtualize, inline, or merge vtables on the way. Oxidizer does not recover traits, and no tool measured does. What you get is an indirect call through a pointer, faithful to the binary but silent about the trait the programmer wrote.

## Closures

A closure compiles into an anonymous struct that holds the variables it captured, plus an ordinary function that takes that struct as an argument. Iterators and `async` functions are built the same way, a tower of small generated types that the optimizer fuses into a plain loop or an enum-shaped state machine. The loop, the struct, and the branches all recover correctly, so behavior is preserved, but the closure or the iterator chain the programmer wrote is gone and there is no reliable way back. Oxidizer recovers the underlying code and leaves the abstraction alone, the same as it does for generics and traits.

## Macros

A macro is arbitrary code generation. It matches a token pattern and expands into whatever code the macro author wrote, so by the time it reaches the binary there is no fixed shape to invert. Recovering the original call means recognizing that some block of code was generated rather than written, and in general that is a probabilistic judgment rather than a rule. Oxidizer does carve off a deterministic slice of the problem: the standard-library macros such as `println!`, `format!`, and `panic!` expand in known ways, so it outlines those recognized expansions back into a call. Even with the whole standard library covered, that is only about a tenth of the macros in the dataset, and the rest, especially user-defined macros and anything compiler optimization has reshaped, offer no fixed handle to grab.

## Enums the compiler hid completely

The previous section recovers an enum as long as its discriminant is somewhere to be read. When every variant fits without a spare field, the compiler stops storing a discriminant at all and folds it into an unused bit pattern, so `Option<&T>` becomes a single pointer that encodes `None` as null. There is no tag to find, and the layout is indistinguishable from a plain nullable pointer or integer. The paper names this directly as a limitation. A fully niche-optimized enum cannot be told apart from the raw value it was packed into.

## When the binary is honestly ambiguous

Some cases have no deterministic answer even in principle. A function that returns a struct can look identical to one that returns an enum of two same-size variants, because at the machine level both write the same bytes into the same slot. Recovering some function argument types runs into the same wall, and Oxidizer leaves a few unrecovered for exactly this reason. This is the floor the opening of the post was pointing at. Where two source constructs compile to the same code, no decompiler recovers which one was written, only a plausible one.

## Tied to one compiler

The last open problem is not about any single construct but about how the recovery is built. The heuristics encode what one `rustc` version does, so they depend on type databases regenerated per version, and a change in code generation, enum layout, or the unstable ABI can move the ground under them. Compiler optimization makes it worse, leaving stray `goto`s and reshaping the very expansions that macro and string recovery look for. Reverting optimization before structuring is known to help for C, and doing the same for Rust is still future work.

# Where Are We, Then

Putting the four kinds side by side gives a rough map of where Rust decompilation stands today.

| Concern | Method | Why |
| --- | --- | --- |
| Disassembly, CFG, function boundaries | Deterministic | LLVM output, boundary-finding is no different from C |
| Struct and enum initialization | Deterministic given types | Field writes group into the known type by a fixed rule |
| Pattern matching and `?` | Deterministic given types | Discriminant values map one-to-one onto named variants |
| Deref coercion | Deterministic given types | Folded back per type pair once the types are known |
| Standard-library function identification | Deterministic heuristic | Per-version signature matching, fails on unknown versions or inlining |
| Calling conventions | Deterministic heuristic | ABI read by a fixed rule, recovers the interface shape not the types |
| Struct and enum types | Probabilistic | A likelihood judgment at heart, approximated by a deterministic rule at single-digit precision |
| Generics, traits, closures | Probabilistic | Not attempted, the abstraction is gone from the binary |
| Macros | Probabilistic | Arbitrary expansion, only the known standard ones outline back |
| Fully niche-optimized enums | Probabilistic | No stored discriminant, indistinguishable from a raw value |
| Struct-vs-enum returns, some argument types | Probabilistic | No deterministic answer at the machine level |
| Compiler-version and optimization fragility | Probabilistic | Heuristics tied to one rustc, optimization adds noise |

The shape of the map is the honest answer to the question in the title. The base of the pipeline is deterministic and solid. A wide band above it is deterministic too, but only once the types are known, so it is exactly as good as the type recovery feeding it. And type recovery is a probabilistic problem at heart, one Oxidizer can only approximate with a deterministic rule, at single-digit precision on the structs and enums everything downstream depends on, so the reliable parts of the pipeline are standing on a probabilistic floor. The abstractions that make Rust feel like Rust, generics, traits, and closures, sit further out on that same floor, where no fixed rule reaches them at all. That is the gap between getting into S&P and having the decompiler I actually wanted.

If I had to say where the next gains are, it is type recovery, full stop. So much of the pipeline is already deterministic given the types that lifting their precision would lift everything above it at once. It is a probabilistic problem and a genuinely hard one, since Rust erases enough that different types share a layout, but it is not the deepest kind of hard, the kind where two constructs compile to the very same bytes by definition. There is room to make the guess better, by encoding what the Rust compiler actually does instead of what a C compiler would have done, and by pushing the recovery to survive a change of compiler version. That is the bet Oxidizer makes, and it is where I think Rust decompilation goes next.
