---
title: "LESv3 design nearing completion"
layout: post
toc: true
XXXXXcommentIssueId: 41
tagline: LES is more.
---

LESv3 was set back somewhat by the news that WebAssembly's developers weren't interested in going beyond s-expressions for the MVP. I wanted to suggest using a subset of LESv3 as the text format, something that could be implemented quickly; for instance instead of using syntax like this:

    (func $doubleExp (param $0 f64) (result f64)
        (get_local $0)
        (call $exp)
        (f64.const 2)
        f64.mul
    )

They could have used a more compact representation like this - something that would be straightforward to parse and compatible with LES, but lacking syntactic sugar that could be added after MVP (minimum viable product):

    .memory 1
    .func doubleExp(`0`: f64): f64 {
        `0`
        .call $exp
        2.0f64
        f64.mul
    }

LESv3 would have some nice properties even for the MVP, like that fact that it defines clearly how to write identifiers that contain arbitrary UTF-8 characters, arbitrary bytes or both. Understandably, though, they had a lot of other things to worry about and didn't want to get into the fine bikeshedding details. Hopefully they will pick up the text format issue again after MVP, but it wouldn't surprise me if they end up using s-expressions forever.

In the meantime I have been reviewing LESv3 - on and off - and made several decisions. The core of LES remains unchanged: it consists of C-like expressions with function calls, infix operators, prefix operators, suffix (`++ --`) operators, and indexing (e.g. `i32[x] = y`). It will also have JavaScript-like `[lists]`, `{braced blocks}`, and tuples. There will be an infinite number of operators, and an operator's precedence is chosen automatically by a set of rules.

On top of that core, LESv3 will have:

