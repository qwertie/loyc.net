---
layout: page
title: Loyc trees
date: Edited 14 Jun 2016
toc: true
---

Introduction
------------

A major part of my plans for Loyc is the concept of an "interchange format" for source code. In three words, the concept is "XML for code"--a general representation of syntax trees for any language.  Just to be clear, I do not advocate XML as an interchange format. In fact, I find XML's syntax very annoying (who among you likes typing `&lt;`?) Rather, I believe there should be a standard language for describing syntax trees, just as XML or JSON is used to describe data. In fact, the language I developed is useful for describing data too, as it is a superset of JSON (though this [might change](http://loyc.net/2016/potential-new-les.html) in version 3).

The interchange format will:

- Assist tools that convert code between different languages, by providing a way to change a tree incrementally from one language until it conforms to the rules of another language.

- Promote compile-time metaprogramming to the entire software industry. With Loyc trees, it is straightforward to add macro-based metaprogramming on top of any existing programming language. For example, [the LeMP](http://ecsharp.net/lemp) appears to be a macro processor for C# only, but in fact it could support any programming language for which a parser and printer has been written. LeMP can be compared to the C/C++ preprocessor: you _could_ use the C/C++ preprocessor with any programming language, but it's clunky, its usefulness is limited, and it integrates poorly into the host language. LeMP is a preprocessor too, but it operates on syntax trees rather than text, so it integrates much better into the "host" language, and its macros can run arbitrary code at compile time, so it is more powerful (also more dangerous, but I digress.)

- If a compiler uses Loyc trees internally, it can assist people writing compilers and compiler extensions by
 
    1. providing a simple textual representation of the compiler's intermediate output at various stages. Intermediate output will make it easier to see and discuss the processes going on inside a compiler. 
    2. via LeMP, allowing pattern matching on syntax (the [C# implementation is a proof of concept](http://ecsharp.net/lemp/lemp-code-gen-and-analysis.html)). 

The key concept for representing code from different programming languages is the Loyc tree. "Loyc tree" refers both to the conceptual structure that I invented, and to its _in-memory_ representation. An interchange format (typically [LES](/les) or [EC#](http://ecsharp.net/)) will allow Loyc trees to be converted to plain text and vice versa.

Definition
----------

A Loyc tree, also known as a Loyc node, is one of three things:

* An identifier, such as the name of a variable, type, method, operator or keyword.
* A literal, such as an integer, double, string or character.
* A "call", which represents either a method call, or a construct like a "class" or a "for loop".

Unlike in most programming languages, Loyc identifiers can be any string--any string at all. Even identifiers like `\n\0` (a linefeed and a null character) are supported. This design guarantees that a Loyc tree can represent an identifier from any programming language on Earth. Literals, similarly, can be anything, but when I say that I am referring to a Loyc tree that exists in memory. When a Loyc tree is serialized to text, obviously it will be limited to certain kinds of literals (depending on what is supported by the language used for serialization).

Each Loyc node also has a list of "attributes" (usually empty), and each attribute is itself a Loyc tree. Loyc nodes also contain position information (location within a source file).

In other words, a Loyc tree (or Loyc "node") is a data structure with these properties (potential parts):

1. `Range`: tuple of (source file name, integer position, integer length)
2. `Attrs`: a list of attributes (metadata)
3. One of: a literal (`Value`), an identifier (`Name`), or a call (`Target` plus `Args`).

Call nodes are the most interesting. A "call" represents either a method call, or a construct like a "class" or a "for loop". By convention, constructs that are built into a language use a special identifier that starts with `#`, such as `#class` or `#for` or `#public`. By convention, then, `foo(x, y)` (where `Target` is `foo` and `Args` is a list of two items) would be a normal function call, while `#foo(x, y)` would represent some kind of special construct, e.g. `#var(Foo, x)` could represent a declaration for a variable `x` of type `Foo`. However, for the sake of convenience, this convention about `#` may be broken when using LES directly as a programming language.

Loyc trees versus s-expressions
-------------------------------

Loyc trees are inspired by LISP trees, but designed for non-LISP languages. If you've heard of [LISP](http://en.wikipedia.org/wiki/Lisp_(programming_language)), well, Loyc Expression Syntax (LES) is basically a 21st century version of the [S-expression](http://en.wikipedia.org/wiki/S-expressions). The main differences between s-expressions and Loyc trees are:

* Loyc trees are a data structure, whereas s-expressions are a _syntax_ whose underlying data structure _could_ be a singly-linked list. LES is to Loyc trees as s-expressions are to singly-linked lists.
* Each "call" has a "target". Whereas LISP represents a method call with `(method arg1 arg2)`, Loyc represents a method call with `method(arg1, arg2)`. In LISP, the method name is simply the first item in a list, and there is no way to tell if `(x y z)` is a list of three items or a function call that takes two arguments. In contrast, most other programming languages separate the "target" from the argument list, with the notation `target(arg1, arg2)`. In a Loyc tree, a specific target would be used to denote a list (e.g `#tuple(item1, item2)`).
* Each node has a list of attributes. The concept of attributes was inspired by .NET attributes, so naturally a .NET or Java attribute would be represented in a Loyc tree by a Loyc attribute. But also, "trivia" such as comments and blank lines can be represented by attaching attributes to nodes, and modifiers like "public", "private", "override" and "static" are conventionally represented by attributes.
* Each node has an associated source file and two integers that identify the range of characters that the original construct occupied in source code. If a Loyc tree is created programmatically, a dummy source file and a zero-length range can be used.

Loyc trees: text representation
-------------------------------

Obviously, a text format is needed for Loyc trees.

My original plan was to use a subset of [Enhanced C#](http://ecsharp.net) as my "XML for code". However, since EC# is based on C#, it inherits some very strange syntax elements. Consider the fact that `(a<b>.c)(x)` is classified a "cast" while `(a<b>+c)(x)` is classified as a method call. EC# has a lot of oddball rules like that, which create unnecessary complication that should not exist in an AST interchange format.

Threfore I invented ([Loyc Expression Syntax](/les)). However, we can do better than _just_ an interchange format; I've made LES flexible enough to be used as a modern programming language in its own right.

Since LES can represent syntax from any language, I thought it was sensible to design it with no keywords. So currently, LES has no reserved words whatsoever, and is made almost entirely of "expressions". But "expressions" support a type of syntactic sugar called "superexpressions", which resemble keyword-based statements in several other languages.

Here is a simple Loyc tree expressed in LES:

    #if(c < 0, Print(@[en] "negative"), Print(@[en] "non-negative"));

The top-level loyc tree calls the identifier `#if`. `#` is a standard identifier character; it is treated no differently than a letter or an underscore, but by convention it marks an identifier as being somehow "special". `#if` is "called" with three arguments (we say "called" for lack of a better word, but probably `#if` would be a built-in construct, not a function). `c < 0` is also a call. `c < 0` calls the identifier `<` with two arguments, "c" and "0". The strings each have an attribute attached, which is an identifier called `en`.

I cannot say what this statement "means" in LES. It explicitly doesn't have a meaning; LES is merely a data structure, not a programming language, so constructs in LES have no inherent meaning. Remember, the LES concept is "XML for code": just as `<IF>` has no predefined meaning in XML, `#if` has no predefined meaning in LES.

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

Node styles and trivia attributes
---------------------------------

My implementation of Loyc trees has a concept of "node style", an 8-bit number that represents something stylistic and non-semantic about the source code. For example, 0xC and 12 are the same integer in two different styles. It is semantically the same—the compiler always produces the same program regardless of which form you choose. But it's a striking visual difference that should be preserved during conversion between languages. In my implementation, this difference is preserved in a node's `NodeStyle` property, using the bit flag `NodeStyle.Alternate`. `NodeStyle.Alternate` indicates that a number is hex, that a C# string is [verbatim](http://msdn.microsoft.com/en-us/library/aa691090(v=vs.71).aspx), or that an LES string is triple-quoted. 

**Style bits are an unnecessary flourish**; an implementation of Loyc trees can use trivia attributes instead, but the concept of style bits may save memory.

A trivia attribute is a Loyc node in an attribute list whose `Name` starts with `#trivia_`. Trivia attributes can be simple identifiers or calls. By convention, trivia attributes have low importance and can be optionally dropped when converting a Loyc tree to text.

Probably the most important use of trivia attributes is to denote comments. My plan is that when my parsers are complete, comments like

~~~C#
// Before
result = /* in the middle */ Func(); // after
~~~

will be represented using the following Loyc tree:

~~~
@[#trivia_SLCommentBefore(" Before")]
@[#trivia_SLCommentAfter(" after")]
result = (@[#trivia_MLCommentBefore(" in the middle ")] Func());
~~~

`#trivia_SLCommentBefore` and `#trivia_SLCommentAfter` are for single-line comments, while `#trivia_MLCommentBefore` and `#trivia_MLCommentAfter` are for multi-line comments.

If you manually insert a `#trivia_` attribute in your source code, it may disappear or change form when the code is printed out (it affects the output in some special way if the printer understands it, as with comments.)

Mappings between Loyc trees and programming languages
-----------------------------------------------------

It is necessary to standardize the Loyc trees that are used to represent code in a particular language, or there will be confusion and less interoperability.

For C# I chose a Loyc tree representation that closely mimics the original source code. Here are some examples:

| C# code                 | Loyc tree (LES prefix notation) | Loyc tree (friendly notation in LESv2) |
|-------------------------|---------------------------------|-----------------------------------|
| `if (c) A(); else B();` | `#if(c, A(), B())`              | N/A                               |
| `x = y + 1;`            | `@=(x, @+(y, 1));`              | `x = y + 1;`                      |
| `switch (c) { case '1': break; }` | ``#switch(c, @`{}`(#case('1'), #break));`` | `#switch c { #case '1'; #break; }` |
| `public string name = "John Doe";` | `[#public] #var(#string, @=(name, "John Doe"));` | `[#public] #var(#string, name = "John Doe");` |
| `int Square(int x) { return x*x; }` | ``#fn(#int32, Square, #(#var(#int32, x)), @`{}`(#return(@*(x, x))));`` | `#fn(#int32, Square, #(#var(#int32, x)) { return x * x; });` |
| `class Point { public int X, Y; }` | ``#class(Point, #(), @`{}`(@[#public] #var(#int32, x, y)));`` | `#class(Point, #(), { @[#public] #var(#int32, x, y); };` |
| `class List<T> : IList<T> { }` | ``#class(#of(List,T), #(#of(IList,T)), @`{}`());`` | `#class List!T #(IList!T) { };` |
| `x = (int)y;`           | `@=(x, #cast(y, #int32));`     | `x = #cast(y, #int32);` |

As you can see, there's a clear and obvious relationship between the Loyc tree and the original source code (read [LES](Loyc Expression Syntax) to understand the second notation better). Most keywords are represented by `#` plus the keyword name (I'm translating "int" as "#int32", however, which makes sense as a standard name common to all programming languages, or at least, all programming languages that support 32-bit integers.) Note: the names of operators [will change in the future](http://loyc.net/2016/put-back-the-sharp.html).

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

Using Loyc trees in .NET
------------------------

The .NET implementation is the only one that exists as of May 2016.

You can create Loyc trees programmatically using the [`LNodeFactory`](http://ecsharp.net/doc/code/classLoyc_1_1Syntax_1_1LNodeFactory.html) class. You have to provide a "source file" object that will be associated with each of the [`LNode`s](http://ecsharp.net/doc/code/classLoyc_1_1Syntax_1_1LNode.html) created by the factory; since you are creating nodes programmatically, just use `EmptySourceFile.Default` or create a `new EmptySourceFile("my source file's name")`. (If you feed your nodes into a compiler later, the source file name may be used by the compiler to display error messages regarding the nodes you created.)

An `LNodeFactory` is often named `F`:

~~~C#
LNodeFactory F = new LNodeFactory(new EmptySourceFile("Foo.cs"));
 
// Create a call to foo(xyz, 123)
LNode callFoo = F.Call("foo", F.Id("xyz"), F.Literal(123));
 
// Create a method definition: void foo(int x, string y) { return; }
LNode fooDecl = F.Def(F.Void, F.Id("foo"), 
                      F.Tuple(F.Var(F.Int32, F.Id("x")), F.Var(F.String, F.Id("y"))),
                      F.Braces(F.Call(S.Return)));
~~~

An easier way to create nodes is to parse LES code, although this can be costly because it happens at runtime. If you're using EC# then you can use [`quote {...}`](http://ecsharp.net/lemp/ref-other.html#quote) to produce code at compile time that will construct a Loyc tree directly.

To parse LES into a Loyc tree, call `LesLanguageService.Value.Parse("your expression here;")` (this is an extension method so you'll also need `using Loyc.Syntax;`). To convert a node (or list of nodes) to a string with the LES printer, call `LesLanguageService.Value.Print(node)`.

To parse EC# into a Loyc tree, call `EcsLanguageService.Value.Parse("your code here;")` (this is an extension method so you'll also need `using Loyc.Syntax;`). To treat a node (or list of nodes) as EC# code and print them with the EC# printer, call `EcsLanguageService.Value.Print(node)`. Don't worry, the EC# printer is fairly reliable at printing Loyc trees that do not represent EC# code; Loyc trees almost always round-trip correctly from a Loyc tree to EC# code and back.

The LES printer is the default when you call `LNode.ToString()`. To use EC# as the default instead, set `LNode.Printer` to `EcsLanguageService.Value.Printer`, or better yet, use code like this:

~~~
    using (LNode.PushPrinter(EcsLanguageService.Value.Printer)) {
      /* print Loyc trees with `tree.ToString()` here */
    }
~~~

### Important properties of Loyc nodes ###

A Loyc tree node (`LNode`) consists of the following main properties:

* `Range`: tuple of (`ISourceFile Source`, `int StartIndex`, `int Length`)
* `Attrs`: a list of attributes
* One of: `Value` (the value of a literal node), `Name` (the name of an identifier node), or `Target` and `Args`, which are child `LNode`s (e.g. The `Target` of `f(1, a)` is `f` and the `Args` list contains two nodes `1, a`). Note: the `Name` property is special because it works for simple calls in addition to identifiers, e.g. the name of `foo(x)` is `foo`.
* The `Kind` property returns the node type: `NodeKind.Literal`, `NodeKind.Id`, or `NodeKind.Call`. However, it's almost always easier to call one of the three test properties `IsLiteral`, `IsId` or `IsCall`.
* `Range`: indicates the source file that the node came from and location in that source file.
* `Style` an 8-bit flag value that is used as a hint to the node printer about how the node should be printed. For example, a hex literal like `0x10` has the `NodeStyle.Alternate` style to distinguish it from decimal literals such as 16. Custom display styles that do not fit into the `Style` property should be expressed with trivia attributes.

Identifier names are stored in `Symbol`s; a `Symbol` is a singleton string. One purpose of symbols is performance; in order to compare `"foo1" == "foo2"`, the actual characters much be compared one-by-one. `Symbol` comparisons, on the other hand, are lightning-fast reference comparisons. To get a symbol from a string `s` in C#, just use a cast: `(string) s`. To get the string out of a `Symbol`, call the `Symbol.Name` property or `ToString()` which is an alias for `Name`.

Common symbols for keywords and datatypes are defined in the `Loyc.Syntax.CodeSymbols` class in Loyc.Syntax.dll. Throughout the Loyc codebase, a `using S = Loyc.Syntax.CodeSymbols;` statement is often used to abbreviate `CodeSymbols` as `S`, so you can write things like `S.While` (which represents the `#while` symbol), `S.Class` (`#class`) `S.And` (`&&`), `S.AddSet` (`+=`), and so on. See the [source code of CodeSymbols](https://github.com/qwertie/ecsharp/blob/master/Core/Loyc.Syntax/CodeSymbols.cs).

You should also be aware of these helper methods:

* `IsIdNamed(Symbol name)`: returns true if the node is an identifier with the specified name.
* `Calls(Symbol name, int argCount)`: returns true if the node calls the specified name with the specified number of arguments, e.g. if I create a call with `c = F.Call("x", F.Literal(123))` then `c.Calls(GSymbol.Get("x"), 1)` is true.
* `CallsMin(Symbol name, int argCount)`: returns true if the node calls the specified name with the specified _minimum_ number of arguments.
* `HasPAttrs()`: returns true if the node has any "printable", meaning non-trivia, attributes attached to it.
* `IsParenthesizedExpr()`: checks for a `#trivia_inParens` attribute.
* `Descendants()`, `DescendantsAndSelf()`: enumerates the children of a node.
* `ReplaceRecursive(node => {...})` performs a find-and-replace operation, returning a new Loyc tree since `LNode` is immutable. The lambda should return `null` when it does not want to change the current node.

Node comparisons with `Equals()` test for structural equality rather than reference equality, and tend to be expensive. `GetHashCode()` can be expensive when first called, but the hashcode is cached.

### Modifying nodes ###

Since `LNode`s are immutable, you'll typically use one of the "`With`" methods to create modified nodes:

~~~C#
 // For modifying Id nodes (WithName(x) can also be used with call
 // nodes; in that case it means WithTarget(Target.WithName(x))).
 public virtual  LNode WithName(Symbol name)
 
 // For modifying Literal nodes
 public abstract LiteralNode WithValue(object value);
 
 // For modifying Call nodes (note: you can add arguments to a non-call node,
 // which produces a call node.)
 public virtual  CallNode WithTarget(LNode target);
 public virtual  CallNode WithTarget(Symbol name);
 public abstract CallNode WithArgs(RVList<LNode> args);
 public virtual  CallNode With(LNode target, RVList<LNode> args);
 public          CallNode With(Symbol target, params LNode[] args);
 public          LNode PlusArg(LNode arg); // add one parameter
 public          LNode PlusArgs(RVList<LNode> args);
 public          LNode PlusArgs(IEnumerable<LNode> args);
 public          LNode PlusArgs(params LNode[] args);
 public          LNode WithArgChanged(int index, LNode newValue);
 
 // For modifying the attribute list
 public virtual  LNode WithoutAttrs()
 public abstract LNode WithAttrs(RVList<LNode> attrs);
 public          LNode WithAttr(LNode attr)
 public          LNode WithAttrs(params LNode[] attrs)
 public          LNode WithAttrChanged(int index, LNode newValue)
 public          CallNode WithArgs(params LNode[] args)
 public          LNode PlusAttr(LNode attr); // add one attribute
 public          LNode PlusAttrs(RVList<LNode> attrs);
 public          LNode PlusAttrs(IEnumerable<LNode> attrs);
 public          LNode PlusAttrs(params LNode[] attrs);
 
 // Other
 public         LNode WithRange(SourceRange range) { return With(range, Style); }
 public         LNode WithStyle(NodeStyle style)   { return With(Range, style); }
 public virtual LNode With(SourceRange range, NodeStyle style)
~~~

Argument lists are stored in [VList data structures](http://www.codeproject.com/Articles/26171/VList-data-structures-in-C).

### List-node duality ###

Often, a single node can be treated as a list. For example, in C-like languages you can have a single statement attached to a `while` loop, or a braced block:

~~~csharp
while (c) single_statement();
while (c) { zero_or_more_statements(); ... }
~~~

The `AsList` extension method helps you treat that single statement as a list:

~~~csharp
public static VList<LNode> AsList(this LNode block, Symbol listIdentifier)
~~~

If `block` is the second argument to `#while` then `block.AsList(CodeSymbols.Braces)` would return a one-item list containing `single_statement()` in the first example, and in the second example it would return a list containing the contents of the braced block. The inverse operation is `AsLNode`:

~~~csharp
public static LNode AsLNode(this VList<LNode> list, Symbol listIdentifier)
~~~

#### Splicing

Occasionally a "splicing" operation is useful:

~~~csharp
// in LNodeExt class
public static VList<LNode> WithSpliced(this VList<LNode> list, LNode node, Symbol listName = CodeSymbols.Splice)
public static VList<LNode> WithSpliced(this VList<LNode> list, int index, LNode node, Symbol listName = CodeSymbols.Splice)
~~~

"Splicing" refers to conditionally inserting the arguments of one node into another node, if the node calls an identifier with a particular `Name`. For example, if you have a list `10, 11, 12` and a node `#splice(1, 2, 3)` then `WithSpliced(list, 0, node)` returns the list `1, 2, 3, 10, 11, 12`. But if the node does not call `#splice` then it is simply added to the list, e.g. if the `node` is `foo(1, 2, 3)` then `WithSpliced(list, 0, node)` returns the list `foo(1, 2, 3), 10, 11, 12`.

There are helper methods in `LNode` to do splicing:

~~~csharp
public CallNode WithSplicedArgs(int index, LNode from, Symbol listName)
{
    return WithArgs(LNodeExt.WithSpliced(Args, index, from, listName));
}
public LNode WithSplicedAttrs(int index, LNode from, Symbol listName)
{
    return WithAttrs(LNodeExt.WithSpliced(Attrs, index, from, listName));
}
~~~
