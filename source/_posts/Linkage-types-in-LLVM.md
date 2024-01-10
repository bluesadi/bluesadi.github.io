---
title: Linkage Types in LLVM
date: 2024-01-05 00:08:08
tags:
---

Like many other C/C++ compilers, LLVM also needs to inform the linker (ld, lld, etc.) how to merge global values (global variables, functions) of different modules (one module in LLVM corresponds to one object file or one translation unit for linker) at link-time. In LLVM, global values can be divided into 10 categories according to their linkage types. The linker will treat each type of global values differently based on their linkage types. Not considering the linkage types of global values when writing a LLVM Pass may result in unexpected behaviors, especially when you are manipulating the global variables by a Module Pass. In this post, I will demystify the 10 linkage types in LLVM to help you write more robust LLVM Passes.

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

This linkage type doesn't really exist in general linkers. It's only used for some special variables generated by LLVM like `llvm.global_ctors`, which is an array that stores functions to execute before the main function. In the case of ELF file, the `llvm.global_ctors` will be compiled to `.init.array` later in the compilation process. Global variables with AppendingLinkage must be a pointer to an array and equivalent global variables with AppendingLinkage will be merged by appending their array contents togother. 

Here is a simple example. I defined two constructor functions in 1.cpp and 2.cpp respectively, and compiled them to LLVM IR.
```cpp
// 1.cpp
void startupfun1(void) __attribute__((constructor)) { printf("Hello, 1.cpp\n"); }
```
```cpp
// 2.cpp
void startupfun2(void) __attribute__((constructor)) { printf("Hello, 2.cpp\n"); }
```
I got an array with a pointer to function `startupfun1` in 1.ll, and an array with a pointer to function `startupfun2` in 2.ll.
```llvm
@llvm.global_ctors = appending global [1 x { i32, void ()*, i8* }] [{ i32, void ()*, i8* } { i32 65535, void ()* @_Z11startupfun1v, i8* null }]
```
```llvm
@llvm.global_ctors = appending global [1 x { i32, void ()*, i8* }] [{ i32, void ()*, i8* } { i32 65535, void ()* @_Z11startupfun2v, i8* null }]
```
These two arrays are combined into one array with two functions after linked by llvm-link:
```llvm
@llvm.global_ctors = appending global [2 x { i32, void ()*, i8* }] [{ i32, void ()*, i8* } { i32 65535, void ()* @_Z11startupfun1v, i8* null }, { i32, void ()*, i8* } { i32 65535, void ()* @_Z11startupfun2v, i8* null }]
```


## LinkOnceAnyLinkage & LinkOnceODRLinkage

LinkOnceAnyLinkage/LinkOnceODRLinkage is used to implement inline and template functions. The difference between inline/template functions and normal functions is that inline/template functions must be defined in each translation unit that uses this function.

