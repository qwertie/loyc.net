---
title: "My best month in years"
layout: post
toc: false
tagline: "What causes motivation? Damned if I know!"
commentIssueId: 130
---

I've just had the most productive month since I was laid off in April, and the most productive month _without pay_ in years. Why?

![](2020-github-contributions.png)

It's hard to say exactly, but a number of factors came together.

First and foremost, moral support by [Maksim Volkau](https://github.com/dadhi) was invaluable. Thank you so much! Sometimes people complain about how users of open-source projects can be demanding and not contribute anything back, but I think that demanding users are not nearly as bad as having no users at all. I make open source software for one reason: I want to make the world a better place. Simple as that. The fact that I have no users and no participants is therefore a massive hole in my life. [Jonathan van der Cruysse](https://github.com/jonathanvdc) used to be a heavy user, but his work tapered off a couple of years back for reasons unknown to me, so it was a big boost to have Maksim come in with some bugs and feature requests. On very, VERY rare occasions I get nice message from a lurker, but only a bug report can prove someone's actually using my software. So, to those rare individuals who step up with a bug report, thank you so much.

Another factor is that I quit looking for a job. Suffice it to say that I am giving up a bunch of EI income in order to do this.

The third factor was that I wanted to make an Blazor version of LeMP, so users could try it online, but first I wanted a macro for building other macros, which the online version would definitely need. For this macro to be as powerful as I wanted it to be, I had to add some new features to LeMP, which was more work than expected - I ended up making a set of related general-purpose classes with names like [`CollectionWithChangeEvents`](http://ecsharp.net/doc/code/classLoyc_1_1Collections_1_1Impl_1_1CollectionWithChangeEvents.html), because I wanted to use _just one_ of those classes for an internal design I thought I wanted to use in LeMP - I decided to also write very similar classes ([`ListWithChangeEvents`](http://ecsharp.net/doc/code/classLoyc_1_1Collections_1_1Impl_1_1ListWithChangeEvents.html) and [`DictionaryWithChangeEvents`](http://ecsharp.net/doc/code/classLoyc_1_1Collections_1_1Impl_1_1DictionaryWithChangeEvents.html) for completeness. But ultimately didn't need these classes in the final design, so basically I wasted my time. Still, the classes I built and unit-tested will no doubt be useful for someone, someday. I hope. And then of course I [added a `macro` macro](https://github.com/qwertie/ecsharp/commit/3e353a6664d92297d40cc66fcc95d29bdad98345?branch=3e353a6664d92297d40cc66fcc95d29bdad98345) - there was already a `define` macro for doing simple syntactic transforms, but `macro` gives you the full power of the C# language. It's been possible since the beginning to define your own macros, but other than Jonathan I don't know of anyone who bothered.

### Old And Busted ###

![define change_temporarily macro](change_temporarily.png)

### New Hotness: complex compile-time logic ###

![](new-hotness.jpg)

~~~cs
macro change_temporarily($lhs = $rhs)
{
    using Loyc.Ecs;
    
    var tempOld = LNode.Id("old_" + context.IncrementTempCounter());
    
    matchCode(lhs) {
      case $prop.$member:
        // Avoid evaluating a property or function call twice in the output.
        // Also support `ref` attribute: it causes the prop var to be a ref var.
        var @ref = LNode.List().AddIfNotNull(node.AttrNamed(EcsCodeSymbols.Ref));
        unless (@ref.IsEmpty && prop.IsId && prop.Name.Name.ToLower() == prop.Name.Name)
        {
            var tempProp = LNode.Id(EcsValidators.KeyNameComponentOf(prop).Name
                                    + context.IncrementTempCounter());
            var propRef = quote([$(..@ref)] $prop);
            return quote {
                [$(..@ref)] var $tempProp = $propRef;
                var $tempOld = $tempProp.$member;
                $tempProp.$member = $rhs;
                on_finally { $tempProp.$member = $tempOld; }
            };
        }
    }
    // In simple cases, use the old logic
    return quote {
        var $tempOld = $lhs;
        $lhs = $rhs;
        on_finally { $lhs = $tempOld; }
    };
}  

void Example() {
    ref change_temporarily(GetParent().Mode = Mode.Awesome);
    AwesomeModeIsTemporarilyActive();
}

// Generated code (LeMP v2.9.0.1)
void Example() {
	ref var GetParent12 = ref GetParent();
	var old_11 = GetParent12.Mode;
	GetParent12.Mode = Mode.Awesome;
	try {
		AwesomeModeIsTemporarilyActive();
	} finally {
		GetParent12.Mode = old_11;
	}
}
~~~

I also wanted to release semantic version 29 before the web version of LeMP, and to to that I had to finish any breaking changes I wanted to do for version 29, and one thing led to another...

- I had to finish custom literal handling for LES2 and LES3, and this called for a [new feature](https://github.com/qwertie/ecsharp/commit/9581238e3a5af62ca343053c9ace9f4e7d8d551e?diff=split) in the language-agnostic [Token](http://ecsharp.net/doc/code/structLoyc_1_1Syntax_1_1Lexing_1_1Token.html) structure, something I later called ["uninterpreted literals"](https://github.com/qwertie/ecsharp/commit/ccb86e50ba6dd7f48072fc9e1c227c5566985adc) (which means "a literal represented by a type marker and a string without escape sequences that has not been parsed into its actual data type, unless that data type happens to be a string, of course").
- On a whim, I wanted to investigate how to support non-ASCII identifiers and operators which caused me to build an awesome page that displays [every unicode character in existence](http://david.loyc.net/misc/CharCategories.html). Emojis as identifiers? I'm thinking yes.
- To make macros play better with token lists, I added an ability for LES3 token lists to be interrupted and resumed within a single argument list](https://github.com/qwertie/ecsharp/commit/15f0baae7ac7c8846596f891105e0e3279d6a478).
- I [stripped out support for `NodeStyle.OneLiner`](https://github.com/qwertie/ecsharp/commit/e8c05695b55482090386ca02bba0b5d60913974f) because that whole feature was a premature optimization I shouldn't have done in the first place.
- [Tweaked collection interfaces](https://github.com/qwertie/ecsharp/commit/56a6a004361bfc76e83d70796b43c860c1626960) (again) and other interfaces
- [Added `LNodeRangeMapper`](https://github.com/qwertie/ecsharp/commit/1ec9af2b04f62266a78a8ca00a119e0b5411d747) so that [`compileTime` and `macro` can figure out](https://github.com/qwertie/ecsharp/commit/ba629f12b9cb9d0df659c17803ad9045cec0da20), given an error location in the C# output, the location of the error in the original Enhanced C# code.
- [Added `PreviousSiblings` and `AncestorsAndPreviousSiblings` to `IMacroContext`](https://github.com/qwertie/ecsharp/commit/22b699b088e5ad669e167ecdedac74ebdda92c57) so that [`macro` can scan _above_ the macro definition for `using` statements](https://github.com/qwertie/ecsharp/commit/74879b8ae44872481c57f28197243f9ace3a7e6c)
- I wanted to figure out the best way to write printers. Step one: [post musings on an old thread](https://github.com/qwertie/ecsharp/discussions/89#discussioncomment-180845). Step two: I noticed I had implemented two separate printer-helper classes and decided to combine their features in one: goodbye `DefaultNodePrinterWriter` and `PrinterState`, hello [`LNodePrinterHelper`](https://github.com/qwertie/ecsharp/commit/f76d487b945a3c35585436cdc20858bfc7bb331b)! Step 3: incorporate it into all three existing printers. Step 4: delete `DefaultNodePrinterWriter`.
- For fun I [added an `--eval` command-line option](https://github.com/qwertie/ecsharp/commit/40c2bff9d5511da709418e0dac8ff61efc6399a0)
- [Laboriously convinced C# to understand `.TryGet(i)`](https://github.com/qwertie/ecsharp/commit/41d3a2f4cf1d290f5e0168ad9b3442dd67dc2f70) on a type-by-type basis
- [Added `#preprocessArgsOf`, `#preprocessChild` macros](https://github.com/qwertie/ecsharp/commit/cf82baccb2da1027e71bcf896cd3885febd4c8f2) to help with those pesky macro order-of-operation issues.

Over at [Future of Coding slack](https://futureofcoding.org/community), I wanted to post "two-minute week" for the past month, so I started thinking of demos to show. And I thought I wanted a demo that used a `switch` expression. Problem: EC# doesn't support switch expressions yet. Solution: [implement those too](https://github.com/qwertie/ecsharp/issues/129). Problem: C# 8 switch (and `is`) patterns are [poorly documented,  hideously ambiguous](https://github.com/qwertie/ecsharp/blob/master/Main/Ecs/Parser/EcsParserGrammar.les#L1014), incompatible with LL(k), and, dare I say, [worse than my older Enhanced C# design](https://github.com/qwertie/ecsharp/blob/master/Main/Ecs/Parser/EcsParserGrammar.les#L1403) in some ways. Solution: spend 2 full days on it.

Of course, Enhanced C# is a liberalization of C#, so the obvious question was which features of C# 9 patterns should be included throughout EC#. I decided against adding unary relational operators (like `< 0`) everywhere, but in favor of adding the `when` operator everywhere, as well as a variant of the `switch` operator that is _not_ followed by braces.

And then I thought "why stop there?" and added a `where` operator too, because I think some people (e.g. me) would be interested in writing Haskell-style code where you write your code top-down instead of bottom-up. Take, for instance, the code in `compileTime` for error handling when you give it some invalid code. What I _wanted_ to write was some code to print an error message:

~~~cs
context.Sink.Error(errorLocation, "{0} - in «{1}»"
        .Localized(e.Message, lineOfCodeFromOutput));
~~~

But first, I need to figure out the `errorLocation` and a `lineOfCodeFromOutput` to include in the error message. At this point programmers usually insert code **above** this point to retrieve the desired information. I'm thinking, **no, that harms code readability!** Instead what we should do is write the code in a more natural top-down order:

~~~cs
// (I'd prefer the shorter syntax $errorLocation, but 
// `var` is what I think the Roslyn team would choose)
context.Sink.Error(var errorLocation, "{0} - in «{1}»"
  .Localized(e.Message, var lineOfCodeFromOutput)) where
{
    Microsoft.CodeAnalysis.Text.TextSpan range = e.Diagnostics[0].Location.SourceSpan;
    var C#Location = new IndexRange(range.Start, range.Length);
    var errorLocation = outputLocationMapper.FindRelatedNodes(C#Location, 10)
        .FirstOrDefault(p => !p.A.Range.Source.Text.IsEmpty);
    lineOfCodeFromOutput = codeText.Substring(var lineStart, (var lineEnd) - lineStart) where {
      int column = e.Diagnostics[0].Location
        .GetLineSpan().StartLinePosition.Character;
      lineStart = range.Start - column;
      lineEnd = codeText.IndexOf('\n', lineStart);
      if (lineEnd < lineStart)
        lineEnd = codeText.Length;
      string line = codeText.Substring(lineStart, lineEnd - lineStart);
    }
}
~~~

The new release won't actually include the macro necessary to use this feature, but it's a macro that you (or anyone else) could write yourself. You don't even need the new version! Enhanced C# already has a "user-defined operator" feature, you just have to write ``expr `where` {...}`` instead of the slightly better-looking ``expr where {...}``.

Anyway, while working on C# 8/9 pattern matching I realized that if I didn't want to write a bunch of ugly custom parser code _again_, I would need some kind of new feature in my [parser generator](http://ecsharp.net/lllpg/) to give me better control over code generation for recognizers. So I spent two days adding new LLLPG operators called `recognizer` and `nonrecognizer` to help me out.

And that's where we are today. In the coming days I'll be producing videos for Future of Coding, figuring out how to write printers more effectively by implementing a printer for TypeScript, and releasing semantic version 29 of [Enhanced C#](http://ecsharp.net/), [LeMP](http://ecsharp.net/lemp/), [LLLPG](http://ecsharp.net/lllpg/) and the [Loyc .NET Libraries](http://core.loyc.net/).

Did I somehow regain my hope — hope that more work would lead to renewed interest in my software and my ideas? Maybe. Hope is a fragile thing. [**Edit:** on second thought, it wasn't fragile when I started the EC# project 8 years ago; time has a way of wearing me down.] I can feel the despair clawing at my psyche from below. A couple of times in the past month, I remembered the bad times, like when I would write a ten-page article on a new feature, post it to Reddit and get two upvotes and zero traction. So far I've managed to refocus my mind on more positive thoughts. I worry that my success won't last much longer, but I am confident that I will at least manage to post the videos, release v29 and continue work on the web version of LeMP.
