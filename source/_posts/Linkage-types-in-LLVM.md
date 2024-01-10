---
title: Linkage Types in LLVM
date: 2024-01-05 00:08:08
tags:
---

As many other C/C++ compilers, LLVM also needs to tell the linker (ld, lld, etc.) how to merge the global values (global variables, functions, and declarations) of different objects at link-time. In LLVM, global values can be divided into 10 types according to their linkage types. The linker will treat each global value differently based on their linkage types. Not considering the linkage types of global values when writing a LLVM Pass may result in unexpected behaviors, especially when you are manipulating the global variables by a Module Pass. In this post, I will demystify the 10 linkage types in LLVM to help you write more robust LLVM Passes.

<!-- more -->

## PrivateLinkage
To start off, let's have a look at the simplest linkage type in LLVM - PrivateLinkage. PrivateLinkage means the global value should only be used in current LLVM Module (one LLVM Module usually corresponds to one object file). Because private global values are local to current module, no symbol needs to show up in any symbol table in the object file. One typical example is constant string as shown below.

```cpp
int main() { 
    puts("this is a private global variable"); 
}
```
After compiled to LLVM IR, it becomes a global variable with `private` prefix, which means it has a private linkage type.
```llvm
@.str = private unnamed_addr constant [34 x i8] c"this is a private global variable\00", align 1

; Function Attrs: mustprogress noinline norecurse optnone uwtable
define dso_local noundef i32 @main() #0 {
  %1 = call i32 @puts(i8* noundef getelementptr inbounds ([34 x i8], [34 x i8]* @.str, i64 0, i64 0))
  ret i32 0
}
```
This is quite straightforward and intuitive. In C/C++, anonymous strings can only be used in current file (more precisely, it can only be used in where it's defined), and obviously it won't have any symbol in the symbol table. 

If two objects have two private global variables with the same name, they will be renamed to avoid collision. Let's compile two cpp files with two strings with the same name, and link then through llvm-link to get the merged LLVM IR.

```llvm
; 1.ll compiled from 1.cpp
@.str = private unnamed_addr constant [5 x i8] c"test\00", align 1
```

```llvm
; 2.ll compiled from 2.cpp
@.str = private unnamed_addr constant [28 x i8] c"this is a ? global variable\00", align 1
```
After linking, one of the variable is renamed to `.str.1` to avoid collision.
```llvm
; llvm-link -S 1.ll 2.ll
@.str = private unnamed_addr constant [5 x i8] c"test\00", align 1
@.str.1 = private unnamed_addr constant [28 x i8] c"this is a ? global variable\00", align 1
```
## InternalLinkage
What if the consant string is named and with a static modifier?
```cpp
static const char str[] = "this is a ? global variable";

int main() { 
    puts(str); 
}
```

This time the global variable's prefix changes from `private` to `internal`, which means it has an internal linkage type. 
```llvm
@_ZL3str = internal constant [28 x i8] c"this is a ? global variable\00", align 16

; Function Attrs: mustprogress noinline norecurse optnone uwtable
define dso_local noundef i32 @main() #0 {
  %1 = call i32 @puts(i8* noundef getelementptr inbounds ([28 x i8], [28 x i8]* @_ZL3str, i64 0, i64 0))
  ret i32 0
}
```
InternalLinkage is similar to PrivateLinkage. The only difference is that global variables with internal linkage will show up in the symbol table as a local symbol (still can't be used by other objects).

Functions with the static modifier also have the InternalLinkage type:
```cpp
static void foo() {
    printf("test\n"); 
}
```
```llvm
define internal void @_ZL3foov() #1 {
  %1 = call i32 (i8*, ...) @printf(i8* noundef getelementptr inbounds ([6 x i8], [6 x i8]* @.str.1, i64 0, i64 0))
  ret void
}
```

## ExternalLinkage

ExternalLinage is also a very common linkage types. This linkage type could be very misleading in the first time. Note that ExternalLinage in LLVM is not equivalent to the `extern` keyword in C/C++. Instead, all global values that are accessible to other objects could have ExternalLinkage, no matter whether they are defined in current module or not.

These are all global variables with ExternalLinkage because they are accessible to other object files at link-time.
```cpp
extern int var1;
int var2 = 1;

namespace ExampleNameSpace {
int var3 = 2;
}
```
```llvm
@var2 = dso_local global i32 1, align 4
@_ZN16ExampleNameSpace4var3E = dso_local global i32 2, align 4
@var1 = external dso_local global i32, align 4
```

Note that global values in anonymous namespace don't have ExternalLinkage because they are inaccessible to other modules.

```cpp
namespace {
int var4 = 3;
}
```

```llvm
@_ZN12_GLOBAL__N_14var4E = internal global i32 3, align 4
```
This rule also applies to functions. For conciseness of this post, I decide not to include the examples here.

## AppendingLinkage

This linkage type doesn't really exist in general linkers. It's only used for some special variables generated by LLVM like `llvm.global_ctors`, which is an array that stores functions to execute before the main function. In the case of ELF file, the `llvm.global_ctors` will be compiled to `.init.array` later in the compilation procedure. Global variables with AppendingLinkage must be a pointer to an array and equivalent global variables with AppendingLinkage will be merged by appending their array contents togother. 

Here is a simple example. I defined two constructor functions in 1.cpp and 2.cpp respectively, and compiled them to LLVM IR.
```cpp
// 1.cpp
void startupfun1(void) __attribute__((constructor)) { printf("Hello, 1.cpp\n"); }
```
```cpp
// 2.cpp
void startupfun2(void) __attribute__((constructor)) { printf("Hello, 2.cpp\n"); }
```
I got this in 1.ll.
```llvm
@llvm.global_ctors = appending global [1 x { i32, void ()*, i8* }] [{ i32, void ()*, i8* } { i32 65535, void ()* @_Z11startupfun1v, i8* null }]
```
And got this in 2.ll.
```llvm
@llvm.global_ctors = appending global [1 x { i32, void ()*, i8* }] [{ i32, void ()*, i8* } { i32 65535, void ()* @_Z11startupfun2v, i8* null }]
```
These two global variables will be linked into an array with two functions by llvm-link:
```llvm
@llvm.global_ctors = appending global [2 x { i32, void ()*, i8* }] [{ i32, void ()*, i8* } { i32 65535, void ()* @_Z11startupfun1v, i8* null }, { i32, void ()*, i8* } { i32 65535, void ()* @_Z11startupfun2v, i8* null }]
```


## LinkOnceLinkage & LinkOnceODRLinkage