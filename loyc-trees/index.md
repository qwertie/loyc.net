---
layout: page
title: Loyc trees
date: Edited 5 Nov 2016
toc: true
tagline: The universal syntax tree
---

Introduction
------------

Loyc trees are a kind of inverted tree structure. They are designed for representing source code, but are also useful for data in general, particularly if that data includes code-like elements such as "expressions", "commands" or "variables".

The term "Loyc tree" refers to the conceptual or in-memory representation of the tree; the standard text representation of Loyc trees is [LES](/les).

The goals of LES and Loyc trees are:

- To define a neutral standard syntax for expressions, for use in diverse applications

- To provide a common language for humans to talk about syntactic structure

- To help jump-start development of programming languages and DSLs by eliminating the need to design a new syntax or write a new parser

- To assist tools that convert code between different languages, by providing a way to change a tree incrementally from one language until it conforms to the rules of another language.

- To provide a textual representation for compilers' intermediate output at various stages.

- To form the foundation of "universal" tools that can manipulate the syntax tree of any language that can be converted to and from Loyc trees. For example, [LeMP](http://ecsharp.net/lemp) is a tool that, in principle, can add a LISP-style macro processor to any programming language.

Definition
----------

A Loyc tree, also known as a Loyc node, is one of three things:

* An identifier, such as the name of a variable, type, function, method, operator or keyword.
* A literal, such as an integer, double, string or character.
* A "call", which represents either a method call, or a construct like a "class" or a "for loop".

Unlike in most programming languages, Loyc identifiers can be any string--any string at all. Even identifiers like `\n\0` (a linefeed and a null character) are supported. This design guarantees that a Loyc tree can represent an identifier from any programming language on Earth. To support WebAssembly, LESv3 takes this a step further, supporting not just _any string_ but _any sequence of bytes_.

Normally, literals can be any value representable in the language in which Loyc trees being used, but this is only true of Loyc trees that exists in memory. When a Loyc tree is serialized to text, some in-memory data may not be representable, although LESv3 has a system for defining "custom" literals.

Each Loyc node also has a list of "attributes" (usually empty), and each attribute is itself a Loyc tree. Loyc nodes also contain position information to keep track of the location within a source file where the node was stored.

In other words, a Loyc tree (or Loyc "node") is a data structure with these properties (potential parts):

1. `Range`: tuple of (source file name, integer position, integer length)
2. `Attrs`: a list of attributes (metadata)
3. One of: a literal (`Value`), an identifier (`Name`), or a call (`Target` and `Args`).

Call nodes are the most interesting. A "call" represents either a method call, or a construct like a "class" or a "for loop". By convention, constructs that are built into a language use a special identifier that starts with `#` or `.` or `'`, such as `.class` or `#public` or `'==`. By convention, then, `foo(x, y)` (where `Target` is `foo` and `Args` is a list of two items) would be a normal function call, while `#foo(x, y)` would represent some kind of special construct, e.g. `#var(Foo, x)` could represent a declaration for a variable `x` of type `Foo`. **Note:** these naming conventions aren't quite settled yet.

![](loyc-tree-diagram.png)

Loyc trees versus s-expressions
-------------------------------

Loyc trees are inspired by LISP trees, but designed for non-LISP languages. If you've heard of [LISP](http://en.wikipedia.org/wiki/Lisp_(programming_language)), well, Loyc Expression Syntax (LES) is basically a 21st century version of the [S-expression](http://en.wikipedia.org/wiki/S-expressions). The main differences between s-expressions and Loyc trees are:

* Loyc trees are a data structure, whereas s-expressions are a _syntax_ whose underlying data structure _could_ be a singly-linked list. LES is to Loyc trees as s-expressions are to singly-linked lists.
* Each "call" has a "target". Whereas LISP represents a method call with `(method arg1 arg2)`, Loyc represents a method call with `method(arg1, arg2)`. In LISP, the method name is simply the first item in a list, and there is no way to tell if `(x y z)` is a list of three items or a function call that takes two arguments. In contrast, most other programming languages separate the "target" from the argument list, with the notation `target(arg1, arg2)`. In a Loyc tree, a specific target would be used to denote a list (e.g `#tuple(item1, item2)`).
* Each node has a list of attributes. The concept of attributes was inspired by .NET attributes, so naturally a .NET attribute or Java annotation would be represented in a Loyc tree by an attribute. But also, "trivia" such as comments and blank lines can be represented by attaching attributes to nodes, and modifiers like "public", "private", "override" and "static" are conventionally represented by attributes.
* Each node has an associated source file and two integers that identify the range of characters that the original construct occupied in source code. If a Loyc tree is created programmatically, a dummy source file and a zero-length range can be used.

Loyc trees: text representation
-------------------------------

Obviously, a text format is needed for Loyc trees.

My original plan was to use a subset of [Enhanced C#](http://ecsharp.net) to represent Loyc trees. However, since EC# is based on C#, it inherits some very strange syntax elements. Consider the fact that `(a<b>.c)(x)` is classified a "cast" while `(a<b>+c)(x)` is classified as a method call. EC# has a lot of oddball rules like that, which create unnecessary complication that should not exist in an AST interchange format.

Threfore I invented ([Loyc Expression Syntax](/les)). However, we can do better than _just_ an interchange format; I've made LES flexible enough to be used as a modern programming language in its own right.

Here is a simple Loyc tree expressed in LES version 2:

    #if(c < 0, Print(@[en] "negative"), Print(@[en] "non-negative"));

The top-level loyc tree calls the identifier `#if`. `#` is a standard identifier character; it is treated no differently than a letter or an underscore, but by convention it marks an identifier as being somehow "special". `#if` is "called" with three arguments (we say "called" for lack of a better word, but probably `#if` would be a built-in construct, not a function). `c < 0` is also a call. `c < 0` calls the identifier `'<` with two arguments, "c" and "0". The strings each have an attribute attached, which is an identifier called `en`.

I cannot say what this statement "means" in LES. It doesn't have a meaning; LES is merely a data structure, not a programming language, so constructs in LES have no inherent meaning. Just as `<IF>` has no predefined meaning in XML, `#if` has no predefined meaning in LES.

However, the statement does have meaning in [EC#](http://ecsharp.net). In fact, it is equivalent to a standard "if" statement:

~~~C#
if (c < 0)
   Print([en] "negative");
else
   Print([en] "non-negative");
~~~

Again, `[en]` is an attribute. Whereas plain C# allows attributes only on declarations such as fields and classes, EC# allows attributes on any expression. Attributes are sometimes used to provide extra information to macros (compiler extensions) at compile-time; otherwise they are meaningless and the compiler should produce a compiler warning about their uselessness.

In this case, one could imagine writing a compiler extension that helps do internationalization. You could define `[en]` to mean that the text is in English and needs to be translated to all other supported languages. Again, that's not something that EC# will support directly--it's a feature somebody might add. (Note: I'd probably support translations in a different way, using an attribute on the function being called rather than at the call site. But both approaches might be useful.)

Please see the [LES](/les) and [EC#](http://ecsharp.net) pages for more information.

Single-list perspective
-----------------------

In languages that use 0-based indexing, the argument list is typically numbered from `0` to `ArgumentCount - 1`, so if `N` is a call node, `N[0]` refers to its first argument, `N[1]` refers to its second argument, and so forth.

Sometimes it is useful to view a node as having a single list of children. You can think of *all* nodes as having a single contiguous list of child nodes indexed starting from `-AttributeCount - 1` to `ArgumentCount - 1`. That is, if `N` is a Loyc tree node then `N[-2]` refers to its final attribute, `N[-1]` refers to its target, `N[0]` refers to its first argument, and so forth. If a node is an identifier or literal then `N[-1]` (and above) does not exist, but `N[-2]` and below may still exist. In .NET ([`LNode`](http://ecsharp.net/doc/code/classLoyc_1_1Syntax_1_1LNode.html)), `N.Min` and `N.Max` tell you the range of valid indexes (if a node `N` has no children, `N.Min == -1 && N.Max == -2`.)

Node styles and trivia attributes
---------------------------------

The C# implementation of Loyc trees has a concept of "node style", an 8-bit number that represents something stylistic and non-semantic about the source code. For example, `0xC` and `12` are the same integer in two different styles. It is semantically the same—the compiler always produces the same program regardless of which form you choose. But it's a striking visual difference that should be preserved during conversion between languages. In my implementation, this difference is preserved in a node's `NodeStyle` property, using the bit flag `NodeStyle.HexLiteral`.

**Style bits are an unnecessary flourish**; an implementation of Loyc trees can use trivia attributes instead, but the concept of style bits may save memory.

A trivia attribute is a Loyc node in an attribute list whose `Name` starts with `#trivia_`. Trivia attributes can be simple identifiers or calls. By convention, trivia attributes have low importance and can be optionally dropped when converting a Loyc tree to text.

Probably the most important use of trivia attributes is to denote comments. By convention, comments like

~~~C#
// Before
result = /* in the middle */ Func(); // after
~~~

are represented by the following Loyc tree:

~~~
@[#trivia_SLComment(" Before")]
@[#trivia_trailing(#trivia_SLComment(" after"))]
result = (@[#trivia_MLComment(" in the middle ")] Func());
~~~

If you manually insert a `#trivia_` attribute in your source code, it may disappear or change form when the code is printed out (it affects the output in some special way if the printer understands it, as with comments.)

Mappings between Loyc trees and programming languages
-----------------------------------------------------

It is necessary to standardize the Loyc trees that are used to represent code in a particular language, or there will be confusion and less interoperability.

For C# I chose a Loyc tree representation that closely mimics the original source code. Here are some examples:

| C# code                 | Loyc tree (LESv2 prefix notation) | Loyc tree (friendly notation in LESv2) |
|-------------------------|-----------------------------------|-----------------------------------|
| `if (c) A(); else B();` | `#if(c, A(), B())`                | N/A                               |
| `x = y + 1;`            | `@'=(x, @'+(y, 1));`              | `x = y + 1;`                      |
| `switch (c) { case '1': break; }` | ``#switch(c, @`'{}`(#case('1'), #break));`` | `#switch c { #case '1'; #break; }` |
| `public string name = "John Doe";` | `[#public] #var(#string, @'=(name, "John Doe"));` | `[#public] #var(#string, name = "John Doe");` |
| `int Square(int x) { return x*x; }` | ``#fn(#int32, Square, #(#var(#int32, x)), @`'{}`(#return(@'*(x, x))));`` | `#fn(#int32, Square, #(#var(#int32, x)) { return x * x; });` |
| `class Point { public int X, Y; }` | ``#class(Point, #(), @`'{}`(@[#public] #var(#int32, x, y)));`` | `#class(Point, #(), { @[#public] #var(#int32, x, y); };` |
| `class List<T> : IList<T> { }` | ``#class(#of(List,T), #(#of(IList,T)), @`'{}`());`` | `#class List!T #(IList!T) { };` |
| `x = (int)y;`           | `@'=(x, #cast(y, #int32));`       | `x = #cast(y, #int32);` |

As you can see, there's a clear and obvious relationship between the Loyc tree and the original source code (read [LES](/les) to understand the second notation better). Most keywords are represented by `#` plus the keyword name (I'm translating "int" as "#int32", however, which makes sense as a standard name common to all programming languages, or at least, all programming languages that support 32-bit integers.) Note: the prefix on operator names has changed to apostrophe, as [planned](http://loyc.net/2016/put-back-the-sharp.html).

Occasionally, it is not possible (or not ideal) to use the original keyword. For example, C# has two unrelated statements that are both called "using":

~~~C#
    using System.Collections;
    using (Foo()) { ... }
~~~

In this case I decided to use `#using` for the second statement but `#import` for the first. I could have used `#using(...)` for both (distinguished by the fact that one of them takes a single argument, and the other takes two), but that would cause trouble, as it would be easy to write code that accidentally treats one as if it were the other.

It would be nice to use a consistent mapping for each programming language, and where possible, to use the same or similar mappings in multiple programming languages. While I have chosen `#var(#int32, x, y = 0)` to represent `int x, y = 0`, it could just as easily be `#var(#int32, x, y(0))` or `#varDecl(x, #int32, y = 0, #int32)` or something else. Eventually an organization could be set up (perhaps with loyc.net as a home base) with a mandate to 

* regulate these mappings
* document these mappings
* offer "reference implementations" for parsing each language into a Loyc tree
* offer "reference implementations" for printing a Loyc tree to text in a given language

For now, I suggest these rough guidelines:

1. The Loyc tree should resemble the original code. For example, notice how `#var(Foo, x = -1)` resembles `Foo x = -1`, and `#fn(#void, f(), {})` resembles `void f() {}`.
2. The Loyc tree should be consistent between languages, if this is easy to achieve. An example of this is using `#int64` rather than `#long` in C# to represent a 64-bit integer. In the future we might define [Standard Imperative Language](https://github.com/qwertie/ecsharp/wiki/Standard-Imperative-Language) as an "anchor" for future mappings. If SIL contains a construct that is semantically identical to a construct in language X, then language X's mapping should use the SIL construct, rather than inventing a new construct that means the same thing. Sometimes this rule will override rule #1.
3. The tree should be easy to interpret after it is parsed. When I used `#import` to represent the `using` directive, I was favoring this rule over rule #1. On the other hand, I violated this rule slightly for variable declarations. Although the variable name is always stored in the second argument (or Nth argument for multi-variable declarations), you must check if the second argument calls `=` or not. If it does, the variable name is stored inside the call to `=`. This complication was a pain point, but I didn't see an ideal solution. But, unless you have a specific reason to violate this rule, try to ensure that interpreting the tree is easy.

These rules are sometimes in conflict, so if two people both try to define mappings they will inevitably make different decisions. That's why we'll need to standardize the mappings eventually.

Loyc tree implementations
-------------------------

The .NET implementation is the only one that exists as of October 2016.

See [Using Loyc trees in .NET](dotnet.html).
