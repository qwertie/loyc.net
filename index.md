---
layout: default
title: Home
---

# <center>Loyc: Language conversion & interoperability</center> #

Loyc is a label for standards, code libraries, and programming tools that 

- Are programming-language **agnostic** (universal or multi-language),
- Help programs and programming languages to **interoperate** (talk to each other),
- Help **convert** code between programming languages, or
- Are built out of other Loyc tools and libraries (e.g. a "Loyc parser" converts text to a Loyc tree.)

We believe that programming languages should interoperate with each other on a high level and share as much technology and terminology as possible. Poor interoperability is a serious problem today, and we want to fix that.

We also believe interoperability is a serious problem _within_ individual languages, because of a lack of standardization of data types and interfaces, and in some cases, standard types being poorly-designed: inefficient, limited, etc. We want to fix that too, but haven't really gotten started on it yet.

The original idea of Loyc was a multi-syntax compiler, which would have taken code written in several languages and compile it as a single program with a single compiler, allowing code to call between languages and share data, easily and directly. We have not had the necessary volunteers or resources to create such a compiler; instead we working on bits and pieces that may be useful for such a compiler eventually.

In general, the Loyc initiative is about finding ways to bring the world's programs and programming languages closer together, to grease the wheels of developers everywhere.

Currently, all Loyc projects are written in C# and [Enhanced C#](http://ecsharp.net) because the .NET platform is the only one specifically designed for multiple languages. Eventually though, we hope to shift our focus to WebAssembly, especially since Microsoft has not been fixing the CLR's [design flaws](http://loyc.net/2014/dotnet-annoyances.html).

Core projects
-------------

### Loyc trees ###

<div class="sidebox" markdown="1">![](/loyc-trees/loyc-tree-diagram-sm.png)
</div>

[Loyc trees](/loyc-trees) are a general-purpose tree structure designed to represent source code. They are comparable to Lisp syntax trees, with just enough added complexity to represent programming languages from the Algol family, such as C#.

### LES ###

[LES](/les) is a syntax/data interchange format for Loyc trees, designed to resemble other languages in the C family, such as Javascript.

### LeMP ###

