---
layout: post
title:  "A module proposal for Cforall (Masters thesis)"
date:   2025-08-14 09:00:00 -500
categories: jekyll update
---

*My masters thesis topic is on building a module system for the [Cforall](https://cforall.uwaterloo.ca) programming language. This differs from a standard blog post because it needs to be rigorous, and it is still a work in progress. I will be updating it as I continue working on this subject.*

## Motivation

This proposal adds a module system to Cforall, an extension to the C programming language. However, most of the information discussed can be used to create a module system in C or another system programming language.

**Why do we need modules?** It's worth noting that not all programming languages have modules (C being the obvious example here). In truth, modules don't tend to increase a language's expressive power, as its features can usually be emulated by other code structures. Where modules become really useful is when managing a codebase with multiple collaborators. Modules have the practical benefit of helping us focus only on the parts of the program we're working on. They also help us avoid accidentally relying on implementation details by restricting what can be accessed by other code, thereby keeping a codebase easier to extend and maintain.

But **isn't a C translation unit a module?** (a c translation unit is a `.c` file after preprocessing) It has `static` and `extern`, but there isn't a good way to analyze which symbols are used where in the program. A symbol defined in `foo.c` can be declared in `foo.h`, but that symbol can also be declared in `unrelated.c`. Calling a C translation unit a module is like saying a struct is well-defined, even if it can only be used if some arbitrary set of functions is called. We don't consider it a module because it doesn't provide a way to control which modules use its symbols. **We argue that C translation units, with some access control, can be considered a module.**

*Some paragraphs of this proposal are italicized - this is to indicate it is specific to Cforall, or my working thoughts. The rest of this proposal is applicable to systems programming languages in general.*

## Objectives

Let's clarify what we're trying to achieve with our modules. The term "module" is as abused as "polymorphism" in programming languages, which makes it challenging to determine what we're working towards with a module system for C. Here are our 3 main objectives:

1. **Make C translation units into modules** (needs to support cyclic modules)
2. **Generate module interfaces** (so .h files are unnecessary)
3. **Symbol exports are decoupled** (symbols are imported independently from each other)

Other topics that will be dicussed include module friendship, backwards compatibility and interactions with Cforall features.

Handling initialization order and importing existing standard libraries is also of major importance, though these topics are complex enough to be pulled into separate sections.

## Overview

A key guiding principle behind this proposal is the idea that symbol exports are decoupled. This means that exporting a symbol should only provide enough functionality for the importer to use it, and nothing more. We try to take this even further: **only the statement in which a symbol is defined is allowed to export its implementation details,** as opposed to having one export implicitly exporting another's details. This means importers only get what they need (and nothing more) and exporters have full control over who has access to their details. To explain by example:

```
// module A
module;
export struct Inner {int a; int b;};

// module B
module;
import A;
export struct Outer {int i; struct Inner f1;};
export struct Outer constructOuter() { /* implementation omitted */ }

// module C
module;
import B;
export struct Outer o = constructOuter();
int x1 = o.i;
struct Inner x2 = o.f1;
int x3 = o.f1.a;  // error, must import definition of struct Inner to use
```

Module C can access the fields of `struct Outer`, but it cannot access the fields of `struct Inner` because it did not import module A. Note that module C can access the name `struct Inner`, because the type system requires the type of `x2` to be written out. Additionally, any module that imports module C does not receive the implementation details of `struct Outer` or `struct Inner`.

The fact that importing an individual symbol is well-defined here forms the basis of our module system. This is because it allows us to reason about individual symbols, without having to consider everything within a module. This greatly simplifies our proofs of correctness. This also makes it trivial to get order-agnostic imports, enables us to make modules cyclic and allows us to remove the need for forward declarations. In our setup, a module file is simply a convenient way to manage symbol scope and package symbols to be exported together.

This module system currently allows 3 kinds of symbols to be exported:
* Types (ie. `struct` and `typedef` statements)
* Functions
* Variables (ie. global variables)

As shown above, sometimes type names need to be visible even if they are not explicitly imported, in order to enable full functionality of other symbols. However, the type must be imported in order to access a type's fields. As for functions and variables, they must be imported in order to be visible or used. Importing an inline function should appear no different than a non-inline function - the only difference being more compilation dependencies and more information available to the optimizer.

Note that there is a distinction between "visible to the module" and "visible to the optimizer" that is different from regular C: Just because something is provided to the optimizer, doesn't mean the code in a module can access it. In contrast, regular C code would be like putting `export` on every exportable symbol and import statement (ie. transitive module dependency). In this proposal, "visible" refers to "visible to the module".

*This module system was in part inspired by the concept of weak and strong modules as described by [this article](https://thunderseethe.dev/posts/whats-in-a-module/) and the paper [Backpack: Retrofitting Haskell with Interfaces](https://plv.mpi-sws.org/backpack/). Put roughly, weak modules are dependent on other modules, while strong modules are dependent on module interfaces. Unfortunately, the concept of what is an interface and what is not is unclear in C (where header-only libraries exist), and certain aspects of systems programming require exposing certain details that would otherwise be hidden in other languages (eg. unboxed types). After multiple attempts of trying to figure out the key principle behind the distinction (as well as figure out if it can be implemented), I eventually reached the key insight of decoupling symbol exports.*

## Walkthrough

A module holds a collection of symbols to be exported. A module also uses imports to control what symbols are visible and how those symbols can be used. We will analyze:
* What are the kinds of symbols and what does the importing module see?
* How does the importing module disambiguate between symbols?

As stated in the overview, we wish to enforce that **only the statement in which a symbol is defined is allowed to export its implementation details.** This means that you cannot use a function or struct that is defined in another module unless that module exports it and you import said module (of course, it may be accessible using `extern "C"`). We will see certain cases where C forces this encapsulation to be somewhat broken (eg. types and inline functions), but we try to limit this "information leakage" as much as we can.

### Symbols of a module

Symbol definitions in our modules (ie. the symbols the module can export) are either types, functions or variables. While types can be defined in the same line as a variable (eg. `export struct {int i;} name = {5};`), we will make the simplifying assumption right now that this does not happen (in the example, both `name` and the anonymous struct would be exported together).

#### Types

There are 3 kinds of types: builtins (eg. `int`), user-defined (ie. `struct`) and aliases (ie. `typedef`). We defer discussion on pointers, references and arrays until afterwards.

Built-ins are provided by the compiler to every module, so we don't need to consider them. We treat aliases like a `struct` with one field, which is implicitly accessed, allowing us to only consider the `struct` case (note that exporting a `typedef` without exporting the underlying type produces different behaviour than what one may be used to).

Taking the most idealized version of our guiding principles, if we export `struct A` that has a `struct B` field (ie. not itself), then we should not export any information related to `struct B`. The importer will be able to access all fields in any `struct A` variable, but the `struct B` field is essentially an opaque type (if a module also wants to access its fields, it needs to import `struct B` too).

Unfortunately, this runs into problems. First, we have unboxed types (boxed types are implemented as a pointer to the actual object). This means we need to know the size and alignment of every field, so we know how large the struct is and the offsets of each field. Second, a very common pattern (so for practical purposes, we cannot ban it) is to initialize a variable with a field, which requires writing down the type name (eg. `struct Exported e; struct Field f = e.f1;`). So importers need to know the size, alignment and name of a field (we also need to consider move/copy constructors for Cforall types, but in C we simply copy bytes). A similar problem occurs when returning types from functions (eg. `export struct Unexported foo();`), which we will discuss in the Functions section.

How do we obtain this size/alignment/name information? We use compile-time reflection to crawl import modules as needed (see Formalism section for details and a proof of correctness). Our technique handles circular imports, as it should only fail if we encounter a type name ambiguity (eg. we import module `A` and `B` which both export `struct S`, and we need a `struct S`) or the type is infinitely recursive (eg. `struct A {struct B b;}; struct B {struct A a;};`). This means we could end up analyzing the entire codebase to determine a the size/alignment of a `struct`, but in practice this should not happen often (regular C also requires the type definitions of every field, so we are arguably not much worse).

Pointers and references are boxed values, so we know its size/alignment (so no need to analyze the underlying type). However, we still require modules to import the symbol in order to handle name disambiguation. Arrays may be parameterized by an integer whose value must be immediately available. This is important, as otherwise we can end up with cases such as `struct S {struct T t[sizeof(struct U)];};`, which may be difficult to resolve. In our case, we can simply leverage compile-time reflection to obtain the integer value, similar to the way we resolve field types.

#### Functions

Functions have the form `returnType name(parameterType, ...numberOfParameters) {...body}`. In order for importers to use this function, we need to provide the size, alignment and name of any types in its declaration. We use compile-time reflection to resolve the details (see Types section for details).

If we are exporting an inline function, we also need to provide the function body to the compiler/optimizer (though from the perspective of the module, symbol visibility is the same whether inline or non-inline). This also uses compile-time reflection in order to resolve the symbols within the function body (which is done within the context of the containing module, not the importing module).

*Cforall introduces a keyword called `forall`, which can be used to create polymorphic functions. This is currently implemented using a technique called "dictionary passing", similar to Haskell or Ocaml. This means there is only one instance of the polymorphic function, and callers need to pass any trait information as additional (hidden) parameters. This means we must provide trait information to the importer. Within a `forall` block we can also define polymorphic types, which can be used as the return value of a polymorphic function. In that case we need size/align/name information, and the first two are currently handled by calling a "layout function" at runtime (this could possibly be moved to happening at compile-time, but I don't believe it is). As such, we also need to provide the layout function to the importing module. So polymorphic functions can be made to work with our system, but we may need to relax some aspects of our guiding principles.*

#### Variables

Variables have the form `variableType name = expression;`. We make the simplifying assumption that only one name is defined at a time (ie. no `int a, b, c;`). Similar to types and functions, the exporting module needs to also provide the size, alignment and name of `variableType` to any importing modules, which is handled by compile-time reflection (see Types section for details).

*If the variable is actually a polymorphic type (see Function section for details), I'm actually not sure how this works. The layout function can only be called at runtime, but you need the information in order to allocate the appropriate amount of space in the `.data` section. More analysis is needed here.*

### How importing modules use symbols

The previous section notes compile-time reflection as the solution to many of our challenges with modules. Here we will explain how symbol names are resolved to the correct modules, helping us define how our compile-time reflection crawls through modules. See Formalism section for a more rigorous proof of correctness.

#### Gathering symbols

First, we scan the top-level declarations of a module, gathering a list of imported modules and top-level symbol names defined within the module.

```
// module data/graph/node
module;
import edge;
import ../minheap;
import stdlib;
const int maxEdges = 10;
export struct Node {
    int id;
    int value;
    struct Edge edges[maxEdges];
    struct MinHeap someField;
};
// Shadows imported MinHeap
struct MinHeap {int capacity; int size; int *data;};
struct MinHeap initHeap(int capacity) {
    // Assumes malloc doesn't return error code
    return {capacity, 0, (int*)malloc(sizeof(int) * capacity)};
}
void destroyHeap(struct MinHeap* h) {
    free(h->data);
    h.capacity = 0;
    h.size = 0;
}
```

We first look for the import relative to the current directory - if that fails we look at libraries. So our set of imports here are: `data/graph/edge`, `data/minheap` and `stdlib` (library module).

The top-level symbol names are: `maxEdges`, `Node`, `MinHeap`, `initHeap` and `destroyHeap`.

#### Crawling other modules

Continuing with our example, we do the same top-level analysis with the imported modules.

If `data/minheap` has `export import minheap/functions;` defined, then would be as if `data/graph/node` has `import ../minheap/functions;` defined. This can keep going - if `data/minheap/functions` has `export import`, it is also added to `data/graph/node`. After we run out of `export import` statements to resolve, we can look at the top-level declarations of every imported module in order to determine what symbols are visible to `data/graph/node`.

After gathering the symbols in the imported modules, we can proceed to bind each symbol within `data/graph/node` to the correct module. The details of how multiple symbols are disambiguated are described in the Resolving symbols section.

In addition to name disambiguation, some symbols need additional information in order to be useable by importers. For example, size/alignment information of types, function bodies for inline functions, and trait information for polymorphic functions. This information is obtained by resolving the symbols on any imported module, and so on, as necessary.

This task is recursive, which raises the problem of circular imports: What if we recurse back to `data/graph/node` (or any module that creates a cycle)? Since we reason at the level of symbol definitions, as long as we are analyzing different symbols inside the circularly imported module, we don't actually have a cycle. This leaves us with handling the problem where we circle back to the same symbol. For size/alignment analysis, coming back to the same type means that said type contains itself, which for our purposes is not resolvable (emit error and stop). If an inline function calls other inline function that mutually recurses with itself, we produce a forward declaration of the inline function within the underlying C code (Cforall compiles down to C). For trait information, a trait works like a collection of conditions, which means it includes itself, which means we can safely ignore circular references (we may want to emit a warning though). Since we can handle all of our circular problems, our system is well-defined here.

#### Resolving symbols

A key aspect about our modules is that imported symbols are used in the same way as they are defined - no namespacing needed (eg. `struct Edge` in the provided example). While this raises the potential for name collisions, we argue that the potential benefits outweigh the limitations (and many of the limitations can be mitigated with additional features).

*To provide additional context, a core feature of Cforall is its advanced overloading system (eg. allowing multiple variables with the same name but different types to coexist). Adding namespaces would limit our ability to research and take advantage of this feature.*

When resolving symbols, symbols defined within our module have precedence over imported symbols. If we want to resolve to a specific imported symbol, we use the `::` operator (eg. `struct a::S` or `struct "a"::S`). The module name on the left side of the `::` operator is looked up using the same mechanism as import statements (note that the module being looked up must have been imported, either directly or through `export import`).

*In Cforall, the addition of the overload system means that symbols defined within our module system only have preference over imported symbols, instead of completely shadowing them. Here, our module system provides a list of candidates for each symbol to the overload resolution system. This will require refactoring the overload resolution algorithm to hold the full name of a symbol (symbol name + module name), and favour symbols defined within the given module (eg. `struct MinHeap` in the provided example).*

## Formalism

Taking the Walkthrough section a step further, we will provide a rough sketch of a formal argument that our module system is well-defined.

We'll first describe the layout of our module system by providing a rough description of the file structure and grammar we are working with. Then we describe what we can grab from each module file separately. By combining this information with those from other modules, we can resolve which module's symbol is being used in each exported declaration. Note that for some symbols to be properly used by importers, additional information such as size/alignment information will be necessary, which we will show is possible to obtain. Finally, we can resolve every symbol within a function body or expression, thereby completing the compilation process.

### Layout of the codebase

In order to provide a formal argument that our module system is sound, we need to first describe the format of the codebase we are trying to analyze.

The file system is set up so that imports are relative to the current directory, or found in a library. For example, a module file `data/graph.cfa` trying to access `data/graph/node.cfa` would write `import graph/node;`. If such a file doesn't exist, we will look for `graph/node` within the provided libraries (starting from the root directory of each library).

Module files have `module;` as their first statement. Import statements are written as `import filePath;`, where the file path can be escaped with double-quotes if necessary. Only top-level statements can be exported, done by having `export` as the first keyword (eg. `export const int i = 3;`). The available top-level statements are:
* Imports: Format is `import path/to/file;` or `import "path to/file";` (double quotes for escaping characters).
* Types: `struct` and `typedef` statements. Only one `struct` can be defined per top-level statement (eg. no `typedef struct {int i;} MyStruct;`). If we parameterize an array with an integer, it must either be a constant or a single variable.
* Functions: Format is `returnType name(parameterType ...numberOfParameters) {...functionBody}`.
* Variables: Format is `variableType name = initializerExpression;`. Initializer may be omitted. Only one variable can be declared per top-level statement (eg. no `int a, b;`).

The syntax of function bodies and initializer expressions can mostly remain unrestricted, as the module system provides name matches to them, rather than the module system being affected by anything within. One notable change is the `::` operator, which allows us to disambiguate which module a symbol belongs to. For example, if `struct S` is exported from module `a` and `b` and we import from both, we can write `struct a::S` or `struct "a"::S` to disambiguate.

### Individual module information

For every module file, we can collect the import list and the top-level symbols, without needing to consult any other files. The top-level symbols include types, functions and variables.

Top-level statements, at least the way we have it set up here, avoid any context-sensitive sections (eg. `MyType * a;` could be interpreted as an expression, but we don't allow those as a top-level statement). This means we have no issues parsing all symbols at the top-level (function bodies and initializer expressions are context-sensitive, but at this stage we don't need to look inside them). This means that for each module file, we can grab an import list, a type list, a function list and a (global) variable list, as well as determine which are exported.

### Resolving names of exported declarations

Now we wish to resolve every symbol in any module's top-level declaration (this does not include the function bodies or initializer expressions). We achieve this with the following steps:
1. Collect a module's full import list
2. Use the full import list to collect all imported symbols
3. Use all imported symbols to resolve our symbols

In order to determine a module's full import list, we look at the individual module information of each imported module. To find the imported module file, we initially look for it relative to the current directory of the module that the import statement is in - if the module file does not exist, we look starting at the root directory of any provided libraries (if we can't find the module, we error out). If any imported module has `export import moduleName;` (or a name other than `moduleName`), then we recursively examine those too. If we end up importing any modules we have already imported (or the module we are getting the full import list of), we can ignore those. Since there are a finite number of modules in a codebase (plus any provided libraries) and we don't analyze a module twice, this search will terminate with a full import list.

To collect all the imported symbols, we take every module in the full import list and we grab all of the top-level statements that have `export` as their first keyword. This gives us an imported type list, an imported function list and an imported variable list.

Now that we have a list of imported symbols as well as the module's own, we can start resolving symbols in the top-level declarations. Since we are not considering function bodies or initializer expressions (and our top-level statements are simple), we do not have ambiguity on whether a type, function or variable is needed. To resolve names, we first check to see if it uses the `::` operator - if so, then we use the same strategy as when resolving import statements (also, we error out if the module isn't in the full import list). Otherwise we look in the module's own top-level symbols to see if we have a match (this means a module's symbols shadow its imported symbols). If we don't find a match, then we look in the list of imported symbols. If the list of imported symbols has more than one match, we error out due to ambiguity (print diagnostic telling user the options and suggesting the use of `::`).

