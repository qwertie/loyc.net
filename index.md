---
layout: default
title: Home
---

# <center>The Power of Choice</center>

Coders are constrained in the way they express themselves by the language they are using. Different languages have different strengths and weaknesses, but expert coders are often unable to choose a programming language or library with the combination of strengths they desire. There are many scenarios that produce this result...

- You're working on a large project based on Language X. You probably have to keep using Language X, no matter how weak it is for the task at hand.
- The best library for doing T is written in Language X, but you also need to do Y and X doesn't support Y very well.
- You've chosen language X and realize later that you need a good library for doing T. Sadly, none of the libraries for doing T in X are any good.
- Your code _might_, _just might_ need to run in a web browser. Now you can only use JavaScript, or one of the languages that transpile to JavaScript (where applicable, be careful to use only the parts that transpile properly).
- You're not writing code for the browser, but you want to re-use code that _was_ designed for the browser. There's a strong pull toward JavaScript, but it might not be the right fit for the new code you want to write.
- Your code needs high performance, but needs to interoperate with a slower language like Ruby, Python, etc. Now your design is highly constrained because it's hard to trade data between the two languages, due to fundamental mismatches between data types and memory management schemes. And if you don't choose C/C++, interoperability may be very hard in a language that doesn't understand C header files.
- You want a language with strong support for A, B and C, but no language exists that is strong in all three areas at the same time.
- You find a language that is excellent for A, B, and C, and start using it for a new project, only to discover that its IDE/Intellisense/Debugger/third party libraries are crap.
- You want to use two libraries related to the same topic (whether it's graphics, GIS, math, persistence, GUIs...), written in the same language, but it's painful because the two libraries use completely different interfaces and conventions.

# About Loyc

Loyc is the idea that all programming languages should interoperate with each other on a high level, so that you can code in the language of your choice without losing interoperability. The Loyc initiative is about finding ways to bring the world's programming languages closer together, and making developers more productive by giving them options they've never had before.

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

I call these "Loyc projects" because they are all meant to contribute to the twin goals of 

- Making programming languages interoperate better
- Giving developers more powerful tools

Here are some other projects that I have been thinking about:

- [MLSL](http://loyc.net/2014/design-elements-of-mlsl.html) - a Multi-Language Standard Library
- [SIL](https://github.com/qwertie/Loyc/wiki/Standard-Imperative-Language)
- [The Ultimate Programming Language](http://loyc.net/2015/ultimate-language.html)

These days I'm thinking the MLSL and SIL should be based on WebAssembly, which, if I have anything to say about it, will take over the world. So far, I haven't had much say in the matter.

At the moment, all these "Loyc tools" are limited to the .NET platform, but Microsoft has refused to fix the [design flaws](http://loyc.net/2014/dotnet-annoyances.html) in .NET, and it is clear that they will not generalize .NET to support new programming languages, either. Therefore, I have decided to shift my focus to WebAssembly in the future.

I'd like help to

- Make programming languages more interoperable, especially by building a standard on top of WebAssembly
- Make an extensible programming language
- Write parsers/printers to parse a programming language into a Loyc tree or the reverse (print a Loyc tree as text)
- Write a parser for LES for a language of your choice

Please send me a message if you want to help, or to inform me of any other interoperability-related initiatives. You can reach me at `gmail.com`, with account name `qwertie256`.
