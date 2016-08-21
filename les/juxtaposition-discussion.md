---
title: "LES juxtaposition idea"
layout: article
toc: false
---

Out of date: some syntactic elements have changed since this was written; see [the new plan](http://loyc.net/2016/lesv3-and-wasm.html). Last update: June 14, 2016

This document explores a potential new version LES with a different set of syntactic sugars than LESv2 has. It is more complex than LESv2, but it satisfies two desires that LESv2 does not:

### "I want no separate '`(`' and '` (`' tokens.", and ###

### "I want to write `x = new Foo()` or `i32.reinterpret_f32 $N` without parentheses ###

To satisfy these desires, we could throw out the concept of superexpressions and start fresh. From now on, assume '`(`' and '` (`' are now always equivalent.

### The juxtaposition operator ###

To satisfy the second desire, we could define a Haskell-inspired "juxtaposition" operator. This operator would have a very high precedence, but not as high as `.`. In Haskell, `foo x y` is the idiomatic equivalent of `foo(x, y)`, although it literally means `(foo(x))(y)`.

I suppose `foo x y` could mean `(foo(x))(y)` or `foo(x(y))` or even `foo(x, y)`, depending on how the parser is designed. So let's consider the scenarios:

 - `i32.reinterpret_f32 $N` should mean `(i32.reinterpret_f32)($N)`. It does.
 - Ideally, `x = new Foo()` would mean `x = (new(Foo()))` - but since we're eliminating whitespace sensitivity, `new Foo ()` would mean the same thing, which implies it wouldn't work quite the same as Haskell's juxtaposition operator - not _necessarily_ because the operator itself is any different, but because Haskell doesn't have a separate "call-with-parens" operator. By the way, note that `x(y)(z)` _must_ continue to mean `(x(y))(z)`, since that's what it means in all mainstream languages.
- `x y z` could mean `(x(y))(z)` (left associative) or `x(y(z))` (right associative). The first interpretation follows what Haskell does, while the second is the logical interpretation if we want to treat `x` and `y` as unary operators (as in `log sqrt $N` or `not isHappy camper`)... **and that seems to be what we want here**.

To make this work, we could introduce the juxtaposition operator with a precedence somewhere between `**` and `::`. This operator would work as a backup to the normal function-call operator, and would only apply when the normal function operator doesn't. That is, if you write `f (a.b).c`, it's a normal function call parsed as `(f(a.b)).c`. However, if you write `f a.b.c`, it's a juxtaposition parsed as `f((a.b).c)`. A related case is `a [i]`, which would be an indexing operation; you would have to write `a([i])` instead in order to pass a list `[i]` to `a`.

Examples:

| Input                    | Meaning
| ------------------------ | ------------
| `f a.b * c`              | `(f(a.b)) * c`
| `f(a.b) * c`             | `(f(a.b)) * c`
| `f (a.b).c`              | `(f(a.b)).c`
| `f a.b.c`                | `f((a.b).c)`
| `i32.reinterpret_f32 $N` | `(i32.reinterpret_f32)($N)`
| `x = new Foo()`          | `x = (new(Foo()))`
| `x y z`                  | `x(y(z))`
| `(Foo) x`                | `Foo(x)`

Juxtaposition would have the curious effect of making `(int) x` (the cast operator of C languages) equivalent to `int(x)`, which is an ordinary call but also acts as a cast operator in other languages including C++. So that seems ... okay.

### Block calls

Already, juxtaposition allows expressions like `if {...} else {...}`; unfortunately, such an expression would have the strange tree structure `if( ({...})(else({...})) )`. That's weird, so we should add a syntactic rule for braces `{...}` to get the kind of tree we want.

So let's add braces `{...}` as an optional add-on to the ordinary method call syntax, so that `foo (x, y) {z}` means `foo(x, y, {z})`. Also, we'll need a "continuator" feature so that `if (c) {a} else {b}` gives us a reasonable tree, maybe `if(c, {a}, else, {b})` (like LESv2) or `if(c, {a}, else({b}))`. To be clear, this feature would be completely independent from the juxtaposition operator. A continuator could perhaps be defined as a word from a pre-set list, followed by something in parentheses and/or braces, to be added as an additional argument to the original call.

So our expression grammar with juxtaposition plus "block calls" (braced blocks added to calls) might contain rules that look roughly like this:

    // Here "$" represents _all_ of the maximum-precedence unary operators
    Particle : Identifier | Literal | Parentheses | BracedBlock | "$" Particle;
    // Here "." represents _all_ binary operators at the primary level 
    PrimaryExpr: Particle [ "." Particle | CallArgs ]*;
    CallArgs : ArgList [BracedBlock Continuator*]? ;
    BracedBlock: "{" StatementList "}";
    Parentheses: "(" ExpressionList ")";
    Continuator : ContinuatorKeyword (BracedBlock | ArgList BracedBlock?);
    ContinuatorKeyword : "else" | "catch" | "except" | "finally" | /* more continuators TBD */;
    Juxtaposition: PrimaryExpr Juxtaposition;

To reduce potential confusion, we could restrict the left-hand side of the juxtaposition to be a simple identifier or particle:

    Juxtaposition: Identifier Juxtaposition;
    Juxtaposition: Particle   Juxtaposition;

Then confusing expressions like `if (c) {a} else {b} foo` would be illegal, and `(Foo) x` would be illegal if we picked the first version of the rule. However, this restriction would also prohibit things like `f32.sqrt` which, due to the `.` operator, are not particles. It would be possible to carve out an exception to allow that, though.

The result is nice, because it allows C-like expressions such as 

- `switch (y) {...};`, meaning `switch(y, {...});`
- `if (x) {...} else {...};`, which could mean `if(x, {...}, else, {...});` or `if(x, {...}, else({...}));`.

You could also write this:

    x = switch (y) { 0 => "zero"; 1 => "one"; };

which was illegal in LESv2.

Unfortunately, the design as described produces this odd syntax tree for `try`:

| Input                               | Meaning       
| ----------------------------------- | --------------
| `try {...} catch {...}`             | `( ( try({...}) )(catch) ) ({...})`

To solve this, we could treat `{...}` as an alternative to the ordinary call syntax, by modifying the `CallArgs` rule above:

    CallArgs : (ArgList [BracedBlock Continuator*]? | BracedBlock Continuator*) ;

So now we have 

| Input                               | Meaning       
| ----------------------------------- | --------------
| `loop {x}`                          | `loop ({...})`
| `try {...} catch {...}`             | undecided: could be `try({...}, catch, {...})` or `try({...}, catch({...}))`

And if arbitrary continuators were allowed, you could also write `do {...} while (condition)`. I'm not sure yet if that's a good idea.

### Downsides ###

The first major downside to this design is that statements like 

    var x = z * z;
    return x + y;

Change their meaning to

    (var(x)) = z * z; // in LESv2, it was `var(x = (z*z))`
    (return(x)) + y;  // in LESv2, it was `return(x + y)`

I think `(var(x)) = z * z` is actually a perfectly reasonable syntax tree for a variable declaration; who says initialization has to be part of variable creation? The `return` statement, however, has become nonsensical. It's not a bad sacrifice, I think: writing `return (z*z)` isn't that onerous, and for programmers that accidentally write `return x + y` it would be possible to give a helpful error message. Perhaps we could add a special parsing rule analogous to the Haskell binary `$` operator (in Haskell, `return $ x + y` means `return (x + y)`, although it's a feature of the standard library, not the parser.)

The worst problem IMO is what happens to function and type declarations:

| Input                             | Strange Meaning
| --------------------------------- | -------------------------------------
| `fn Foo(x: i32) {...}`            | `fn( (Foo(x: i32, {...})) )`
| `fn Foo(x: i32) -> i32 {...}`     | `fn( (Foo(x: i32)) -> (i32({...})) )`
| `class Foo {...}`                 | `class( Foo({...}) )`
| `class Foo : IFoo {...}`          | `class( Foo : (IFoo({...})) )`

Unlike the current version of LES, which associates the braces with `fn` or `class` in these examples (as it should), this hypothetical LES associates braces with the return value, the base class, or whatever else happens to be located on the right side. Yuck.

### Keyword statements

Luckily, there is one more thing we could add to make this work: we could introduce a "keyword" concept, in which keywords are explicitly marked by adding `#` on the front. These keywords would appear at the start of a new type of "superexpression" that works similarly to superexpressions in LESv2. This gives us a new way to support a `return` statement:

| Input                             | Meaning       
| --------------------------------- | -------------------------------------
| `#fn Foo(x: i32) {...}`           | `@#fn(Foo(x: i32), {...});`
| `#fn Foo(x: i32) -> i32 {...}`    | `@#fn(Foo(x: i32) -> i32, {...});`
| `#class Foo {...}`                | `@#class(Foo, {...});`
| `#class Foo : IFoo {...}`         | `@#class(Foo : IFoo, {...});`
| `#return x + y`                   | `@#return(x + y)`

Here, the `@` simply indicates that identifiers like `#fn` are not to be treated as keywords.

In LESv2, `#` is an ordinary identifier character, treated no differently than `_` or a letter, but by convention it is used to represent keywords in Loyc trees. So it's a very appropriate choice to use `#` here to denote a "keyword statement".

But `#` is a "heavy-looking" character - it draws attention to itself, and is also clumsy to write on a whiteboard. Perhaps a lighter alternative is better? We could use a single quote, because a single quote is normally used for character literals - but by definition a character is only _one_ character, whereas every keyword I've ever seen is at least two characters, so there's no ambiguity.

| Input                             | Meaning       
| --------------------------------- | -------------------------------------
| `'fn Foo(x: i32) {...}`           | `@'fn(Foo(x: i32), {...});`
| `'fn Foo(x: i32) -> i32 {...}`    | `@'fn(Foo(x: i32) -> i32, {...});`
| `'class Foo {...}`                | `@'class(Foo, {...});`
| `'class Foo : IFoo {...}`         | `@'class(Foo : IFoo, {...});`
| `'return x + y`                   | `@'return(x + y)`

Parsing this requires a little hack: the expression parser needs to be told it's in "superexpression" mode so that it can stop at the braced block, which it would ordinarily consume.

The grammar would be something like this:

    Superexpression : Keyword ExpressionWithoutBraces? BracesWithContinuators?;
    BracesWithContinuators : BracedBlock Continuator*;
    Continuator : ContinuatorKeyword (BracedBlock | Parentheses BracedBlock?);
    BracedBlock: "{" StatementList "}";
    Parentheses: "(" ExpressionList ")";
    Continuator : "else" | "catch" | "except" | "finally" | /* more continuators TBD */;

### Should there be a keyword list?

Unfortunately, the `#` (or `'`) would make LES look different from most C-like languages. If it's super important for the code to look "natural", we could introduce a set of keywords to cover the most common cases where this syntax would be needed: `fn function proc property struct class enum interface type data template trait alias namespace` - with two function keywords to cover the Javascript and Rust camps, and perhaps `proc` for good measure. These keywords would be treated as if they started with `#`/`'`, and ordinary identifiers with those names could be specified with `@`, i.e. `class` means `#class` or `'class`, and `@class` means `class`.

We could also add a few others like `return import using case throw`, but we have to draw the line somewhere since it's impossible to specify a set with enough stuff to satisfy everyone.

### Operators with letters in them

If we use `'` to identify keywords, we could also use it to identify binary operators with letters in them. I would propose in fact that _all_ binary operators include a single quote in the Loyc tree, [as I discussed in March](http://loyc.net/2016/put-back-the-sharp.html).

Then WebAssembly could have its signed and unsigned operators at the cost of one extra character:

    $x = $y '>s $z // signed comparison
    $x = $y '>u $z // unsigned comparison

Or with the letter first:

    $x = $y 's> $z // signed comparison
    $x = $y 'u> $z // unsigned comparison

And it would be a reasonable way to write "word operators":

    Dinner = pizza 'with anchovies 'and stuff;

However, using `'` this way has a small price: it requires a space _before_ the `'` as well as after, because `'` is also a legal character in identifiers (an idea taken from Haskell). Therefore, the existing syntax with backticks can actually be more compact:

    Dinner = pizza`with`anchovies`and`stuff;

On the whole I like this idea, because the `'` could be defined as the way to identify operators and keywords in the syntax tree. If `'` in the syntax corresponds to a `'` in the AST, it makes LES easier to learn and understand.

### Minor points

A problem that re-emerges in this proposal is that you need a semicolon at the end of your block-call expressions like `switch (x) {...};`. To fix this, we could specify that if the _leftmost_ expression is a call expression that ends in a braced block, no semicolon is needed. I'm a bit concerned about the complexity of implementing such a rule, but we won't need it if we use newlines as the main terminator instead of semicolons.

If we're going to define keywords, then certainly we should define `null`, `false` and `true` (rather than using `@null`, `@false` and `@true` as in LESv2).

In LESv2 with superexpressions, there was an ambiguity in `A - B` between the normal infix interpretation and the superexpression interpretation `A (-B)`. Perhaps this is why Haskell only has a single prefix operator in total - to limit the probability that someone would erroneously write things like `f !x` or `f ~x`, expecting their punctuation to be treated as a prefix operator. However, I don't think our new juxtaposition operator has an ambiguity related to this, nor does Haskell (because the precedence of the RHS of the juxtaposition excludes prefix operators).

Currently in LES, `(a; b)` is a tuple. Arguably, `f (a.b; c)` should be illegal because it's not clear if this was meant to be a normal function call or a juxtaposition-call with one parameter that happens to be a tuple.

### Conclusion

At first I didn't like this plan, but now I'm relatively happy with it. It adds significant complexity, but also significant value.
