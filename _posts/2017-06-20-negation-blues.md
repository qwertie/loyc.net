---
title: "Negation blues"
layout: post
toc: false
---

There's a nasty little difficulty in the design of LES: negative literals.

The fundamental problem is, LES must represent Loyc trees faithfully, so we _need_ some way to represent negative literals such as `-1`. These literals _must_ be distinct from _negated literals_ such as `- 1`, meaning, the unary operator `'-` with a positive literal argument.

How do we do this?

My first design was to simply treat `-1` as a literal, requiring a space after `-` if you wanted it to be interpreted as an operator.

This is how LESv2 still works.

But that was a sucky design, because you couldn't write `x-1`: it would be a syntax error. I mean, the identifier is followed immediately by a literal. What could that possibly mean?

An alternative is for the lexer to produce only positive numbers, and then have the parser negate them if there happens to be a `-` beside the number. The main problem with this plan is that you can write, for example, `-0x8000_0000i32`. But `0x8000_0000i32`, without the minus, can't be stored as a positive 32-bit integer.

So in LESv3 I introduced a special `NegativeLiteral` token type. The lexer would parse things that look like negative literals as numbers, and then use `NegativeLiteral` as their token type. Then in the parser, if `NegativeLiteral` appears where a binary operator was expected, a special negating function is called to convert the negative value to a positive one, and then the minus sign is reinterpreted as the binary `'-` operator. Therefore, `x-1` parses successfully.

Then I made my [graphing calculator with LES](https://github.com/qwertie/GraphingCalculator), and I found out that this approach is wrong. While playing with it, I wrote an expression with an interesting graph, then I added a space (after a `-`) and it completely changed the result.

The problem was that I interpreted `a -2 ** x` as `(a - 2) ** x` when it should be `a - (2 ** x)`. It's not easy for the parser to interpret this expression correctly, because the (immutable) token list represents `-2` as a single token, so there's no easy way to make the subexpression parser see the sequence `2 ** x`.

So I reconsidered my whole approach. It was clunky to parse negative numbers in the lexer and then negate them in the parser, especially since it involved a long series of type tests (is it `Int32`? Here's how to negate `Int32`. Is it `Int64`? Here's how to negate `Int64`. Is it `Single`? etc.)

So then I thought, well, LES has Custom Literals. If the user writes `-0x8000_0000i32`, we can store `0x8000_0000i32` as a custom literal in its original string form and then maybe parse it later, in the parser. 

This is really clunky too, having both the lexer and parser in charge of parsing integers, but at least the lexer would no longer have to deal with negative numbers.

So, I stripped out the code that parses the minus sign at the beginning of integers in the lexer, since now it's the parser's job. Right? Wrong.

I had forgotten. The plan for LESv3 was to unify numeric and string literals into a single concept. For example, `i32"1234"` and `1234i32` should produce identical Loyc trees. This also means that you can write `i32"-1234"` and get a negative number out of it. So it seems the lexer _still_ has to deal with negative numbers occasionally.

One possible solution is represent _every_ literal in the lexer as a `CustomLiteral` object, and let the parser handle all literal parsing. This would eliminate the awkwardness of sharing responsibility for negative numbers between the parser and lexer. Unfortunately, this would raise the number of memory allocations: originally we just needed one memory allocation per literal (the boxing overhead), except for one-digit numbers which were cached and used zero allocations. This solution needs three allocations per literal: one for boxing, one for a `CustomLiteral` object, and at least one more for either a string holding the text of the literal, or a boxed `UString` pointing to the text. We can't even null-out the `Value` when we finish processing a token, which would help garbage collection. That's because the parser treats the token list as immutable (it's `IList<Token>` because it's derived from LLLPG's `BaseParserForList<,>`,  but it's likely to be read-only. And by the way, `IParsingService`, which `Les3LanguageService` implements, separates lexing and parsing, so a user can modify the output of the lexer or supply a custom token list to the parser.)

It's not a good idea, either, to leave the `Value` of the `Token` blank and then parse strings and numbers entirely in the parser. That would imply throwing away and reconstructing information that the lexer already gathered. The lexer knows where the number begins and ends, if it's a number, and if it's a string, it knows whether it contains escape sequences that need to be transformed. It knows where the type marker, if any, begins and ends. By contrast, the parser only gets a single range of characters.

We can't expand the `Token` structure used to communicate from the lexer to the parser, because it is a fixed-size structure designed for all languages (Roslyn gave me that idea). Using any other token type would make LES incompatible with the `IParsingService` interface. Anyway, the .NET Framework guidelines recommend against making large structures.

So now I'm thinking, maybe it's time to give up on automatically including the `-` as part of the literal. Instead, I'm thinking negative literals could be represented in two ways:

1. With string notation: `number"-1"` or `i"-1"`
2. With U+2212, the minus character. This would allow `−1` to be expressed more compactly, though unfortunately the difference is almost invisible to the eye _and_ there's typically no way to type this character. The lexer would have to recognize both minuses, `-` and `−`, since a dash can still be used for negation inside a quoted literal.

This plan isn't very satisfying, but it is at least straightforward to implement.

The only other possibility that looks viable is to use `CustomLiteral` only occasionally in the lexer, and use normal boxed values most of the time. So `-7` would be represented with token values `CodeSymbols.Sub` followed by `7`, but `-0x8000_0000i32` would be represented as `CodeSymbols.Sub` followed by `CustomLiteral`. Unfortunately this solution is more complex, as the lexer parses some integers and the parser parses others. And what about `-0x8000_0000`? It would be tough to make that end up as `Int32`; it would likely end up as `Int64`.

And what should the official LESv3 spec say about all this?