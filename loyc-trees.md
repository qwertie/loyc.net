---
layout: page
title: Loyc trees
---

## Introduction

A major part of my plans for Loyc is the concept of an "interchange format" for source code. In three words, the concept is "XML for code"--a general representation of syntax trees for any language. The interchange format will:

- Assist tools that convert code between different languages, by providing a way to change a tree incrementally from one language until it conforms to the rules of another language.
- If a compiler uses Loyc trees internally, it can assist people writing compilers and compiler extensions by providing a simple textual representation of the compiler's intermediate output at various stages. Intermediate output will make it easier to see and discuss the processes going on inside a compiler.
- Promote compile-time metaprogramming to the entire software industry. With Loyc trees, it should be straightforward to add macro-based metaprogramming on top of any existing programming language, if someone has written a Loyc _node printer_ for that language.

Let me be clear, I do not advocate XML as an interchange format. In fact, I find XML's syntax extremely annoying! If I have to type "`&lt;`"  one more time I could scream. Rather, I want to create a standard language for describing syntax trees, just as XML or JSON is used to describe data. (In fact, my language could be useful for describing data too, just like [s-expressions](http://en.wikipedia.org/wiki/S-expressions) can describe data, although that is not its main purpose).

The key concept for representing code from different programming languages is the Loyc tree. "Loyc tree" refers both to the conceptual structure that I have invented, and to its _in-memory_ representation. An interchange format (typically [LES](Loyc Expression Syntax) or [EC#](https://github.com/qwertie/Loyc/wiki/Enhanced-C%23)) will allow Loyc trees to be converted to plain text and vice versa.

Every node in a Loyc tree is one of three things:

* An identifier, such as the name of a variable, type, method or keyword.
* A literal, such as an integer, double, string or character.
* A "call", which represents either a method call, or a construct like a "class" or a "for loop".

Unlike in most programming languages, Loyc identifiers can be any string--any string at all. Even identifiers like `\n\0` (a linefeed and a null character) are supported. This design guarantees that a Loyc tree can represent an identifier from any programming language on Earth. Literals, similarly, can be any object whatsoever, but when I say that I am referring to a Loyc tree that exists in memory. When a Loyc tree is serialized to text, obviously it will be limited to certain kinds of literals (depending on the language used for serialization).

Each Loyc node also has a list of "attributes" (usually empty), and each attribute is itself a Loyc tree. Loyc trees also contain position information (location within a source file).

In other words, a Loyc tree is a data structure with these properties (potential parts):

1. `Range`: tuple of (source file name, integer position, integer length)
2. `Attrs`: a list of attributes
3. One of: a literal (`Value`), an identifier (`Name`), or a call (`Target` plus `Args`).

Currently the only implementation is in C#, which has no ADTs, so the properties `Value`, `Name`, `Target` and `Args` exist at all times and you can use `IsLiteral`, `IsId` and `IsCall` to distinguish between the three types of nodes.

## Loyc trees versus s-expressions ##

Loyc trees are inspired by LISP trees, but designed for non-LISP languages. If you've heard of [LISP](http://en.wikipedia.org/wiki/Lisp_(programming_language)), well, Loyc Expression Syntax (LES) is basically a 21st century version of the [S-expression](http://en.wikipedia.org/wiki/S-expressions). The main differences between s-expressions and Loyc trees are:

* Each "call" has a "target". Whereas LISP represents a method call with `(method arg1 arg2)`, Loyc represents a method call with `method(arg1, arg2)`. In LISP, the method name is simply the first item in a list; but most other programming languages separate the "target" from the argument list, hence `target(arg1, arg2)`.
* Each node has a list of attributes. The concept of attributes was inspired by .NET attributes, so it will not surprise you to learn that a .NET attribute will be represented in a Loyc tree by a Loyc attribute. Also, modifiers like "public", "private", "override" and "static" will be represented by attaching attributes to a definition.
* A "call", which represents either a method call, or a construct like a "class" or a "for loop". By convention, constructs that are built into a language use a special identifier that starts with `#`, such as `#class` or `#for` or `#public`.
* Each node has an associated source file and two integers that identify the range of characters that the original construct occupied in source code. If a Loyc tree is created programmatically, a dummy source file and a zero-length range will be used.

Tuples like `(a, b)` have special syntax in EC# and LES and are stored as calls with `#tuple` as the target (i.e. `#tuple(a, b)` is equivalent).

## Loyc trees: text representation ##

Obviously, a text format is needed for Loyc trees. However, I think I can do better than just an interchange format, so I have a plan to make LES into both an interchange format and a general-purpose programming language in its own right. The interchange format is called [LES](Loyc Expression Syntax), and the programming language (which does not exist yet) will called [LEL](https://github.com/qwertie/Loyc/wiki/Loyc-Expression-Language).

Since LES can represent syntax from any language, I thought it was sensible to design it with no keywords. So tentatively, LES has no reserved words whatsoever, and is made almost entirely of "expressions". But "expressions" support a type of syntactic sugar called "superexpressions", which resemble keyword-based statements in several other languages.

I've made a somewhat radical decision to make LES partially whitespace-sensitive. I do not make this decision lightly, because I generally prefer whitespace to be ignored inside expressions*, but the whitespace sensitivity is the key to making it keyword-free. More on that later.

&#42; I don't mind whitespace-based nesting, like Python has, but it should be optional, and I don't want to mandate whitespace-sensitive expressions. For example there exist languages that parse `x+1 * 2`  as `(x+1) * 2`  and that's a bit too radical for me. I wouldn't mind using a language like that, mind you, but I don't think I should (or could) "push" a language like that onto developers. So, I really don't want LES to be whitespace-sensitive inside expressions, but I have adopted partial whitespace sensitivity for LES because it has a large benefit that I do not know how to accomplish any other way. I would welcome an alternative that provides a nice syntactic sugar without being whitespace-sensitive.

My original plan was to use a subset of Enhanced C# as my "XML for code". However, since EC# is based on C# it inherits some very strange syntax elements. Consider the fact that `(a<b>.c)(x)` is classified a "cast" while `(a<b>+c)(x)` is classified as a method call. Features like that create unnecessary complication that should not exist in an AST interchange format.

Therefore, I invented Loyc Expression Syntax. Here is a simple Loyc tree in LES and EC#:

    @#if(c < 0, Print([en] "negative"), Print([en] "non-negative"));

At the expression level, LES and EC# are syntactically very similar; thus, this statement is valid EC# and LES code at the same time (and of course, it means the same thing in both languages).

The top-level loyc tree calls the identifier `#if`. The `@` sign is used to prevent the EC# compiler from treating `#if` as preprocessor directive. `#` is a standard identifier character; it is treated no differently than a letter or an underscore, but by convention it marks an identifier as being somehow "special". `#if` is "called" with three arguments (we say "called" for lack of a better word, but of course `#if` is a built-in construct, not a function). `c < 0` is also a call. `c < 0` calls the identifier `<` with two arguments, "c" and "0". The strings each have an attribute attached, which is an identifier called `en`.

I cannot say what this statement "means" in LES. It explicitly doesn't have a meaning; LES is merely a data structure, not a programming language, so constructs in LES have no inherent meaning. Remember, the LES concept is "XML for code": just as `<IF>` has no predefined meaning in XML, `#if` has no predefined meaning in LES.

However, the statement does have meaning in EC#. In fact, it is equivalent to a standard "if" statement:

~~~C#
if (c < 0)
   Print([en] "negative");
else
   Print([en] "non-negative");
~~~

Again, `[en]` is an attribute. Whereas plain C# allows attributes only on declarations such as fields and classes, EC# allows attributes on any expression. Attributes are sometimes used to provide extra information to macros (compiler extensions) at compile-time; otherwise they are meaningless and the compiler should produce a compiler warning about their uselessness.

In this case, one could imagine writing a compiler extension that helps do internationalization. You could define `[en]` to mean that the text is in English and needs to be translated to all other supported languages. Again, that's not something that EC# will support directly--it's a feature somebody might add. (Note: I'd probably support translations in a different way, using an attribute on the function being called rather than at the call site. But both approaches might be useful.)

Please see the [LES](Loyc Expression Syntax) and [EC#](https://github.com/qwertie/Loyc/wiki/Enhanced-C%23) pages for more information.

## Node styles and trivia attributes ##

My implementation of Loyc trees has a concept of "node style", an 8-bit number that represents something stylistic and non-semantic about the source code. For example, 0xC and 12 are the same integer in two different styles. It is semantically the same&mdash;the compiler always produces the same program regardless of which form you choose. But it's a striking visual difference that should be preserved during conversion between languages. In my implementation, this difference is preserved in a node's `NodeStyle` property, using the bit flag `NodeStyle.Alternate`. `NodeStyle.Alternate` indicates that a number is hex, that a C# string is [verbatim](http://msdn.microsoft.com/en-us/library/aa691090(v=vs.71).aspx), or that an LES string is triple-quoted.

For information that doesn't fit in the 8 bits available, you can use "unprintable trivia attributes" instead. An unprintable attribute is a Loyc node in an attribute list whose `Name` starts with `#trivia_`. Trivia attributes can be simple identifiers or calls.

Probably the most important use of trivia attributes is to denote comments. My plan is that when my parsers are complete, comments like

~~~C#
    // Before
    result = /* in the middle */ Func(); // after
~~~

will be represented using the following Loyc tree:

~~~
    [#trivia_SLCommentBefore(" Before")]
    [#trivia_SLCommentAfter(" after")]
    result = ([#trivia_MLCommentBefore(" in the middle ")] Func());
~~~

`#trivia_SLCommentBefore` and `#trivia_SLCommentAfter` are for single-line comments, while `#trivia_MLCommentBefore` and `#trivia_MLCommentAfter` are for multi-line comments.

If you manually insert a `#trivia_` attribute in your source code, it will disappear or change form when the code is printed out (it disappears if the printer doesn't specifically understand it, and it affects the output in some special way if the printer does understand it, as with comments.)

## The mapping between Loyc trees and programming languages ##

It is necessary to standardize the Loyc trees that are used to represent code in a particular language, or there will be confusion and less interoperability.

For C# I have chosen a Loyc tree representation that closely mimics the original source code. Here are some examples:

| C# code                 | Loyc tree (LES prefix notation) | Loyc tree (LES friendly notation) |
|-------------------------|---------------------------------|-----------------------------------|
| `if (c) A(); else B();` | `#if(c, A(), B())`              | `#if c A() B()`                   |
| `x = y + 1;`            | `@=(x, @+(y, 1));`              | `x = y + 1;`                      |
| `switch (c) { case '1': break; }` | ``#switch(c, @`{}`(#case('1'), #break));`` | `#switch c { #case '1'; #break; }` |
| `public string name = "John Doe";` | `[#public] #var(#string, @=(name, "John Doe"));` | `[#public] #var #string name = "John Doe";` |
| `int Square(int x) { return x*x; }` | ``#fn(#int32, Square, #(#var(#int32, x)), @`{}`(#return(@*(x, x))));`` | `#fn #int32 Square #(#var #int32 x) { return x * x; };` |
| `class Point { public int X, Y; }` | ``#class(Point, #(), @`{}`([#public] #var(#int32, x, y)));`` | `#class Point #() { [#public] #var #int32 x y; };` |
| `class List<T> : IList<T> { }` | ``#class(#of(List,T), #(#of(IList,T)), @`{}`());`` | `#class List!T #(IList!T) { };` |
| `x = (int)y;`           | `@=(x, #cast(y, #int32));`     | `x = #cast(y, #int32);` |

As you can see, there's a clear and obvious relationship between the Loyc tree and the original source code (read [LES](Loyc Expression Syntax) to understand the second notation better). Most keywords are represented by `#` plus the keyword name (I'm translating "int" as "#int32", however, which makes sense as a standard name common to all programming languages, or at least, all programming languages that support 32-bit integers.) At one time, operators were named with `#` plus the operator name, but I [reconsidered](http://loyc.net/2014/in-operator-names-to-be-removed.html).

Occasionally, it is not possible (or, I felt, not ideal) to use the original keyword. For example, C# has two unrelated statements that are both called "using":

~~~C#
    using System.Collections;
    using (Foo()) { ... }
~~~

In this case I decided to use `#using` for the second statement but `#import` for the first (I guess I could have used `#using(...)` for both, but then it would be necessary to check the arguments to figure out which kind of `using` statement it is.)

Full documentation of the mapping from C# to Loyc trees will come later (as far as I know, no one is reading this).

Obviously, it's important to use a consistent mapping. While I have chosen `#var(#int32, x, y = 0)` to represent `int x, y = 0`, it could just as easily be `#var(#int32, x, y(0))` or `#varDecl(x, #int32, y = 0, #int32)` or something else. It would be inconvenient if multiple mappings existed for the same language, so part of the Loyc project's mandate will be to 

* regulate these mappings
* document these mappings
* have code for parsing code into a Loyc tree
* have code for printing a Loyc tree to text in a given language

The following guidelines should be followed to design a mapping:

- The Loyc tree should resemble the original code. For example, notice how `#var(Foo, x = -1)` resembles `Foo x = -1`, and `#fn(#void, f(), {})` resembles `void f() {}`.
- The Loyc tree should be consistent between languages, if this is easy to achieve. An example of this is using `#int64` rather than `#long` in C# to represent a 64-bit integer. In the future I'll define [Standard Imperative Language] as an "anchor" for future mappings. If SIL contains a construct that is semantically identical to a construct in language X, then language X's mapping should use the SIL construct, rather than inventing a new construct that means the same thing. Sometimes this rule will override rule #1.
- The tree should be easy to interpret after it is parsed. When I used `#import` to represent the `using` directive, I was favoring this rule over rule #1. On the other hand, I violated this rule slightly for variable declarations. Although the variable name is always stored in the second argument (or Nth argument for multi-variable declarations), you must check if the second argument calls `=` or not. If it does, the variable name is stored inside the call to `=`. This complication was a pain point (I felt there was no ideal solution), which perhaps I will write about later (actually I'm reconsidering the decision now). But, unless you have a specific reason to violate this rule, try to ensure that interpreting the tree is easy.

These rules are sometimes in conflict, so if two people both try to define mappings they will inevitably make different decisions. That's why we need to standardize the mappings as part of the Loyc project.

## Using Loyc trees in .NET ##

You can create Loyc trees programmatically using the [`LNodeFactory`](http://loyc.net/doc/code/classLoyc_1_1Syntax_1_1LNodeFactory.html) class. You have to provide a "source file" object that will be associated with each of the [`LNode`s](http://loyc.net/doc/code/classLoyc_1_1Syntax_1_1LNode.html) created by the factory; since you are creating nodes programmatically, just use `EmptySourceFile.Default` or create a `new EmptySourceFile("my source file's name")`. (The source file name may be used later to display error messages, if necessary, that are related to the nodes that you created.)

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

An easier way to create nodes is to parse LES code, although this can be costly because it happens at runtime. Call `LesLanguageService.Value.ParseSingle("your expression here;")` to parse a string into a Loyc tree. Once EC# is a viable programming language, you'll be able to use code quotes to produce Loyc trees at compile-time; most likely, quoted code will end up using `LNodeFactory` behind the scenes.

The EC# printer is currently more mature than the LES printer, although the LES printer is the default. To print a node with the EC# printer, call `EcsNodePrinter.Print()`. To use the EC# printer by default when calling `LNode.ToString()`, set `LNode.Printer` to `EcsNodePrinter.Printer`. You could also use code like this:

~~~
    using (LNode.PushPrinter(EcsNodePrinter.Printer)) {
      /* print some Loyc trees with ToString() here */
    }
~~~

### Important properties of Loyc nodes ###

A Loyc tree node (`LNode`) consists of the following main properties:

* `Range`: tuple of (source file name, integer position, integer length)
* `Attrs`: a list of attributes
* One of: `Value` (the value of a literal node), `Name` (the name of an identifier or the name of a function being called), or `Target` and `Args`, which are child `LNode`s (e.g. The `Target` of `f(1, a)` is `f` and the `Args` list is `{ 1, a }`). Note: the `Name` property works for simple calls as well as identifiers; the name of `foo(x)` is `foo`.
* The `Kind` property returns the node type: `NodeKind.Literal`, `NodeKind.Id`, or `NodeKind.Call`. However it's usually easier to call one of the three test properties `IsLiteral`, `IsId` or `IsCall`.
* `Range`: indicates the source file that the node came from and location in that source file.
* `Style` an 8-bit flag value that is used as a hint to the node printer about how the node should be printed. For example, a hex literal like `0x10` has the `NodeStyle.Alternate` style to distinguish it from decimal literals such as 16. Custom display styles that do not fit in the `Style` property can be expressed with attributes.

Identifier names are stored in `Symbol`s; a `Symbol` is a singleton string. One purpose of symbols is performance; in order to compare `"foo1" == "foo2"`, the actual characters much be compared one-by-one. `Symbol` comparisons, on the other hand, are lightning-fast reference comparisons. To get a symbol from a string `s`, call `GSymbol.Get(s)`. To get the string out of a `Symbol`, call `Symbol.Name`.

Common symbols for keywords and datatypes are defined in the `Loyc.Syntax.CodeSymbols` class in Loyc.Syntax.dll. A `using S = Loyc.Syntax.CodeSymbols;` statement is often used to abbreviate `CodeSymbols` as `S`, so you can write things like `S.While` (which represents the `#while` symbol), `S.Class` (`#class`) `S.And` (`&&`), `S.AddSet` (`+=`), and so on. See the [source code of CodeSymbols](https://github.com/qwertie/Loyc/blob/master/Src/Loyc.Syntax/CodeSymbols.cs).

You should also be aware of these helper methods:

* `IsIdNamed(Symbol name)`: returns true if the node is an identifier with the specified name.
* `Calls(Symbol name, int argCount)`: returns true if the node calls the specified name with the specified number of arguments, e.g. if I create a call with `c = F.Call("x", F.Literal(123))` then `c.Calls(GSymbol.Get("x"), 1)` is true.
* `CallsMin(Symbol name, int argCount)`: returns true if the node calls the specified name with the specified _minimum_ number of arguments.
* `HasPAttrs()`: returns true if the node has any "printable", meaning non-trivia, attributes attached to it.
* `IsParenthesizedExpr()`: checks for a `#trivia_inParens` attribute.
* `Descendants()`, `DescendantsAndSelf()`: enumerates the children of a node.

Node comparisons with `Equals()` test for structural equality rather than reference equality; note that `GetHashCode()` tends to be somewhat expensive currently.

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

Argument lists are stored in [RVList data structures](http://www.codeproject.com/Articles/26171/VList-data-structures-in-C).

### Splicing ###

Occasionally a "splicing" operation is useful:

~~~C#
 public CallNode WithSplicedArgs(int index, LNode from, Symbol listName)
 public CallNode WithSplicedArgs(LNode from, Symbol listName)
 public LNode WithSplicedAttrs(int index, LNode from, Symbol listName)
 public LNode WithSplicedAttrs(LNode from, Symbol listName)
 public static LNode MergeLists(LNode node1, LNode node2, Symbol listName)
~~~

"Splicing" refers to conditionally inserting the arguments of one node into another node, if the node calls an identifier with a particular `Name`. For example, if `fooCall` represents the code `foo(10, 11, 12)` and `child` represents the call `#splice(1, 2, 3)` then 

~~~C#
 list.WithSplicedArgs(0, child, S.Splice);
~~~

Returns `foo(1, 2, 3, 10, 11, 12)` (S.Splice refers to the `#splice` symbol). On the other hand, if `child` represents the statement `x += y` (which is equivalently written as a call to `+=`, i.e. ``@`+=`(x, y)``), then 

~~~C#
 list.WithSplicedArgs(0, child, S.Splice);
~~~

Returns `foo(x += y, 10, 11, 12)`. The point is, a splice operation inserts the arguments only if the node has the specified `Name`. In the first case the name matched, so splicing occurred, while in the second case there was no match; the `Name` of `x += y` is `+=`, so the splicing function simply inserts the node itself.

You can also convert a single `LNode` into a list of nodes or vice versa, using these extension methods:

~~~C#
/// <summary>Interprets a node as a list by returning <c>block.Args</c> if 
/// <c>block.Calls(listIdentifier)</c>, otherwise returning a one-item list 
/// of nodes with <c>block</c> as the only item.</summary>
public static RVList<LNode> AsList(this LNode block, Symbol listIdentifier)
{
    return block.Calls(listIdentifier) ? block.Args : new RVList<LNode>(block);
}
    
/// <summary>Converts a list of LNodes to a single LNode by using the list 
/// as the argument list in a call to the specified identifier, or, if the 
/// list contains a single item, by returning that single item.</summary>
/// <param name="listIdentifier">Target of the node that is created if <c>list</c>
/// does not contain exactly one item. Typical values of this parameter 
/// include "{}" and "#splice".</param>
public static LNode AsLNode(this RVList<LNode> list, Symbol listIdentifier)
{
    return list.Count == 1 ? list[0] : LNode.Call(listIdentifier, list, SourceRange.Nowhere);
}
~~~
