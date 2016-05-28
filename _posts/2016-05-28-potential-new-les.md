---
title: A potential new LES
layout: post
toc: true
commentIssueId: 30
tagline: for great LISPification
---
I've proposed that [WebAssembly adopt LES as the basis for its text format](https://github.com/WebAssembly/design/issues/697) (or at least, to constrain the Wasm text format such that LES is a superset of it.) As part of that proposal, I've agreed to modify LES to suit the tastes of CG members. So far, only a couple of people have weighed in; in the meantime, I've been thinking preemptively about what changes to LES might make it more liked as a text format.

This document will take a while to read, but is designed to require only a passing familiarity with [LES](http://loyc.net/les/).

But before I talk about a new version, let's discuss what works well in the current version (LESv2) and then I'll mention the pain points I've noticed when using LES, and potential issues for LES + WebAssembly.

**A reminder about notation**: syntax trees will be expressed as simple LESv2 without superexpressions. For example:

- `f(x)` or `(f)(x)` are call nodes in which the target is `f` and the argument list has length one, with item `x` inside.
- `x + f(x)` is a call node in which the target is an identifier called `+` and there are two children, `x` and `f(x)`. `@+(x, f(x))` is the same tree, since `@` is used to parse operators (or mixed punctuation and letters) as identifiers.

What works well in LESv2
------------------------

The basic expression stuff works well and there's no reason to change it:

- Identifiers `abcd`, literals `1234` and calls `foo(bar, baz)`, with 
  - escaping mechanisms for identifiers so they can contain any unicode character (or _no_ characters)
  - optional suffixes to indicate numeric types
  - single-, double-, and triple-quoted strings.
- Infix binary operators (`+ - >>`...), prefix operators (`$ - ~`...), suffix operators (`++`...), and an infinite number of operators with a fixed rule-based precedence for each.
- Tuples `(_;_;_)`, Lists `[_,_,_]`, indexing `foo[bar]`
- Braced blocks `{...}`
- Keyword-free language: since LES is language-agnostic, it shouldn't have keywords that only make sense in a small number of languages. But it seems like there's little use in having an overly short, fixed list of keywords, since there would not be enough keywords to meet _all_ the needs of a typical language. I could have defined a mechanism to define keywords (or entire syntactical rules), but that would mean that part of the file could be parsed differently depending on code that appeared much earlier. Instead, I chose a keyword-free design, which is nice because it keeps the lexer and parser simple, and also guarantees that code from one file can be copy-pasted to another file without changing its meaning or causing a syntax error. Finally, it guarantees that a human reader (who knows the rules of LES) can mentally parse a snippet of LES without reference to any other code. These are all significant advantages that I'd like to keep.

Also as long as the empty statement '`;`' is allowed, it seems reasonable to also keep empty expressions (e.g. `Foo(4,)` takes two arguments, the second being the empty identifier), unless perhaps we switch the tuple syntax to use commas (in the interest of concision, I won't explain the issue here).

Existing pain points in LESv2
-----------------------------

 - Although LES ignores newlines, it does have a small amount of whitespace sensitivity since "` (`" (with a space before it) is a separate token from "`(`" with no space. The two tokens '` (`' and '`(`' distinguish normal expressions from superexpressions (i.e. expressions that look _as if_ they start with a keyword, like `struct Foo {}`). For example, `return (x+y)*z` is parsed like `return((x+y)*z)`, whereas `return(x+y)*z` is parsed like `(return(x+y))*z`.

   This is not a problem when reading LES code, since readers who are unaware that there are two tokens may never even notice the difference. So for Wasm this shouldn't be a problem, as Wasm is usually read, not written.
    
   For humans writing LES, mixing up '` (`' and '`(`' _usually_ (not always) produces a syntax error (e.g. `function foo (x:i32)`) or simply works as intended (`foo(x);` and `foo (x);` should always have the same semantics, although the distinction is recorded as trivia).

 - Superexpressions may need extra parentheses: the expression `var foo = new Foo();` can't be parsed; you must write `var foo = (new Foo());` instead because a superexpression can appear only at the beginning of a subexpression ('`(`' starts a new subexpression). Again, this is not a major problem for reading code, but might be confusing for those who are starting to write LES. When it comes to Wasm, people who just want the best possible syntax (and don't care about LES) may be annoyed about needing extra parentheses.

 - Superexpressions don't play nice with certain prefix operators. For example, `if !x {...}` is a syntax error because `if !x` appears to be an identifier with a generic type parameter (`#of(if, x)`).

 - The most annoying thing I've noticed about LES is the need for semicolons after `}`. While I have learned not to write `if !...`, I still often write `}` rather than `};`, as old habits die hard.

 - Precedence of `!`: In of-expressions like `List!int` (list of int), I suspect the precedence of `!` should probably be increased. `a[b].c!d` is currently parsed as `((a[b]).c)!d` but probably be parsed as `(a[b]).(c!d)`. It wasn't initially clear which precedence was better.

 - No upgrade path has been planned to add arbitary new syntax for literals (byte strings? [unums](http://ubiquity.acm.org/article.cfm?id=2913029)?) without breaking backward & forward compatibility.

 - Arguably, there should be a unary suffix operator that can be an arbitrary string (this was intentionally punted to a future version.)

 - Since there are an infinite number of operators, you need a space between any two operators, such as `x + -y` or `-*y`. IMO, that's fine. Compiler writers just need to be careful to give a good error message if the user types `x*-y`, maybe something like "There is no operator called `*-`. Consider adding a space between the operators: `* -`. You could even postprocess the code to transform, for example, `-*ptr` into `-(*ptr)`, with or without printing a warning.

- In the output syntax tree, there's no quick and easy way to distinguish operators from normal identifiers. See my [earlier post](http://loyc.net/2016/put-back-the-sharp.html) about that.

Issues for WebAssembly
----------------------

- It seems being able to put any character into an identifier isn't good enough in WebAssembly, which also allows bytes that are _invalid_ UTF-8 - outside the realm of characters!

- Opcode names like `i32.reinterpret/f32` are not LES-compatible since `/` is an operator, of course. I don't think allowing `/` in an identifier is a good idea, and I see no particular reason to use `/` in the first place, so I propose the opcode name should change to `i32.reinterpret_f32` or `i32.reinterpret'f32`.

- Dan Gohman (a Wasm dev on team Mozilla) suggests operators without parentheses like `$y = i32.popcnt $x`, which is not possible in LES today, even for unary opcodes.

- He also suggests operators with _letters_ in them like `>s` which, of course, parses as two separate tokens in LES. This probably shouldn't change since it seems very wasm-specific; outside wasm, you would expect `r>s` to be parsed as `r > s`.

- Dan also prefers a language without semicolons. LES could be changed to work that way, and it may have an advantage, because we would no longer have to worry about accidental mis-parses caused by a forgotten semicolon. However, this change would also kill JSON compatibility since

        { "foo"
          : ["bar"] }

   would suddenly meaning something else.

- I made some syntax suggestions, for which the precedence must be considered. See "Precedence issues in WebAssembly" below.

Ideas to change LES
-------------------

### Dealing with identifiers containing invalid UTF-8 ###

I'm inclined to think this cannot be solved in a nice way, because (at least outside the Wasm world) it is not reasonable, from the standpoint of API usability, to return identifiers as byte arrays on platforms like Java and .NET that use UTF-16 strings. I think we can get a little wiggle room by exploiting orphaned surrogate pair characters, though, to guarantee round-tripping from arbitrary bytes to UTF16 and back, without affecting the conversion of normal UTF-8.

A special notation could be offered for invalid UTF-8, e.g. ``@`\?AA` `` for the byte string `"\xAA"` (I think the identifier ``@`\xAA` `` should refer to the character 0xAA rather than the byte 0xAA).

### "I don't want to write `;` after `}`" ###

It's easy to forget that semicolon at the end of `if (c) {...};`, and I plan the following rule to eliminate the need for it:

If the outer expression is a superexpression with an `AfterParticle` that is a braced block not followed by a semicolon, then the expression must end at the closing brace, as if a semicolon were present, **unless** the braced block is followed by a "continuator" identifier. A _continuator_ is either an identifier that starts with `@`, or one of a small set of words from a predefined list that includes `else`, `catch`, `finally`, and `except` (and others TBD). For example, `if c {...} else {...}` would be parsed as a single expression, whereas `if c {...} loop {...}` would be parsed as two independent expressions.

**Note**: this rule wouldn't apply to the superexpression's initial expression, so for example the closing brace in `do {...} while (foo);` does _not_ count as the end-of-statement, even though `while` is not on the list of continuators. Similarly `loop {...};` would require a semicolon, but `for (x : list) {...}` would not.

**Wait a minute...** it may seem odd that I rejected the idea of a fixed set of _keywords_ but now suggest a fixed set of _continuators_. The reason is that the set of _continuators_ used by most programming languages is far smaller and more predictable than the set of _keywords_: in some languages, the only continuators are `else`, `catch`, and `finally`. Also, continuators still aren't keywords.

### "I want no separate '`(`' and '` (`' tokens." ###

I have a couple of ideas for satisfying this desire, by replacing the current concept of superexpressions with something more... diversified. My first idea is to introduce three bits of syntactic sugar:

1. Block-call expression (adds an argument): `primary_expr {...}` and `primary_expr (...) {...}`
2. After a block-call expression, a "continuator" is permitted from a predefined set that includes `else catch finally where` or any identifier that starts with `#`. The code starting at the continuator is parsed as a primary expression, and added as an additional argument to the original call.
3. Top-level expr: an identifier followed by any expression that does _not_ start with `(` or an infix operator, e.g. `return 0`.

Plus, we can eliminate the need for semicolons with a similar rule to that described above.

The first and second rules let us write C-style executable statements, with or without a space after the "keyword":

~~~js
if (expr) {exprs;}
if (expr) {exprs;} else {exprs;}
try {...} catch(...) {...} finally {...}
for(...) {...}
switch(...) {...}
do {...} #while (...)
~~~

Note the need for `#` before `while`, because `while` is not a continuator.

You could also write things like this:

    x = switch (y) { 0 => "zero"; 1 => "one"; };

which is illegal in LESv2.

The third rule covers things like `var`, `new`, `return`, `break` and `import`:

~~~js
var foo = (new Foo()); // parentheses are required around `new` expression
break outerLoop;
import net.loyc.syntax.@*;
return a + b;
~~~

But users would have to understand that (unlike in LESv2) `return (a + b) * c` would have the unintended meaning `(return(a + b)) * c`.

This plan has a major limitation, as it provides no nice syntax for type declarations and function declarations. Things like this can still be _parsed_ (although semicolons are now required):

~~~
fn foo(bar: i32) {...};
fn foo(bar: i32) -> baz {...};
struct Foo {...};
struct Foo : IFoo!T {...};
~~~

But their syntax trees become a little weird:

~~~
fn(foo(bar: i32, {...}));
fn(foo(bar: i32) -> baz({...}));
struct(Foo({...}));
struct(Foo : ((IFoo!T)({...}));
~~~

Because of this, the plan is hard to endorse as-is.

Therefore, I investigated a more elaborate set of ideas (see the next section).

### "I want to write `x = new Foo()` or `i32.reinterpret_f32 $N` without parentheses ###

I developed a syntax that satisfies this desire along with the previous one, but it's a relatively complicated proposal so I've split it out onto its [own page](/les/juxtaposition-discussion.html).

### "I want operators with letters in them." ###

LES already has a mechanism for this: backticks. You can write ``x`foo`y``, which means `foo(x, y)`. But let's explore alternatives anyway.

As well as signed and unsigned operators in WebAssembly (`$x >s $y`), operator suffixes could provide an interesting way to create named operators, as in `f(x) :where x > 0` (meaning ``(f(x)) `:where` (x > 0)``) - but new users could get confused that `f(x) : where x > 0` is completely different (`(f(x)) : ((where(x)) > 0)`). The other downside is that we'd always need spaces between operators and their arguments. That's especially a problem for prefix and dotted expressions like `-x` and `foo.bar`. So if we really wanted to do this, we would need to compromise by saying that certain operators like `.` and `-` can't have suffixes.

To avoid these problems, we could have "escaping" of letters and words in operators. For example, if we select `\` as our escape character, then `>\s` would be an operator named `>s` and `\where` would be an operator named `where`.

Another possibility is `'`; this is discussed at the end of the [juxtaposition proposal](/les/juxtaposition-discussion.html).

Personally, though, I think backquotes are fine. I certainly hope Wasm developers will not get so hung up on a little punctuation as to reject the [wider benefits](https://github.com/WebAssembly/design/issues/697) of LES.

Whether we stick with backquotes or not, one remaining issue is the precedence of operators that contain letters. Currently, all backquoted operators have the same precedence, which is immiscible with many other operators (e.g. ``a `foo` b + c`` is illegal, because it's unclear if you meant ``(a `foo` b) + c`` or ``a `foo` (b + c)``). Since an operator like `>s` has a normal operator embedded inside, perhaps the initial punctuation characters should be used to decide the precedence of the operator.

### "I don't like semicolons. Let's use newlines instead." ###

LES could be changed to work that way, and it may have a significant advantage, because we would no longer have to think about accidental mis-parses caused by a forgotten semicolon, and we wouldn't need any "semicolon insertion" rules. However, this change would also kill JSON compatibility since

    { "foo"
      : ["bar"] }

would suddenly meaning something else.

If newline is a terminator, its effect should be nullified after a line of whitespace, or an open brace, or inside parentheses...

    {              // newline is ignored here
      Foo(x + y,   // newline is ignored here
          a + b)   // newline is a terminator
    }              // newline is a terminator

unless, of course, the user opened braces inside the parentheses:

    {               // newline is ignored here
      Foo(x + y,    // newline is ignored here
          {         // newline is ignored here
            a = A() // newline is a terminator
            a + b   // newline is a terminator
          })        // newline is a terminator
    }               // newline is a terminator


You can always add parentheses to any expression, so this rule would suffice, although one could argue that we need a more elaborate rule to cover cases like

    x = Foo() +
        Bar() +
        Baz()

Another option is a line continuator, let's say `\`, written as

    x = Foo()
    \ + Bar()
    \ + Bar()

or in the more traditional way,

    x = Foo() \
      + Bar() \
      + Bar()

### "How will we add new literal types in the future?" ###

Since literals in Loyc trees can contain _anything_, you can add new literal types without changing the LES parser by adding a postprocessing stage. For instance, you could support byte literals like `bytes("61 62 63 00")` by (1) adding a postprocessor that finds the `bytes` operator and replaces it with a literal containing a byte array, and (2) adding a preprocessor before converting a node to text that replaces all byte literals with calls to `bytes`.

But this doesn't entirely solve the problem, because round-tripping is imperfect. For instance, if you construct a one-argument call `bytes("AB")`, serialize it and deserialize it again (with your postprocessor attached), you'll get a byte array back rather than the original call.

So, we should have a plan for how new literal types can be added, and where possible, old versions of the parser should be able to handle new literals. Bonus points if new literal types can be round-tripped by old code. Here's my idea about that:

- Numbers can be followed immediately by an identifier indicating the type of number, e.g. `0x1234u128` or `100_000_unum`. If the LES parser doesn't support the number suffix used, a warning (error?) will be printed and the number will be interpreted in the default way (as an integer or double, as applicable). The output will include a `#trivia_literalType()` attribute with the suffix attached as a symbol (e.g. `123.0u128` => `@[#trivia_literalType(@@unum)] 123.0`).

- Strings can be preceded by an identifier indicating the type of string, e.g. `bytes'''BA AD'''` or `re"[a-zA-Z]"`. If the LES parser doesn't support the prefix used, a warning (error?) will be printed and the string will be parsed in the default way. The output will include a `#trivia_literalType()` attribute with the prefix attached as a symbol (e.g. `bytes'''BA AD'''` => `@[#trivia_literalType(@@bytes)] '''BA AD'''`)

- If `@` is applied to an identifier that didn't need it (e.g. let's say `@snan`, as there is currently no signalling NaN in the spec), this will be recorded by setting one of the style bits (although I wasn't planning to _add_ style bits to the LES specification just yet.).

### "Can we add a general mechanism for suffixes?" ###

LES hasn't really given a meaning to the backslash `\` yet, so we could dedicate this for marking suffixes. But what should the precedence of such an operator be? In any case, the backslash itself should probably be included in the name of the operator stored in the Loyc tree, so that ``%x`` (equivalent to `@%(x)`) would not be the same thing as `x\%` (equivalent to ``@`\%`(x)``).

I've always wanted a programming language that supported unit types like "metres", "px", "dp" and "MB/sec", and the most natural way to express this is with a suffix. To that end, the suffix marker `\` could be followed not just by punctuation, but by any fancy identifier (that is, letters, numbers and punctuation). The LES parser wouldn't care whether a suffix like `12.3\metres` is to be treated as an "operator" or a "unit".

Precedence issues with WebAssembly
----------------------------------

LES doesn't permit custom syntax, but you can exploit its built-in syntax creatively. That's what I did when I proposed the following ways to express certain operators:

    function foo($x : i32) : i32 {...}
    br exit => result_value                // unconditional branch
    br exit => result_value ? condition    // conditional branch
    br_table default | [a, b, c] : $index  // branch table (switch) 
    f32.store [$addr,0] = 0x0p0            // store into memory

In the current version of LES, the first four are superexpressions, so if they appear within a larger, outer expression, they must appear within parentheses. However, these parentheses would almost never be needed since `br` and `br_table` do not return a value to the outer expression, and for a `function` there cannot be an outer expression.

The first two would parse as intended in all contexts, as long as we eliminate the need for a semicolon after the function's closing `}`, as described earlier. The second one has the structure `br(exit => result_value)`. The left-hand side of `=>` has high precedence, which could be a problem if the left-hand side were an arbitrary expression, but left-hand side is merely a label so nothing can go wrong. The _right-hand side_ of `=>` has the _lowest_ precedence, which is exactly what we want; if you write `br exit => $1 = $1 + $2` it has the structure `br(exit => ($1 = ($1 + $2)))`: the entire result expression remains a child of `=>`, as it should be.

The conditional branch has the structure `br(exit => (result_value ? condition))`, so as long as the `condition` and `result_value` don't disrupt that structure, all is well.  A typical expression like `br exit => $z & 255 ? $x == $y` preserves that outer structure, since `?` is a low-precedence operator. However, if you use an assignment like one of these:

    br exit => $x * $y ? $x = $y // first case
    br exit => $x = $y ? $x > $y // second case

An assignment, the only thing with lower precedence than `?`, disrupts the structure as follows:

    br(exit => (($x * $y) ? $x) = $y)  // first case
    br(exit => ($x = ($y ? ($x > $y))) // second case

I've been thinking that the left-hand side of `=` should have a higher precedence than the right-hand side; by increasing it, the problem in the first case would disappear, but the second case is still borked. Thus if this syntax were adopted, extra parentheses would be required in certain cases, a fact that could confuse people writing Wasm. As a result, it's probably best to drop the punctuation in favor of either the basic

    br_if(exit, result_value, condition);

or possibly this:

    br (exit => result_value) if (condition);

`br_table` has the same problem, but this time it can be solved consistently if the precedence of the left-hand side of assignments is raised. 

Finally, `f32.store[$addr,0] = -0x0p0` will sometimes need parentheses around it unless the precedence of the left-hand side of `=` is raised quite high, since for example `$x * f32.store[$addr,0] = -0x0p0` is currently parsed as `($x * f32.store[$addr,0]) = 0x0p0`. So I think the precedence should be raised quite high (probably to just above `*`). The only reason _not_ to raise it would be slavish devotion to the precedence rules of existing languages. In practice, existing languages give a semantic error if you write something like `x * y = 0`, so the potential for changing the meaning of existing code when you paste it into LES is low.

### Hold your horses! ###

We're not quite done yet: we have to consider the effect of the proposed changes in the [separate document](/les/juxtaposition-discussion.html). What effect would that have? 

- the proposed syntax `br (exit => result_value) if (condition)` is no longer legal. But if we opt to include binary operators that start with `'`, and we decide that they have very low precedence, then we could use

        'br exit => result_value 'if condition
    
    (syntax tree: `@'br(exit => @'if(result_value, condition))`)

- For the other stuff we need some single quotes:

        'function foo($x : i32) : i32 {...}
        'br exit => result_value
        'br exit => result_value 'if condition
        'br_table default | [a, b, c] : $index
    
- As before we need to raise the precedence of the LHS of `=` to write this without parentheses:

        $x * f32.store [$addr,0] = $y

Okay. Thanks for reading! Comments? Visit [#697](https://github.com/WebAssembly/design/issues/697).