Something to note about this setup is that forward declarations are neither necessary nor allowed, because all visible symbols are known before attempting to resolve names.

### Providing additional information for importers

Unfortunately, in order to use some of these top-level symbols, the importer sometimes needs more than the names within top-level declarations (ie. not including function bodies or initializer expressions) in order for the compiler to process it. For example, if a function returns a `struct`, we need to know the size/alignment of the `struct` in order to use it.

Type names are found within the definitions of other types, the return and parameters of a function, and the type of a variable. In order to use a type, we must know the size/alignment of its fields. Similar problem with functions and their return and parameter types. With variables, size/alignment information is technically not needed, but to keep its semantics consistent with everything else, we require it (eg. `struct Unimported x = s.f1; struct Unimported x1 = importedVariable;`).

In order to figure out size/alignment of types, we first resolve its name. Pointers, references and built-in types have a defined size/alignment, and don't need to be analyzed further. However, an array type needs its element type analyzed further, as well as its integer parameter (our formalism only allows a simple number, but we could extend this in the future). That leaves `struct` or `typedef`, which is determined by looking at the types of its fields or aliased type. This process keeps recursing until we resolve everything or find a cycle (if we have a cycle, we error out). This terminates because there are a finite number of symbols. This means we can obtain the size/alignment of any well-formed type.

