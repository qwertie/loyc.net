---
title: "Primary principles & practices of programming"
layout: post
toc: false
tagline: "Short version, first draft"
---

Since watching Bret Victor's "[Inventing on Principle](https://www.youtube.com/watch?v=PUv66718DII)" I was sure I wanted to "Invent on Principle" as he suggested, but the tricky part is figuring out what my Big Principle is.

One angle I've homed in on is the principle that "easy-sounding things should be easy". Programming is full of things that seem like they should be easy, but are unnecessarily hard in practice. Things like drawing graphics efficiently (you can start easily with Windows Forms but if you want efficient drawing you have to make this massive leap over to DirectX, in which case, good luck implementing your GUI widgets by hand) playing music and sounds (okay, playing a .wav file might be a simple static method call, but generating audio continuously or playing an mp3 or opus stream might be far more difficult), or keeping a user interface synchronized with the underlying data model (but see SwiftUI, Assisticant, MobX/KnockoutJS for reactive solutions in Swift, C# and JS respectively)

The most basic idea of making software easy is pretty simple: 

1. Imagine what your code _would_ look like _if_ your task _were_ as easy as possible. And maybe this is ordinary code, or maybe it's a diagram, or maybe it's a visualization that helps you build the code.
2. Next, you build the tool that makes your dream a reality - it could be as small as a single function, or as large as a new programming language or debugger infrastructure.

