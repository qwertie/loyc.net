---
layout: page
title: History
---

I, David Piepgrass, started the Loyc project in 2007, after I added a feature to a compiler and the makers of that language showed no interest in adding that feature to their language. It got me thinking that progress in the design of popular programming languages is held back by the gatekeepers we put in charge of them. It seemed to me that programming languages should offer ways for users to add features, in such a way that two features made by different people should ordinarily be compatible.

Since Microsoft .NET was the only environment designed for multi-language interoperability, I started with the goal of making a .NET compiler that accepted multiple _syntactic styles_ (e.g. Visual Basic and C#) so that 

1. Code from multiple languages could combine into a single binary, with mutual dependencies between the languages. This would eliminate the need to write all your _new_ code in the same language as your _old_ code. It would also allow building or using "experimental" languages without hurting interoperability.
2. Existing programming languages could be extended with new features. Most importantly, the hope was that new features could be created by third parties unaffiliated with the Loyc compiler, thus democratizing programming language design.

My project didn't get very far, because of the difficulty of creating sufficiently flexible extensibility mechanisms that would allow different programming language features written by different people and _unaware of each other_ to "get along" and work together. Eventually, in early 2012, I decided to tackle a simpler problem first, by designing [Enhanced C#](http://ecsharp.net), which has been a more successful project due to its narrower focus.

In the same time frame that I developed Enhanced C#, I also created

- A parser generator ([LLLPG](http://ecsharp.net/lllpg)) designed to help you write parsers with performance similar to hand-written parsers. I hoped to use this for EC# and all other parsing tasks.
- [Loyc trees](/loyc-trees), a concept for a simple "universal" syntax tree designed for all programming languages in the Algol family (C++, JavaScript, C#, etc.) Loyc trees are inspired by the simplicity of LISP s-expressions, adding just enough additional complexity to make them useful for representing any programming language, not just LISP languages.
- [Loyc expression syntax](/les), a language for storing Loyc trees. LES is a superset of JSON that resembles JavaScript, C#, D, and others.
- [LeMP](http://ecsharp.net/lemp), a LISP-style Macro Processor for Enhanced C#
- Visual Studio extensions for EC#/LES syntax highlighting and LLLPG/LeMP
- General-purpose "core" libraries that you can read about at [core.loyc.net](http://core.loyc.net)

The idea of a single compiler that accepts multiple input syntaxes is now on the back burner. The definition of "Loyc" is continuously evolving, and every time I try to describe it, something different comes out. The original idea of a multi-syntax compiler grew out of two convictions:

1. That a single programming language can be everything to everybody: designed for high performance, for scripting and prototyping, for metaprogramming, for a variety of paradigms (functional, object-oriented, generic, declarative/goal-driven) and for IDEs (code completion and advanced debugger features). At the time, there was no language fitting that description, and only now, ten years later, are languages becoming popular that approximate  that description (although languages that aspired to all these goals, like Nim and D version 2, have existed for a few years now).
2. That users should be able to add features to a programming language by themselves, unconstrained by a language committee acting as gatekeeper.

As time went on, I began to understand that I couldn't do everything myself, and I also noticed that while developers were creating interesting new languages all the time, none of them was gaining traction. So I shifted from the idea of a single monolithic compiler that does everything to a set of tools, and my focus also shifted from "how to design the world's best programming language" to "how to improve interoperability between languages" and "how to convert code between languages."

So how would I describe Loyc today? It's a community of people aspiring to unify the foundations of the software industry by creating open-source tools and standards that are general, powerful, and cut across the lines between programming languages. Except that I haven't figured out how to build a community yet, and most of the necessary tools don't exist. But that's the dream.