While the importing module doesn't change its behaviour whether a function is inline or not, the compiler/optimizer must see the function body. This means the module system must resolve all symbols that the inline function body can access. For example, if we import a module that contains an inline function, we need to resolve all symbols that that module can see. And if any of its imported symbols are inline functions, then we need to keep recursing. If we find a mututally recursive inline function, we would emit a forward declaration (eg. `inline float a();`). Note that the module that owns the inline function must use `extern` when compiling it (eg. `extern inline float a() {return 5;}`), to ensure a definition is emitted in the assembly. This terminates because there are a finite number of modules.

*While the technique used for importing inline functions is sound, it can be optimized (we can resolve symbols within the inline function body first, instead of immediately expanding every import). Here, a single imported inline function forces the containing module to expose its contents to the optimizer. It's as if every import statement within the containing module was converted into `export import`. While our module system ensures these `export import` changes are not visible to the module importing the inline function, it is visible to the optimizer. My major question is: if I add an unused exported inline function to any module, will the additional information provided to the optimizer affect code performance significantly? My intuition says no, since none of it should be accessed by the module that imported the inline function, though I would need to double check.*

*Cforall introduces the `forall` keyword, which allows creating polymorphic functions and types. The polymorphic functions are currently implemented using a technique called "dictionary passing", similar to Haskell or Ocaml. This means there is only one instance of the polymorphic function, and callers need to pass any trait information as additional (hidden) parameters to it. This means that we need to provide the trait information of each polymorphic argument to the importer. In order to obtain this trait information, we could follow a similar technique to obtaining the size/alignment of types, except that we're working to define a trait rather than a type. As such, if a trait mutually includes another trait, we can ignore the circular reference. I have also heard that there are "layout functions" that need to be provided for polymorphic types (since they can be of different sizes/alignments), so we would need to find those functions too.*

