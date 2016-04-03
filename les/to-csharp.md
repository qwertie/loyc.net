---
layout: article
title: "Converting LES to C#/.NET"
toc: true
---

[LES](/les) is very comparable to Enhanced C#, especially its lexer (e.g. `"strings"`, `'c'`haracters, and `@identifiers` are practically the same between the two languages). Both languages have

* Token literals `@{...}`
* Blocks as expressions
* `@@Symbols`
* Triple-quoted strings with double-slash escape sequences like `\n/` for newline

And in addition, LES has superexpressions, which serve a similar function as block-call statements in EC#, and are also used to represent `fn` method declarations, `if` statements, `for` loops, `return` statements and virtually all other "special" syntax.

LeMP macros are available to "compile" LES into C# or Enhanced C#. To do this, your LES file needs to have the following first line:

    import macros LeMP.Prelude.Les;

To gain access to LLLPG, add
    
    import macros Loyc.LLPG;

### Differences & similarities between LeMP/LES and C# ###

I don't want to bore you with all the details, but the most striking difference between LES and C# is that LES has _no keywords whatsoever_.  Words like `if` and `while` are not parsed in a special way because of the actual word used, but because of the _how the statement is formatted_.

Anyway, here's a list:

#### Similarities:

The following statements mean the same thing in LeMP/LES and C#:

    return;
    return x;
    continue;
    break;
    var x = value;
    x = y++ * 10;
    Console.Write("Hi");

#### Differences:

In many cases the difference is simply that you need an extra semicolon or braces.

    using System.Collections.Generic;             // C#
    import System.Collections.Generic;            // LES
    class Foo {                                   // C#
      public Foo() : this(0) { }
      public Foo(int num) { }
    }
    class Foo {                                   // LES
      public cons Foo() { this(0); };
      public cons Foo(num::int) { };
    };

    class Foo : BaseClass, IEnumerable { }        // C#
    class Foo(BaseClass, IEnumerable!object) { }; // LES
    int Square(int x)       { return x*x; }       // C#
    def Square(x::int)::int { return x*x; };      // LES
    void Quit() { throw new QuitException(); }    // C#
    fn Quit()   { throw (new QuitException()); }; // LES
    int x, y; string s;                           // C#
    x::int; y::int; s::string;                    // LES
    int[] list1;      List list2;                 // C#
    list1::array!int; list2::List!int;            // LES
    bool flag = true;   int? maybe = null;        // C#
    flag::bool = @true; maybe::opt!int = @null;   // LES
    int more = (int)(num * 1.5);                  // C#
    more::int = (num * 1.5) -> int;               // LES

    while (x) Foo();                              // C#
    while (x) { Foo(); };                         // LES
    while x   { Foo(); };                         // LES
    do x/=2; while (x>y);                         // C#
    do { x/=2; } while (x>y);                     // LES
    for (x = 0; x < 10; x++) Console.WriteLine(x);      // C#
    for (x = 0, x < 10, x++) { Console.WriteLine(x); }; // LES
    foreach (var item in list) { Process(item); } // C#
    foreach (item in list)     { Process(item); };// LES
    if (c) return 0;                              // C#
    if (c) { return 0; };                         // LES
    if c   { return 0; };                         // LES
    if (i < list.Count) str.Append(list[i]);      // C#
    if (i < list.Count) { str.Append(list[i]); }; // LES
    unless i >= list.Count { str.Append(list[i]); }; // LES
    if (inc) x++; else x--;                       // C#
    if (inc) {x++;} else {x--;};                  // LES
    if inc   {x++;} else {x--;};                  // LES
    switch (x) { case 123: break; default: break; } // C#
    switch (x) { case 123; break; default; break }; // LES
    switch (x) { case 123 { break; }; default { break; }; }; // LES
    try { } catch (Exception ex) { } finally { }; // C#
    try { } catch ex::Exception  { } finally { }; // LES

Some error messages that don't mention a semicolon are actually referring to a missing semicolon; LeMP gets confused because certain sequences like

    if (a) { ... } if (b) { ... };

is parsed successfully as a _single_ statement `if (a, {...}, if, b, {...})`, which the "if" macro does not understand because there are too many arguments.

[Learn more about LES](http://loyc.net/les).