[LeMP](http://ecsharp.net/lemp) is a programming-language agnostic Lisp-style lexical macro processor.

## Flame ##

[Flame](http://github.com/jonathanvdc/flame) is a set of "middle-end" components designed to help write managed (C#-like) compilers. It has is own IR (separate from Loyc trees) which it can optimize, and it supports static linking of Flame IR files. The CLR backend is the only stable one; a WebAssembly backend is in early stages.

### Loyc Core Libraries for .NET ###

The [Loyc Core Libraries](http://core.loyc.net) are libraries of general-purpose stuff, most of which fits the theme "things that should be built into the .NET framework, but aren't". These libraries also include the LES parser and the standard base classes for LLLPG parsers.

Projects built on Loyc technology
---------------------------------

### LLLPG ###

The [Loyc LL(k) Parser Generator](http://ecsharp.net/lllpg) is a parser generator designed to help you write parsers with performance similar to hand-written parsers. The parsers for LES and Enhanced C# use LLLPG.

### Enhanced C# ###

[Enhanced C#](http://ecsharp.net/) is a programming language that is backward-compatible with C#. It's nowhere near done, but you can add Enhanced C# code to any vanilla C# project via the LeMP Custom Tool for Visual Studio.

Future core projects
--------------------

### API standardization projects ####

Interoperability within a single language is often limited. For example, there's no standard for how to represent "points" and "vectors" (X-Y and X-Y-Z groups) in C++. Typically, two geometric libraries that both operate on "points" and/or "vectors" are not directly compatible and require conversion steps between them. Even strings in C++ aren't standardized: `wchar_t` and `wstring` is UTF-16 on Windows and UCS-4 on Linux.

Even when something is standardized, it may be misdesigned and deserves a new and better design. For example, in the .NET BCL, points are not treated as general concepts but as GUI-specific concepts. The types `Point` (int coordinates) and `PointF` (float coordinates) are defined in the namespace for GDI+ (Windows Forms), while WPF has its own `Point` type with `double` coordinates and a somewhat different API. All these APIs are limited in some ways.

So we would like to start a movement to create standard APIs that attempt to simultaneously achieve 

1. high performance (e.g. A 2D or 3D `Point` should be a value type with coordinates stored directly in the structure, rather than as a heap-allocated array)
2. Type safety (e.g. separating the concepts of `Vector` and `Point` even though they are physically identical)
3. Ease-of-use (e.g. taking advantage of operator overloading: `(p - q) * 2`)
4. Generality (e.g. `Point<Coord>` rather than baking in a particular coordinate type, and somehow generalizing points to N dimensions)
5. Interoperability (e.g. a point or vector should implement interfaces that let it act like a list of numbers)

### Multi-Lanuage Standard Library ####

The [MLSL](http://loyc.net/2014/design-elements-of-mlsl.html) would form the basis of a system for writing code that can be converted automatically to many languages.

### Standard Imperative Language ###

When converting code from one programming language to another, [SIL](https://github.com/qwertie/Loyc/wiki/Standard-Imperative-Language) is the concept of a schema for Loyc trees that would lie at the midway point between two languages. SIL has not been designed yet; the plan is to base it on conventions created by WebAssembly and by Enhanced C#.

### The Ultimate Programming Language ###

Humans have puzzled over the question of how programming languages should work for about 65 years, and explored innumerable solutions. Now is the time for consolidation: to create a language in which you can quickly become comfortable, whether you're used to Java, C#, C++, Javascript, Go, Lisp, OCaml, Erlang, Julia or Python.

The [ultimate programming language](http://loyc.net/2015/ultimate-language.html) would be designed for all purposes:

- High performance (C-like)
- Scripting and prototyping
- Powerful, flexible, easy to use genericity
- Metaprogrammable
- Extensible by end-users, perhaps even with type system extensibility
- Many-paradigm (functional, object-oriented, and generic, declarative/goal-driven, aspect-oriented, matrix-oriented)
- IDE-friendly (code completion is assisted by syntactic design and by minimizing the cost of gathering information about a program's structure.)
- Good for parallel, distributed, and constrained/IoT computing
- A reasonably large subset of the language should be suitable for automatic conversion to many other languages
- Could support multiple unrelated syntax "styles" (e.g. C# + Swift code) and "skins" (e.g. a growable array whose API is "skinned" to work like C++ `std::vector<T>`) to aid migration from other languages

Obviously, Loyc would need a lot of funding to do this, and it currently has - let's see now - zero.

Related projects
----------------

We'd like to list other projects here related to interoperability, API standardization and code conversion, even if they are unaffiliated with Loyc. Do you think a project should be listed here? You can [make it an issue](https://github.com/qwertie/loyc.net/issues) or reach me at `gmail.com`, with account name `qwertie256`. All standards and tools listed here have an open source implementation.

- Cross-language interoperability projects
    - [SWIG](http://www.swig.org/): use C/C++ libraries from almost any other language
- General-purpose data formats
    - [JSON](http://www.json.org/): a very simple interchange format for the world's most common data types
    - [Protocol buffers](https://developers.google.com/protocol-buffers/): a standard for binary data storage that tolerates changes between versions.
    - [Cap'n proto](https://capnproto.org/)
- Other important standards
    - [Language server protocol](https://github.com/Microsoft/language-server-protocol) provides interoperability between an IDE and a code-completion service written in any language. Initial users include [Rust Language Server](https://internals.rust-lang.org/t/introducing-rust-language-server-source-release/4209) and [VS Code](https://code.visualstudio.com/Docs/extensions/example-language-server).
    - [HTTP/1.1](https://www.w3.org/Protocols/rfc2616/rfc2616.html) and [HTTP/1.0](http://www8.org/w8-papers/5c-protocols/key/key.html): rudimentary HTTP servers are relatively easy to write and can talk to web browsers and other standard tools, making HTTP the most popular standard built on [TCP/IP](https://en.wikipedia.org/wiki/Internet_protocol_suite).
    - [WebAssembly](http://github.com/webassembly/design): the new standard virtual machine for web browsers. Provides "write once run anywhere" but lacks interoperability features.

Help wanted
-----------

We need help to

- Design general-purpose APIs with high performance, type safety, ease-of-use, generality and interoperability
- Design a Multi-Language Standard Library and Standard Imperative Language
- Write parsers/printers to parse a programming language into a Loyc tree or the reverse (print a Loyc tree as text)
- Write an LES parser for a language of your choice

Currently, we are severely undermanned and have no sponsors. If you'd like to help, [make an issue](https://github.com/qwertie/loyc.net/issues) or reach me at `gmail.com`, with account name `qwertie256`.