### Resolving names for compilation

Now that all symbols visible to a module have been collected, we can proceed to resolve all symbols within function bodies or initializer expressions. Technically, the additional information needed to use them isn't required, though it simplifies our proof and implementation.

We resolve names in the same manner as described in the section Resolving names of exported declarations. This means that symbols in the current module shadow those that are imported, though the imported symbols can still be accessed using the `::` operator.

*In Cforall, the addition of the overload system means that symbols defined within our module system only have preference over imported symbols, instead of completely shadowing them. This works by having our module system provide a list of candidates for each symbol to the overload resolution system. The overload resolution algorithm will need to be refactored to hold the full name of a symbol (symbol name + module name), and favour symbols defined within the given module (eg. `struct MinHeap` in the provided example). Whether favour means "break ties" or "always use current module's symbols if possible" is a design choice (I prefer the latter).*

## Concepts

A module is a collection of symbol definitions, which it can export to other modules. In order to define said symbol definitions, it may import other modules' symbols. We consider 3 kinds of symbols: variables, functions and types.

*Cforall introduces other kinds of symbols, such as traits. The C language as well as GCC extensions also support a significantly more complex grammar than what is used in this proposal. I discuss some more about this in Other kinds of symbols section.*

Our proposed module system is one instance within the space of all viable module systems, created from a number of design choices and solutions to challenging problems. We will highlight a number of them, and discuss how our module system handles them. We will also describe a number of functionalities in our system 