And so I long to be a tools developer, to make tools that will make easy-sounding things actually straightforward. (I don't have the money or companionship to do this even part-time, but perhaps someday...)

But this basic separation, of making a tool to make something easier, and then using that tool, is not specific to tools developers. Instead, this basic approach is something _all_ developers should use in their everyday work.

Here's a small code snippet I saw recently (names have been changed to protect the guilty):

~~~cs
    var nameStem = metaData.tableName;
    if (metaData.tableName.EndsWith("_VERSIONS"))
    {
        nameStem = metaData.tableName.Substring(0,
           obj.tableName.Length - ("_VERSIONS").Length);
    }
    if (metaData.tableName.EndsWith("_RECORD"))
    {
        nameStem = metaData.tableName.Substring(0,
           obj.tableName.Length - ("_RECORD").Length);
    }
~~~

I could criticize this is as a a violation of the well-known Don't Repeat Yourself (DRY) principle, because the same code pattern appears twice. But perhaps a simpler, more general way to look at it is as a failure of the developer to make things easy, and specifically, as a failure to use that general technique of separating the code into a general tool, on the one hand, and code to do a specific task, on the other.

So I imagined what the code would look like if it had been easier to write:

~~~cs
    var baseName = metaData.tableName.WithoutSuffix("_VERSIONS")
                                     .WithoutSuffix("_RECORD");
~~~

This doesn't do precisely the same thing as the original code, but I happen to know that each `tableName` only has a single suffix, so in practice it is the same. Notice that this code isn't just easier to write. It is at least as important that this is easier to read (after all _I was reading this code as part of my job_.)

And then you write the tool:

~~~cs
    public static string WithoutSuffix(this string s, string suffix) => 
        s.EndsWith(suffix) ? s.Substring(0, s.Length - suffix.Length) : s;
~~~

Remarkably, within days of creating this new tool, I had used it multiple times in my own code, and I was rather amazed that I had never before thought to write this function. I had internalized the DRY principle well enough to notice the repetition and remove it, but as a tools developer at heart, it was ironic not to have noticed earlier this opportunity to build a tool.

In this particular codebase there was, at the time, no designated location for general-purpose tools like this. Which is not to say there were no general-purpose classes or functions, but to the extent they existed, they were scattered about. I created a folder called GeneralPurpose and gathered a few of the existing general-purpose bits of code and moved them into this folder. And I tested the boundaries by not asking for anyone's permission first. _Because I knew it was the right thing to do_.

If you take one thing away from this post, let it be this basic separation: _imagine_ things are easy, then _make them that way_.

Often the correct way to solve a large problem involves creating a large tool that is fairly elaborate. Often this will not be acceptable to management. They will say they don't have the budget to make a tool. Or they'll say it's not your job and you have to stay on task. Or they'll say we can think about this later, after you've actually done the task. Or if the best approach involves improving an existing open-source tool, they'll say you were hired to write proprietary software, not free software. And sometimes if you want to keep your job, you'll have to put up with this.

But as the example above illustrates, often there are little tools you can make, that will improve your ability to write software without the cost of making a large tool. The little tool might not be as broadly applicable as a bigger, more ideal tool that you can imagine, and it might not even reduce total development time as much as a bigger tool, but the little one will be easier to get through management, the time needed to build it will be more predictable, and by making a small tool, you might learn something that will help you build a larger and better tool later.

So if there's one technique you learn here today, let it be the Seeing Through The Eye of the Tool Writer.

But there are others. I have seen many proposals for programming principles: SOLID, KISS, YAGNI, composition, Law of Demeter, avoid premature optimization, avoid high computational complexity, test-driven development, object orientation, avoid mutability, blah blah blah. But which principles are the most important? Often, principles come into conflict with each other; it is hard, and sometimes impossible, to satisfy every good-sounding principle at the same time.

So as a senior developer, allow me to summarize what I think the most important principles are, in approximate order of priority.

### 1. Separate concerns!! ###

A program is like a house: just as walls give a house structure, separation of concerns gives a program structure. Most programming languages do not facilitate separation of concerns because they allow almost anything to touch almost anything else. Programmer discipline is required to create imaginary walls where no connections are allowed. For example, a language typically can't stop the GUI from directly running SQL update commands, so the wall between GUI and a specific data persistence library rests only in the minds of the developers.

I think some people "solve" this lack of compiler enforcement with "microservices architectures", but any solution that involves huge amounts of otherwise unnecessary plumbing code is a poor solution.

Concerns should be separated both "vertically" and "horizontally". Vertical separation is a stack of dependencies: module C depends on module B which depends on module A. Horizontal separation is _unrelatedness_: line drawing is separate from circle drawing; copying text to the clipboard is separate from deleting text, which is separate from the Undo Stack. The Cut command may use all three of these things, but it should be separate from them vertically, and they are separate from each other horizontally.

You can visualize separation of concerns as a 3-D pyramid of modules, with high-level functionality at the top, and low level functionality at the bottom.

![Marshmallows](marshmallow-software-dependency-pyramid.webp)

As the diagram illustrates, it should be possible to arrange all modules in a vertical way so that there are no horizontal or upward connections between them: higher levels only use lower levels.

- Some modules will be very popular and used by many others (bottom pink marshmallows).
- some will use many modules (but not an _excessive_ number, with the exception of wiring for dependency injection)
- Some will use or be used by only one module
- There can be multiple programs or entry points based on the same modules (upper pink marshmallows).
- Most software will have more vertical levels than the five shown here, but modules can reach down multiple levels to reach their dependencies (green marshmallows).

### 2. The Generalized Don't Repeat Yourself principle ###

Avoid not just repeating information or business logic in multiple places, but minimize repetition of any patterns whatsoever.

**Elegant code minimizes the number of tokens in the program (not including comments)**.

### 3. Use the correct amount of abstraction: no more, no less ###

Create functions and abstractions appropriately, but avoid excessive layering/wrapping.

Wrapping is often motivated by prior failure to refactor, or failure to test, leading to brittle code you don't dare touch. But having lots of layers, that basically all have the same responsibility, with confusingly similar names... well, it's confusing. You're taking a bad situation and making it worse. Don't do that.

Too many abstractions can also occur because you are imagining several use cases that may never materialize, so you write a "framework" for these imaginary use cases. This is where YAGNI comes in: You Ain't Gonna Need It.

![](This-time-I'll-build-it-the-right-way.jpg)

**Don't** design large frameworks. Design libraries that are small, yet flexible and powerful within that constraint of smallness. But **do** design them. You do need some layers: real-world functionality should usually be built upon tools that are built on other tools.

Focus mostly on features that you will actually use; considering those imaginary use cases is fine, but it should influence the code in ways _other than_ making it larger. For example, make sure that the public interface of each layer enables you to improve the power, performance or flexibility of the implementation later.

### 4. Minimize the number of code entities by merging things that turn out to be very similar. ### 

When someone is reading your code, every new and separate entity is something that the reader has to expend mental energy to keep track of. Try to reduce this mental energy requirement, and your code may end up shorter as a happy side effect. For example, I realized that "logging" and "reporting a set of warnings and errors to a user" are mostly the same thing, so in the [Loyc.Essentials](http://core.loyc.net/essentials) way of doing things you would use the same classes and the same interface `IMessageSink` for both (sometimes I wonder if no one else has noticed this).

Similarly I often see interfaces like `ISomethingRetriever` whose job is to retrieve some kind of object given some kind of key. That's basically a dictionary! Just implement `IReadOnlyDictionary`, you fool, or whatever your language's equivalent is. If you really do need a new interface, figure out if you can at least make it similar to something else in your language's standard library, if possible.

Do not create a new and different interface every time you need to retrieve a new kind of object. More broadly, design both your interfaces and your components to be **generic, reusable, and ignorant.**

### 5. Write good tests (or prove correctness) ###

If you've got tools that can mathematically prove important properties of your software, wow, good for you. For everyone else there's automated tests.

They don't have to be "unit" tests. I don't know where this idea came from that the dependencies of the component under test have to be fake (test doubles). It's okay if you are testing component C and C uses components A and B. It's especially okay if you also have separate tests for A and B by themselves.

Avoid using mocks. In my experience, well-designed software never _needs_ mocks in its tests. If you can't fully test the software without mocks, it means the software is structured incorrectly. "White-box" testing may require mocks to check implementation details, but this reduces your freedom to change the internal design of the component. With this form of testing, when you change the internal design you may cause mock-based tests to fail, even if the new implementation is correct and working properly. This is not good.

Do use complex tests with randomized and/or realistic data. 

You have a limited amount of time to write tests, so your first priority should be to make sure that basic functionality - the things you expect users or customers to do with your software - work. Your other main priorities should be making sure that less common, more obscure functionality works; that edge cases work; and that corner cases work. Your _lowest_ priority should be making sure a constructor throws an exception when "null" is passed to it.

### 6. Pick good names ###

_Spend time_ thinking about names, and write documentation that describes things in a different way than the names do. Be on the lookout for ambiguity and vagueness. It may seem clear to you that a `RecordDependencyPropertyCollection` is a sequence (whose order is not important) of pieces of metadata about dependencies between pairs of records in your application, but I can assure you this is not obvious from the name. Sometimes a better name will be more clear; other times, nothing short of a comment will clarify the purpose of something.

### 7. Use comments. Appropriately. ###

I often write documentation for a class before its code, which leads to clear thinking and good separation of concerns. (However, I must remember to review the documentation during the commit process, because I usually tweak my planned implementation midstream, potentially making the comment wrong from the very beginning!)

Documentation of large-scale entities (classes, modules) and mutable state (e.g. invariants) is often more important than documentation of small-scale entities (small functions and code blocks).

A lot of code is self-documenting and doesn't really need comments, although a short comment to summarize a code block can let someone avoid the painstaking work of actually reading the code. The reason to avoid comments is that when people update code, they often forget to change the comments to match the new code.

However, code can only explain _what code does_. Comments are needed to explain _why_: what _purpose_ the code serves, or why it is being done _in a weird way_, or why it is seemingly being done in the wrong module, or why it is doing something that seems redundant or unnecessary, or why X must happen before Y. **Use comments to explain why**, and to teach other developers when and how to use a component or function.

### 8. Think long and hard about your decisions. ###

Agonize over them. **You'll never come up with an optimal design by doing the first thing that comes to mind.** Even after you implement it, be on the lookout for a better approach (either so you can use it in the future, or ideally, improve the original code).

### 9. Keep improving ###

Be on the lookout for new and better general-purpose techniques, and then actually use those techniques. For example, I am slightly irritated whenever I see professional developers using strings when they should be using `Symbols`. Why? Do you use strings in C# because it doesn't have Symbols? But you've seen Symbols in Ruby and ES6 - take the lesson you learned in the other language and apply it to C#.