LinkOnceAny and LinkOnceODR are very similar, but LinkOnceAny linkage allows nonequivalent global values to be merged, which is a feature that is not required by inline/template functions (clang and gcc don't actually check whether two global values are equivalent though). In my experience, LinkOnceODRLinkage is far more common than LinkOnceLinkage in C++. For more information about ODR: [Definitions and ODR (One Definition Rule)](https://en.cppreference.com/w/cpp/language/definition)

### Inline Function

For inline function, if you want to inline a function into callers, you must know the exact definition of this function. The definition of the inline function and it's callers must be in the same translation unit, because function inlining is performed at optimization phase. During optimization stage, the compiler doesn't have direct access to other object files. If the inline function's definition is in a different module from it's callers, the compiler usually won't throw an error, but the function will never be inlined.

Below is a simple example:

```cpp
inline void foo() { printf("hello\n"); }

int main() { foo(); }
```
If compiled with `-O0` optimization level, the `foo` function will not be inlined. The `foo` function is marked with the `linkonce_odr` linkage. If other modules also use the same inline function, these inline functions will be merged into one definition at link-time. This is what the linkonce linkage is for - to avoid duplication or collision at link-time if an inline function is not actually inlined. Note that compiler won't necessarily inline functions with `inline` modifer. Instead, the compiler only uses it as a hint. For more detailed explanation: [When should I write the keyword 'inline' for a function/method?](https://stackoverflow.com/questions/1759300/when-should-i-write-the-keyword-inline-for-a-function-method) 
```llvm
; Function Attrs: mustprogress noinline norecurse optnone uwtable
define dso_local noundef i32 @main() #0 {
  call void @_Z3foov()
  ret i32 0
}

; Function Attrs: mustprogress noinline optnone uwtable
define linkonce_odr dso_local void @_Z3foov() #1 comdat {
  %1 = call i32 (i8*, ...) @printf(i8* noundef getelementptr inbounds ([7 x i8], [7 x i8]* @.str, i64 0, i64 0))
  ret void
}
```

If compiled with `-O1` or higher optimization level, the `foo` function will be directly inlined into it's caller, and the `foo` function is discarded.
```llvm
; Function Attrs: mustprogress nofree norecurse nounwind uwtable
define dso_local noundef i32 @main() local_unnamed_addr #0 {
  %1 = call i32 @puts(i8* nonnull dereferenceable(1) getelementptr inbounds ([6 x i8], [6 x i8]* @str, i64 0, i64 0))
  ret i32 0
}
```

### Template Function
Following is an example of template function usage. We can see the `myMax` function is called three times with different template type.
```cpp
// C++ Program to demonstrate
// Use of template
#include <iostream>
using namespace std;

// One function works for all data types.  This would work
// even for user defined types if operator '>' is overloaded
template <typename T> T myMax(T x, T y) { return (x > y) ? x : y; }

int main() {
    // Call myMax for int
    cout << myMax<int>(3, 7) << endl;
    // call myMax for double
    cout << myMax<double>(3.0, 7.0) << endl;
    // call myMax for char
    cout << myMax<char>('g', 'e') << endl;

    return 0;
}
```
In this case, this template function will be expanded to three monomorphic versions. These functions are almost identical. The only difference is that they use different data types to adapt to different calls.
```llvm
; Function Attrs: mustprogress noinline nounwind optnone uwtable
define linkonce_odr dso_local noundef i32 @_Z5myMaxIiET_S0_S0_(i32 noundef %0, i32 noundef %1) #5 comdat {
  ...
}

; Function Attrs: mustprogress noinline nounwind optnone uwtable
define linkonce_odr dso_local noundef double @_Z5myMaxIdET_S0_S0_(double noundef %0, double noundef %1) #5 comdat {
  ...
}

; Function Attrs: mustprogress noinline nounwind optnone uwtable
define linkonce_odr dso_local noundef signext i8 @_Z5myMaxIcET_S0_S0_(i8 noundef signext %0, i8 noundef signext %1) #5 comdat {
  ...
}
```
Like inline function, template function should also be defined in the same translation unit with it's callers, otherwise the compiler will yield an error. Imagine that a module feeds the template functions with three different data types - int, char, and double. In return it needs three different monomorphic definitions of this function at link-time. However, when compiler compiles the module that contains the template function, the compiler doesn't know how many types of monomorphic functions it is supposed to generate! For more details: [Why Can templates only be implemented in the header file in C++](https://www.educative.io/answers/why-can-templates-only-be-implemented-in-the-header-file-in-cpp)

In practice, template functions are usually written in header files so that different modules can include and use the template functions with ease. At link-time, duplicate definitions will be merged into one because they all have LinkOnceODRLinkage.

## WeakAnyLinkage & WeakODRLinkage

Weak linkage is almost the same as linkonce, except that unreferenced globals with weak linkage may not be discarded. In addition, weak definitions can be overrided by normal definitions. This linkage type is used for globals that are declared "weak" in C source code. 

For example, in 1.cpp we have,
```cpp
void foo() __attribute__((weak)) { printf("Hello\n"); }

int main() { foo(); }
```
In 2.cpp, we have,
```cpp
void foo() { printf("World!\n"); }
```
The first `foo` function will be overrided by the second one.

But as far as I know, this feature is not commonly used in practice.

## CommonLinkage
Common linkage is most similar to weak linkage, but has more restrictions:
- Symbols with common linkage must have a zero initializer
- Symbols with common linkage may not be marked as constant

According to LLVM official documents, this linkage type is used for tentative definitions in C, such as `int X;` at global scope. But I didn't see clang produces this type of global values in my experiment.

## AvailableExternallyLinkage