### Handling unboxed types

In the example below, we note a problem: in order to use `B::foobar` and `B:bb` in C, we also need to provide the definition of `A::a`. This breaks our principle of decoupled symbol exports!

```
                 Module B
               --------------------------------------
               | module;                            |
      imports  | import A;                          |  exports
A::a --------> | export int b = a;                  | --------> B::b
A::aa          | export struct aa foobar();         |           B::foobar (+ A::aa)
               | export struct bb {struct aa foo;}; |           B::bb (+ A::aa)
               --------------------------------------
```

This problem arises because the importer needs to know how much space to allocate on the stack for the type. Languages that use boxed types (eg. Java, Go) do not have this restriction, since the caller only needs to store a pointer on the stack (eg. when calling `B::foobar`). In contrast, C and other systems programming languages (eg. C++, Rust) require handling unboxed types, since it demands control over pointer indirection (also, there may not be a heap available to support boxed types). This means we need to know the definitions of types in order to use them, even if it's just to allocate enough space on the stack. In C/C++, this is done by including transitive dependencies in headers; in Rust, this is done by compiling an entire crate at a time.

It turns out that the importer need not know the definition of `A::aa`, however. **We only need to provide the size and alignment of a type in order to store it (in addition to the name).** This means that if we knew that `A::aa` is 32 bytes in size, aligned to 8 bytes, the caller of `B::foobar` has enough information to output working assembly. This allows any importer of `B::foobar` to assign the output of `B::foobar` to a variable, then pass it by value into a function. However, the importer cannot access the fields of `B::foobar` or change them unless it imports `A::aa`, giving us exactly the symbol decoupling we want.