- Attributes with vaguely Java-like syntax (e.g. `@thisIsAnAttribute`)
- Newlines as expression terminators, both at the top level and within braced blocks (inside parentheses or square brackets, commas or semicolons are used instead)
- A special colon suffix (`:`) for labels, which (unlike normal suffix operators) is also a binary operator.
- Keywords for the literals `true`, `false` and `null`
- A mechanism to treat any sequence of bytes (including non-characters) as an identifier, using `` `backquoted strings` `` with C-like escape sequences. This was [described in the previous post](http://loyc.net/2016/lesv3-and-wasm.html#fancy-identifiers).
- A model for "custom literals". This was [described in the previous post](http://loyc.net/2016/lesv3-and-wasm.html#custom-literals), although the syntax of `@@literals` might be changed.
- A system of "keyword expressions" (see below).
- A plan to fix the precedence mistakes of C. For example `&` and `==` will be immiscible, so that `x & 3 == 0` is a syntax error.
- Custom "word" operators made of letters and underscores

After writing the last post, I made the following decisions:

- Newlines are terminators. A semicolon can also be used as a terminator. Indentation is not significant, however.
- Colon (`:`) can be used as a suffix operator (e.g. `label:`).
- I eliminated the juxtaposition operator (`foo bar` will not be a valid expression)
- I eliminated block call expressions (`foo { bar }` and `foo (bar) { baz }` will not be valid expressions)
- I eliminated single-quote operators as in `stop() 'if x < 0`
- I added "word operators", which are simple identifiers that act as binary operators (e.g. `stop() if x < 0`). I also added "combo operators"; see below.
- To avoid confusion, a space is required after an attribute: `@foo()` will be a syntax error
- Keyword-expressions will be allowed to start mid-expression (e.g. `msg = .switch x { 1: "one"; 2: "two"; _: "many" } + " apples"`)
- To simplify syntax highlighting, keywords like `.foo` will be parsed as a single token (previously, the parser would merge a `.` token with an identifier token.) This means printers will have to be careful: if they are forced to add a space before a dot operator, they must also add a space after it.
- I picked some specific details of how keyword expressions are parsed and how their output trees are structured.
- In LESv2 you could write `Foo(x,,)` which calls `Foo` with three arguments (`x` followed by two "empty" arguments). This will be a syntax error in LESv3, but empty expressions terminated by a _semicolon_, as in `for (;;)`, will still be allowed.
- Argument lists can use comma or semicolon as a separator.

Unlike LESv2, LESv3 will **not** be able to parse JSON directly because newlines are significant in LESv3.

Colon operator
--------------

A colon can appear at the end of any expression to, in effect, create a label. Note that the colon character is first and foremost a _normal_ operator character so, for example, in `x++:`, `++:` is treated as a unit, which in this case will act as a binary operator. If there is nothing after it, `x++:` is a syntax error.

Colon is like a suffix operator with a precedence lower than everything else, but it behaves more like a terminator, as only a single colon is allowed (e.g. `x: :` is not a valid expression).

Normally, a newline is allowed after a binary operator as in

    x = Foo(y) +
        Foo(z)

An exception is made for the binary colon operator so that 

    x = Foo(y) :
        Foo(z)

counts as two separate statements.

Keyword expressions
-------------------

LES has no keywords for creating functions, classes or other things. Instead of keywords, LESv2 used two different kinds of left parenthesis (one with a space in front, the other without), which help distinguish "superexpressions" (such as a function declaration) from normal expressions. This was a pretty cool idea since it allowed code that resembled Javascript or Rust:

    // LESv2 code
    if x > 0 { Foo(); } else { Bar(); };

Unfortunately, while using LESv2 I noticed a couple of usability problems. The first and biggest problem was the need to write a semicolon after the closing brace. As a longtime C/C++/C# programmer, I forgot that semicolon on a regular basis (the effect of a forgotton semicolon was usually a syntax error, though occasionally it caused the statement to "merge" with the following statement, usually causing a semantic-level error like "`if` doesn't expect so many arguments!".) This problem disappears automatically when we treat newline as a terminator.

A second problem was that you couldn't write `if !flag` or `while !done`:

    if !flag { ... } // syntax error at '{': expected ';'

Since `!` is also the generics operator, `if !flag` means `@if<flag>` in C# notation and the braces are unexpected. So you'd have to write `if (!flag)` instead.

A minor third problem was that if-else chains required parentheses after the first `else if`, even though parentheses are not required after the first `if`:

    if condition {
      Foo();
    } else if (x > 0) { // parentheses required
      Bar();
    };

This requirement for parentheses helped increase the chance of a syntax error when the user forgot a semicolon after a closing brace.

In LESv3, "keywords" are explicitly marked with a dot. For example, `.when` is a keyword in this example:

    .when condition {
      Foo()
    } else .when x > 0 {
      Bar()
    }

Of course, as with all LESv3 statements, semicolons are no longer required to mark the end. Parentheses after `.when` are never needed, either.

Keyword expressions have up to four parts, in this order:

1. The dot-keyword (`.when`)
2. An expression (`condition`)
3. A braced block (`{ Foo() }`)
4. A sequence of "continuators" (`else ...`), which is a special word like `else` followed by an expression, a braced block or both.

Parts 2, 3, and 4 are optional, so the core grammar is

    KeywordExpression :
      keyword
      // 'greedy' reflects the fact that the grammar is otherwise ambiguous
      greedy( Expr[StartStmt] )?
      greedy( "\n"? BracedBlock )?
      greedy( Continuator )*;

Examples:

- Keyword alone: `.break`
- Keyword with expression: `.return null`
- Keyword with expr. and braced block: `.while i < length { list[i]++ }`
- Keyword with expr. and continuator: `.try { Foo() } catch { Fail() }`
- Keyword with expr. and multiple continuators: `.try Foo() catch Fail() finally Cleanup()`

Newlines are allowed before braces and continuators:

    .when condition
    {
      Foo()
    }
    else .when x > 0
    {
      Bar()
    }

This is a popular style of code so I thought it was important to allow it, but since braced blocks are valid expressions by themselves, and newlines are normally a statement terminator in LESv3, there is an ambiguity here: is the braced block `{ Foo() }` meant to be associated with `.when foo` or is it an independent block of code? LESv3 assumes that any braced block after a keyword-expression is meant to be part of the keyword-expression, provided there is no more than one newline before the opening brace. For example, if you write

    .if foo
    // this line acts like a terminator
    {
      Foo()
    }

It will be parsed as **two** statements; instead you should write

    .if foo
    { // this line does not act like a terminator
      Foo()
    }

Now, if you write

    .if hungry
    {
      eat
    }
    watch_tv
    {
      etc
    }

The line `watch_tv` is considered independent from the `.when` expression, so this is in fact three separate "statements": first `.if hungry { eat }`, second `watch_tv`, third `{ etc }`. In order to add additional clauses like an `else` clause, LESv3 introduces a concept of "continuator keywords": special words like `else` that allow a keyword expresson to continue. 

Should continuator keywords be bona fide keywords, or (in accordance with LES tradition) contextual keywords: identifiers that become keyword if they appear in certain locations? I am undecided.

The set of continuators will be fixed when the definition of LESv3 is finalized; the current set is the following words:

    else elsif catch except finally
    while until plus minus so
    and or but 

A continuator consists of a continuator keyword, followed by an expression and/or braced block:

    Continuator :
      "\n"? ContinuatorKeyword
      "\n"?
      ( BracedBlock
      / TopExpr "\n"? // top-level expression
        BracedBlock?
      )

### Dealing with the ambiguity ###

As mentioned, if a keyword expression like `.keyword expression` is followed by a newline and a braced block, it's unclear whether the braced block was meant to be associated with the keyword.

Of course, an LES printer is forced to deal with this issue. It cannot print a keyword expression on one line (in the usual way) and a braced block on the next line, lest the two expressions be merged upon reparsing. The conservative solution to this problem is to add a semicolon to every expression within a braced block.

A more careful solution is to add a semicolon on expressions that contain (or end with) a keyword expression (e.g. `x = .foo x { };`) if they are followed by an expression that begins with a braced block (e.g. `{x} = {5}` or `{}()`). It's best for a printer not to try to be more clever than this, because you can't necessarily avoid a semicolon when the keyword expression ends with a closing brace. That's because a braced block is an expression in its own right and can count as one of the "expression" parts of a keyword expression instead of as the "braced block" part. For example, `.if {x} {y} else {z}` ends with a closing brace, but could be followed by another braced block because `{z}` counts as the "expression" part of the else clause, not as the "braced block part".

We don't want end-users to think about this, of course. You can design an LES-based language where this is not an issue; for example, if a language is designed so that a braced block cannot appear at the beginning of an expression, the issue does not arise. And let's consider what happens if we built a C-like language on top of LES. Only six statements in C begin with a specific keyword and _do not_ have an associated braced block:

    break
    continue
    goto
    return
    case
    typedef

All of these except `case` and `typedef` alter control flow, so if there is a braced block after them, the braced block is normally unreachable and didn't really need to exist. Meanwhile, `case` could be expressed in LES without using `.case`, and `.typedef` would appear in a non-executable context where a braced block is meaningless. So in practice, the ambiguity is not a problem here.

Typically, languages will often have some reason why a braced block at the beginning of a statement (and after a keyword-expression) could be valid code, but no good reason for users to actually write that code. As another example, if you use braced blocks to define dictionaries then

    {"print": x => stdout.print(x)}["print"](123)

might be a valid statement that prints `123`, yet there is no real-world reason to write a statement of that form.

### Adding keywords to LES ###

LES itself won't have a mechanism to define traditional keywords (with no "." in front of them), but it should be straightforward to add them programmatically. For instance, the .NET `Les3LanguageService` lets you perform lexing and parsing as two separate steps, which lets you insert a lexer filter in between the lexer and the parser. Within this filter you could find identifiers that you want to treat as keywords, and replace those identifiers with keyword tokens (changing `class` to `.class`, while leaving `` `class` `` alone.) Perhaps in the future I'll add a filter class to the Loyc.Syntax library so you can do this more easily.

However, I'm considering whether to pick a set of predefined keywords so that typical languages built on LES won't want to resort to such tricks.

Word Operators
--------------

An identifier that starts with a lowercase letter can be used as a right-associative binary operator with very low precedence. This allows binary operators that look as if they were "built into" LES:

    Scream("Negative inputs are not allowed!") if input < 0
    .class Foo inherits FooBase implements IBar { /*...*/ }

In the syntax tree produced by LESv3, a single quote (`'`) will appear before the operator name. For example, `paper beats rock` has the syntax tree `` `'beats`(paper, rock)``. Thus, operators are easily distinguished from normal function calls.

There is nothing ambiguous about allowing ordinary words as operators, although it is potentially confusing to users if it is not obvious which words are operators and which are normal identifiers, so an LES-based language should use such words judiciously (e.g. by only allowing words from a predefined list to be used as operators).

If we're not careful, there is a potential for people to write code that is "obviously" wrong but still parses. For this reason there are a couple of special rules about how word-operators are used that do not apply to normal binary operators:

1. Unlike normal operators, a word operator cannot be followed by a newline, except inside parentheses or square brackets.
2. Word operators must be followed by a space or tab character.

The first rule ensures that an error in one expression doesn't cause it to inadvertently "merge" with the line afterward. The second rule catches some mistakes like these:

    Leave() if if MovieIsBoring() // repeated operator
    food = fridge.Foo Food(x, y)  // two identifiers where one was expected
    x = x y*z                     // uhh... what?

I am also considering a third rule:

3. Word operators cannot use uppercase letters, or mixed case.

Mixed-case identifiers are very common in some languages, notably C# and VB where virtually all public APIs have both uppercase and lowercase letters. Disallowing them as operators would form a third line of defense against the parser accepting code that contains a mistake.

I was thinking of defining two kinds of word operator: `UPPERCASE` and `lowercase`. Lowercase operators would have very low precedence, while uppercase operators would have somewhat high precedence. However, for the sake of simplicity perhaps there should be only one precedence in LESv3 and disallow uppercase letters. If LES becomes successful, uppercase operators could be added in LESv4.

The precedence of lowercase word operators produces the following equivalences:

    // Right associative
    a + b op_one c + d op_two e + f === (a + b) op_one ((c + d) op_two (e + f)) 
    // Ultra-low precedence
    a = b && c operator x = y || z  === (a = (b && c)) operator (x = (y || z))
    // The only operator with lower precedence is `=>`, and only on its right
    fn = x => x operator 123        === fn = (x => (x operator 123))
    // A word operator can be embedded within a keyword expression
    .keyword x operator 123 { }     === .keyword (x operator 123) { }
    // When a keyword expression ends, it can be followed by a word operator
    .keyword { } operator 123       === (.keyword { }) operator 123

Their extremely low precedence makes word operators a natural companion of keyword expressions. Keyword expressions like `.eat x { }` cannot accept multiple arguments before the braces, e.g. `.foo x, y { }` is invalid. Word operators solve this problem by taking the place of the comma, as long as you can think of a name that fits the situation, e.g. `.watch "Iron man" at 8pm`. Whatever code handles the `.watch` expression would then check that `'at` was called with two arguments.

An identifier cannot be used as an operator, however, if it is one of the "continuators" (`else`, `catch`, `finally`, etc.). This rule eliminates the ambiguity in `.foo x {} else y`: clearly `else` is a continuator and not a word operator.

Combo Operators
---------------

Dan Gohman proposed sign-aware infix operators for WebAssembly, as in

    $x = $y >s $z

_Background:_ WebAssembly data types are unaware of signedness. For example, `i32` (32-bit integer) is neither signed nor unsigned, and so the operator itself must indicate whether a signed or unsigned operation should be performed: hence `>s` for "signed greater than". This _version_ of signed operators can't work in LES because it is ambiguous: in general, there's no way to tell that `s` is part of the operator and not a separate identifier.

After realizing that word operators were a workable concept, I was delighted to discover we could combine a word with a normal operator like this:

    $x = $y s> $z

This is _not_ ambiguous. `s` appears right after an expression so it cannot be interpreted as a normal identifier. It cannot be word operator either, because word operators must be followed by a space or tab and `s` is not. Instead, the identifier `s` and the punctuation `>` are merged by the parser into a single unit called a _combo operator_. The punctuation portion of the operator determines its precedence; thus `s>` has the same precedence as `>`.

The word and punctuation have unlimited length, and the operator must be followed by a space, tab, or newline character to clearly mark where it ends. In the final syntax tree, the operator gets a single quote in front of it, so `$y s> $z` means `` `'s>`($y, $y) ``.

The disambiguation marker
-------------------------

An LESv3 parser should create a "trivia attribute" to record the fact that an expression is in parentheses. For example, `(x*y)+z` has parentheses trivia while `x*y+z` does not. However, the trivia should not be recorded if the parentheses are just used to associate attributes with a subexpression, as in `@addition (@multiplication x * y) + z`.

Occasionally a printer is asked to print a problematic expression like `` `'*`(x + y, z)`` and would like to use operator notation to do so, even though the tree doesn't include parentheses trivia. For this purpose, LESv3 defines the _disambiguation marker_ `@@`. `@@` acts like an "empty attribute": it is allowed where attributes are allowed, but it does not create an attribute. This allows a printer to more easily represent the presence or absence of parentheses in the Loyc tree. So `(@@ x*y)+z` and `x*y+z` will have identical Loyc trees, and more importantly `(@@ x + y) * z` has the same tree as `` `'*`(x + y, z)``. 

Printers should add a space after `@@`, since `@@x` counts as a custom literal, not as the identifier `x`.

New idea: Token Lists (pseudo-s-expressions)
---------------------------------------------

My implementation of LESv2 had "token tree" literals, which allow custom syntax to be embedded in LES code. I used them to support my [parser generator](http://ecsharp.net/lllpg). For instance you can write a grammar like this in LESv2:

    // Invoke parser with token literal @{...}
    LLLPG lexer @{
      EmailAddress :
        UsernameChars ('.' UsernameChars)*
          '@' DomainCharSeq ('.' DomainCharSeq)*;
      UsernameChars :
        ('a'..'z'|'A'..'Z'|'0'..'9'|'!'|'#'|'$'|'%'|'&'|'\''|
         '*'|'+'|'/'|'='|'?'|'^'|'_'|'`'|'{'|'|'|'}'|'~'|'-')+;
      DomainCharSeq :
        ('a'..'z'|'A'..'Z'|'0'..'9')
        ( '-'? ('a'..'z'|'A'..'Z'|'0'..'9') )*;
    };

What mechanism should LESv3 use for custom syntax? We already have triple-quoted custom literals, so we already have enough for an MVP (minimum viable product) since this grammar could be written as

    .LLLPG lexer'''
      EmailAddress :
        UsernameChars ('.' UsernameChars)*
          '@' DomainCharSeq ('.' DomainCharSeq)*;
      UsernameChars :
        ('a'..'z'|'A'..'Z'|'0'..'9'|'!'|'#'|'$'|'%'|'&'|'\''|
         '*'|'+'|'/'|'='|'?'|'^'|'_'|'`'|'{'|'|'|'}'|'~'|'-')+;
      DomainCharSeq :
        ('a'..'z'|'A'..'Z'|'0'..'9')
        ( '-'? ('a'..'z'|'A'..'Z'|'0'..'9') )*;
    ''';

But this is less convenient; there will be no syntax highlighting, for example.

The main disadvantage of token literals is that they are not Loyc trees; they are a separate data structure that would have to be defined in detail if LES were a widely-used standard. It would be simpler and easier, I think, to store custom syntax as a Loyc tree. Therefore, I propose a mapping from token lists to Loyc trees.

I propose the unary single quote operator (`'`) to introduce a token list. The single quote by itself would creates a call to `` `'`, with each token afterward treated as an arguments. For example, this code:

    x = ' <div style="...">123</div>;

would be equivalent to

    x = `'`(`'<`, div, style, `'=`, "...", `'>`, 123, `'</`, div, `'>`)

To understand this, remember that LES lets you write special identifiers in backquotes and operators in LES are already stored with an initial single quote, e.g. `2 + 2` really means `` `'+`(2, 2)``.

**Note**: Although I used HTML code as an example, you should _not_ try to represent HTML this way since the syntax is too different. LES, for example, uses C-style escapes like `"\""` while HTML does not. Instead, use a custom string like `html'''<p></p>'''` and parse it with a dedicated HTML parser.

The single quote can be followed immediately by a sequence of characters in the same character set as `@@`-literals. This becomes part of the name of the operator, e.g.

    x = 'make my breakfast
    // equivalent to
    x = `'make(my, breakfast)
    // equivalent to
    x = my make breakfast

If the token list contains parentheses, it creates a sublist with, perhaps, a target of `'()`. Example:

    x = '(* (a b c) (Foo))
    // equivalent to
    x = `'`(`'*`, `'()`(a, b, c), `'()`(Foo))

These token lists work significantly differently from s-expressions, mainly because spaces are not generally required between arguments: `Foo*` is two arguments, not one. Still, you could write very LISP-like code this way.

One cannot directly write a prefix expression like `+ x (* y z)` that means `x + (y * z)` in LES, but this can be made possible with a post-processing step such as a [LeMP](http://ecsharp.net/lemp) macro.

By the way, it's possible to examine the output from LES to find out if two tokens were adjacent - a fact which postprocessing steps might exploit for their own reasons. For example, in `' Foo*`, the `*` `Range.EndIndex` of the identifier `Foo` equals the `Range.StartIndex` of the identifier `'*`.

If the token list contains square brackets or braces, the contents of those brackets or braces are parsed as normal expressions. This provides a predictable way for "normal" code to be embedded within custom syntax. For example, my parser generator could use square brackets to represent arguments to rules, and braces to represent grammar actions and semantic predicates:

    .LLLPG parser('
      // Scans up to 10 digits of an integer
      UpTo10Digits ::= Digits[10];
      Digits[maxLen: int] ::=
          {len := 0}
          (&{len < maxLen} '0'..'9' {len++})+;
    )

The stuff like `maxLen: int` and `len++` would be parsed as normal expressions, while everything else would be parsed as token lists. In turn, those token lists would be interpreted by a `.LLLPG` macro that decides what they mean.

Additional details:

- As usual, comments are ignored within a token list
- The token `@` becomes an identifier called `'@`.
- Keywords like `.foo` become identifiers like ` `.foo` `.
- Empty parentheses `()` are stored as `` `'()`()``.
- Opening and closing brackets and braces must match. Combinations such as `'(]`, `'[}`, and `'{)` will produce a syntax error.
- Commas and semicolons terminate a token list just as a normal expression; for example `f(' x, y)` and `f(' x; y)` will produce the same syntax tree as `f((@@ ' x), y)`. However, inside parentheses, commas and semicolons can be part of the token list and they are stored as the identifiers `',` and `';` respectively. For example, `'(x , ; y)` will produce the following syntax tree: `` `'`(x, `',`, `';`, y)``.
- The `'` operator "takes over" parsing the rest of an expression, until reaching a semicolon, EOF, a significant newline, or an unmatched ')', ']' or '}'. If you want to be able to end a token list prior to the end of the statement, enclose it in parentheses.
- What would the `'` operator mean when used within a token list? I propose that we give it no meaning. It would be a syntax error.
- Printers need not support token lists, since any token list can be represented in prefix notation (for that matter, printers don't have to support operators or keyword expressions; it's just a matter of how pretty you want your output to be).

Closing thoughts
----------------

### Undecided issues ###

- Should token lists be adopted in LESv3? (probably: they are not hard to implement, and they seem useful inside the parser for recovering from a syntax error. Expected a comma? Just skip over whatever tokens you got instead by pretending it's a token list.)
- Should a token list like `'(+ x y)` be stored as `` `'`(`'()`(`+`, x, y))`` as suggested above, or should the first token within parentheses become the `Target`, so that this would actually mean `` `'`(x + y)`` (i.e. `` `'`(`'+`(x, y))``)?
- Should attributes be allowed in the middle of an expression, as in `foo: @immutable string`? How would parsing work for that?
- Should comma be allowed as a separator within braces, as in `{ a, b, c }`? Allowing commas allows JSON-style dictionaries, but disallowing commas may be better for detecting and reporting errors in the code. One thing I do _not_ recommend is to define a "comma operator" like C has. A comma operator would make the grammar more complicated.
- Should comma be allowed as a separator within tuples as in `(x, y)`? Probably yes. What about semicolon? In LESv2 the decision was made to use semicolon in tuples, in part because it is a _terminator_, whereas comma is a _separator_. Since `f(x,)` was a call that took two arguments, it wasn't logical that `(x,)` should be a tuple of one item, but semicolon works differently so `(x;)` was logically a tuple of one item. Should this syntax be allowed in LESv2?
- What do we do with the `#` character? Can it be a normal identifier character as in LESv2? (Note: the backslash `\` is already reserved for future use, or for use by supersets of LES.)
- Should continuator keywords be bona fide keywords, or (in accordance with LES tradition) contextual keywords: identifiers that become keyword if they appear in certain locations?
- If the answer is yes, should LES also add a fixed set of keywords from C-like languages that don't require a dot in front of them, e.g. `break`, `return`, `if`, `while`, `do`, `for`, `switch`, `var`? I would propose that these keywords simply be treated as if they had a dot in front (so `break` means `.break`). This could enable a lot of Javascript code to be directly parsed by LES, although most other C-like languages have frequently-used elements LES can't handle. **Limitation:** if we do this, `while` could no longer be a continuator keyword. `do-while` loops would have to become `do {...} until (...)` or something like that.

### What keywords? ###

Here's what a predefined keyword set might look like. We'd start with Javascript keywords that are not type names (`int`, `long`), attributes (`public`, `abstract`) or operators (`typeof`, `instanceof`):

    // Simple statements (7)
    goto  return  break  continue  
    throw  case  var
    
    // Block statements (8)
    function  class
    if  while  do  for  switch  try 
    
    // Expressions (2)
    new  delete
    
    // "Future" reserved words (5)
    interface  enum  import  export  package

Why JavaScript? Because JavaScript is the only language similar enough to LES that partial compatibility is possible. Other languages like C, C#, C++ and Java have too many incompatible elements, e.g. typed variable declarations (`Foo x`), angle-bracket generics (`Foo<T>`), and paren-typecases (`(Foo) x`).

JavaScript also has some elements that LES wouldn't handle; the biggest problem is block statements without braces, like `if (c) f()`. Also unsupported: `typeof e` and `void 0` (you'd have to change these to `typeof(e)` and `void(0)`). The JavaScript expression `x instanceof y` could still work, although its precedence is wrong. Similarly, JavaScript's `yield*` could work, but the precedence of `*` is high so in some cases the expression after `yield*` would need parentheses around it.

Note that LES would treat `new` significantly differently from Javascript (because LES keyword expressions act like they have very low precedence, unlike JS `new`), but typical JavaScript code like `var x = new Foo(y)` would parse successfully.

We would then add some additional keywords, such as:

    // Type & namespace definitions (4)
    struct  type  alias  namespace
    
    // Other (8)
    fn  prop  process  construct  
    data  loop  yield  expect

These cover some common concepts from various programming languages:

- `struct`, `alias`, `type`: structures, alias types (think `typedef`, except that `typedef` is a syntactically strange part of C that LES cannot and should not emulate), and other types
- `namespace`: similar to `package`, but without the connotation of creating a monolithic unit of code
- `fn`: Javascript's `function` keyword is so long as to be unweildy. I propose `fn` from Rust as a shortcut.
- `prop`: Could be read as "property" (as seen in C#, VB, D, etc.), "propose" or "proposition" (for logic programming)
- `process`: Could be useful for defining things we might call tasks, jobs, processes, or flowcharts.
- `construct`: For defining constructors. Could also be read as a creation command.
- `data`: For grouping data together. In Haskell, `data` defines a data type.
- `loop`: For infinite loops (`loop {...}`) or some other kind of loop
- `yield`: To send data from a generator or coroutine, or to wait, without terminating
- `expect`: Could be used to define inline assertions or unit tests

Including literals and continuator keywords, LES would have around 48 keywords: more than C (32), but fewer than C++ (90), C# (77), Javascript (59) or Java (52).

### How complex is LESv3? ###

Compared to JSON or s-expressions, LES is quite complicated. Also, LESv3 is a bit more complex than LESv2.

However, even if we add lots of new keywords, parsing LES is still be easier than parsing all popular languages in the C family. For example, compared with C, LES avoids the complexity of feedback from the symbol table to the parser/lexer, the complexity of the preprocessor, and the subtleties of C type declarations like `char *(*(* const *foo[][8])())[]` and "abstract declarators" like `int (*(*)())()`. In terms of code size, a C parser will be moderately larger before adding the mandatory preprocessor. For reference, the main LESv3 parser grammar (excluding the lexer and some helper code) is currently 333 lines, which the parser generator expands to 810 lines of C#. The C parser pycparser, written in Python, is 1740 lines (excluding the lexer, preprocessor, and other helper code, though this does include an unusually large amount of documentation).

LESv3 is at least 4 times simpler than C# or [Enhanced C#](http://ecsharp.net):

- The official Microsoft C# parser (Roslyn) is over 10,000 lines of code, though it is written in a vertically wasteful style.
- The Enhanced C# parser grammar, which I also wrote, is 2126 lines of EC# which expands to nearly 5000 lines of C#.
- An LES lexer is not much different from lexers of C# or EC# or other languages such as Java, except that the grammar that recognizes operators and keywords is notably simpler.

**Fun fact**: my LES implementation does something you won't get from most other parsers: it preserves comments in the Loyc tree as so-called "trivia attributes". This feature is not implemented by the parser, though. It is a language-independent algorithm used by LESv2, LESv3 and Enhanced C#.