An interesting property of this is that, depending on the ABI (Application Binary Interface, determines exactly how things like functions are called in assembly), if `A::a` changes but does not exceed its size or alignment allocations, any module that imports `B::foobar` but not `A::a` does not need to be recompiled! This could lead to libraries where user-facing `struct` types are set to be larger than they currently are, so that even if the `struct` needs its definition changed, the user's code may not even need to be recompiled!

How do we provide the size and alignment of a type, though? A couple strategies come to mind:
1. We provide the entire type definition to the compiler, but anything within the module is not allowed to access its fields unless it is imported. Much simpler to implement, but we might need to crawl the codebase and a lot of additional code needs to be provided to the compiler.
2. We extract size/alignment information into a separate generated module interface, which is used in place of the actual type. This provides much more succinct code to the compiler. We still need to crawl the codebase to get the information, but it is cached afterwards. One way to calculate size/alignment information is to generate a separate program with all the types, compile it, and inspect the result. Another way is to collect the sizes of the built-in types, then calculate the size/alignments of types from the idea that `struct` types take the maximum alignment of their fields.
3. We store all size/alignment information in some central file. This is the same as the previous idea, except that we store all size/alignment in a central file instead of generating a module interface file for every module. This could help reduce file IO cost and simplify updating size/alignment information when types change.

Option 3 sounds like the most ideal option, but option 1 is the easiest to implement, and is the configuration that is used in the Formalism section.

*Cforall has the sized trait, which may prove useful for me to use when implementing things (it will likely help with handling polymorphic functions and types). I was also made aware of lifetime operations in Cforall, which I am not that familiar with. I assume they are handled the same way as move/copy constructors - they also need to be provided alongside the size/alignment information.*

### C macros are not exportable

**In our proposal, the C preprocessor runs before the module system.** This allows us to write macros that auto-generate symbols to export, which is likely an important C pattern. However, this makes it very challenging to export C macros.

It is worth pointing out that C++20 modules also do not support exporting C macros. A problem with C macros is that they are too powerful - they can make arbitrary changes to your code, forcing you to reparse, and they operate using a separate language that ignores C code structure. With modern C++ and Cforall features, the need to have modules export C macros is usually not needed outside of porting existing C code.

What if you are dead set on making modules export C macros? You could make the C preprocessor module-aware by using directives such as `#pragma module` and `#pragma export MY_VALUE 3`. Unfortunately, due to having to work within the confines of the C preprocessor, you'd likely be stuck with having to settle with a simple module system. A better alternative would be to have the C preprocessor to output all C macros that it encountered while preprocessing a file, then collect all import statements within the file. We could then reparse the file using all of the C macros from the other files. This configuration is still somewhat unideal, but it does satisfy some common use-cases, and could be used as a starting point. Note that if you're really, really dead set on having the exact same behaviour, just use precompiled headers - they're an optimization that ensure the same behaviour.

The other problem with supporting C macros is that they introduce a kind of ordering on `#include` statements, which we are trying to avoid with imports. We also don't wish to promote the use of C macros when other modern language features are often a much better fit. As such, we elect not to support exporting C macros out of modules.

But what about porting existing C code? With our current formulation, we don't provide a way to use C headers within our modules. This problem is complex enough that we will pull it out into a dedicated section (see Porting existing code section).

*The rest of this proposal is not yet ready for public consumption.*
