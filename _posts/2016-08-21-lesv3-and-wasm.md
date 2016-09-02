---
title: "LESv3 and Wasm: the current plan"
layout: post
toc: true
commentIssueId: 41
---

Now that Wasm is moving toward a stack machine, I think that LES is now more relevant than ever, because there seems to be a need for not _one_ Wasm language but two: one for Wasm itself, and another for the pseudo-wasm AST used by producers (i.e. binaryen). Both languages would support expressions, but each would interpret them differently. In the "official" Wasm, `$a() + $b()` could be syntactic sugar for the sequence `$a(); $b(); i32.add` if both functions return `i32`, whereas in binaryen, the same text would represent the AST `i32.add($a(), $b())`.

While only one Wasm variation is expected so far, it's not hard to imagine others:

- People will need to prototype new Wasm features.
- I'm interested in making some kind of higher-level language using wasm's AST variant as a semantic foundation.

A general-purpose syntax like LESv3 allows people to do all this without touching the parser or the printer (serializer), let alone writing new ones. 

This post summarizes my current plan for LESv3 syntax, especially as it relates to WebAssembly, and it highlights a few questions for the Wasm CG.

What is LES? A one-paragraph summary
------------------------------------

If LISP had been invented in the 90s as a native member of the C family, LES would be its parser. Like the s-expression, LES is parsed into a simple data structure called a [Loyc tree](http://loyc.net/loyc-trees/), but unlike s-expressions, the data structure is designed to hold code. Instead of using a "list" as the recursive element, Loyc trees use a "call": `f(x, y)` instead of `(f x y)`. `f` is called the "target" of the call and although most targets are identifiers, a target can be any node, including another call (e.g. `f(x)(y)`). Also, every node has an (optional) list of attributes attached (which can include "trivia" such as comments, and in Wasm might be used for debug info), and should also hold a source code range for use in error messages (e.g. `~/code/my_code.les:1012:12:1012:19`)

Like LESv2, LESv3 will support general C-like expressions with function calls, infix, prefix, and suffix (`++ --`) operators and indexing (possible syntax for stores: `i32[$x] = $y`). It will have JS-like `[lists]`, maybe `{dictionaries}`, and tuples. Operators are represented by calls to identifiers that start with a single quote, e.g. `2 + 3` is a call with a target called `'+`. In the current notation, `2 + 3` could also be written as `` `'+`(2, 3)``.)

Relationship to WebAssembly
---------------------------

I've been optimizing LESv3 to make it a good basis for the WebAssembly text format and I am publishing this in the hope of getting your feedback, opinions and preferences. **I'd much rather have your opinion now than after I've written separate parsers for multiple languages!**

Keep in mind that LES is also designed for general-purpose non-Wasm uses. That's the main reason for syntax differences between LES and the [proposal](https://github.com/WebAssembly/design/pull/704) by Dan Gohman (and Michael Bebenita?), now developed in [this repo](https://github.com/mbebenita/was). For example, in that proposal, plain words like `foo` are reserved for future use as keywords, whereas `$foo` is an "identifier". In LESv3 (and in this document!), plain words like `foo` are considered identifiers, while `$foo` is just an identifier with the prefix operator `$` in front.

Hopefully you'll agree that the changes are not onerous. Even when writing Wasm code by hand, LES is much easier to use than either the s-expression syntax or any traditional assembly language.

Wasm stack machine in LESv3, by example
---------------------------------------

Let's consider the C example 

~~~cpp
int sumIntegers(int* input, int length) {
  int sum = 0;
  for (int i = 0; i < length; ++i) {
    sum += input[i];
  }
  return sum;
}
~~~

The optimized s-expression I got from [Wasm Explorer](http://mbebenita.github.io/WasmExplorer/) is, after adjusting for readability,

~~~
(module
  (memory 1)
  (export "memory" memory)
  (export "_sumIntegers" $_sumIntegers)
  (func $_sumIntegers (param $input i32) (param $length i32) (result i32)
    (local $sum i32)
    (set_local $sum (i32.const 0))
    (block $stop
      (br_if $stop (i32.lt_s (get_local $length) (i32.const 1)))
      (set_local $sum (i32.const 0))
      (loop $unused $loop
        (set_local $sum 
          (i32.add (i32.load (get_local $input)) (get_local $sum)))
        (set_local $input (i32.add (get_local $0) (i32.const 4)))
        (br_if $loop (set_local $length 
          (i32.add (get_local $length) (i32.const -1))))
      )
    )
    (return (get_local $sum))
  )
)
~~~

Manually updating this to stack-machine form, I get

~~~
  (memory 1)
  ...
  (func $_sumIntegers (param $input i32) (param $length i32) (result i32)
    (local $sum i32)
    (i32.const 0) (set_local $sum)
    (block $stop 
      (get_local $length) (i32.const 1) i32.lt_s (br_if $stop)
      (i32.const 0) (set_local $sum)
      (loop $loop
        (get_local $input) i32.load (get_local $sum) i32.add (set_local $sum)
        (get_local $input) (i32.const 4) i32.add (set_local $input)
        (get_local $length) (i32.const -1) i32.add (tee_local $length) (br_if $loop)
      )
    )
    (return (get_local $sum))
  )
~~~

In LESv3, this stack-machine code could be expressed direcly as follows (omitting `get_local` and `i32.const`, which are not needed):

~~~
  .memory 1;
  ...
  .fn $_sumIntegers($input: i32, $length: i32): i32 {
    $sum: i32;
    0; set_local $sum;
    {
      $length; 1; i32'lt_s; br_if stop;
      0; set_local $sum;
      loop (loop) {
        $input; i32'load; $sum; i32'add; set_local $sum;
        $input; 4; i32'add; set_local $input;
        $length; -1; i32'add; tee_local $length; br_if loop;
      }
      stop:
    }
    $sum; // return value
  }
~~~

More typically it would use expression notation, like this:

~~~
  .memory 1;
  ...
  .fn $_sumIntegers($input: i32, $length: i32): i32 {
    $sum: i32;
    $sum = 0;
    {
      br stop 'if $input '<s 1;
      $sum = 0;
      loop (loop) {
        // I picked := for set_local and = for tee_local;
        // feel free to vote your own preference.
        $sum := i32[$input] + $sum;
        $input := $input + 4;
        br loop 'if $length = $length + -1;
      }
      stop:
    }
    $sum; // return value
  }
~~~

The Wasm assembler can allow the two notations to be freely mixed (similar to how the spec tests do today), e.g.

~~~
$length = $length + -1;
br_if loop;
~~~

LESv3 syntax elements
---------------------

Now that you know what it looks like, let's discuss the details.

### Semicolons ###

I need feedback about whether semicolons should be required to terminate statements. If semicolons **are** required then a LESv3 parser can read JSON files; if statements are terminated by newlines then JSON isn't really supported because 

    { "foo"
      : "bar" }

will be parsed like `{ "foo"; (:"bar"); }` which is quite different. Maybe a postprocessing step could restore the intended meaning, but instead you should probably just use a dedicated JSON parser in the first place.

For various technical reasons, it seems easier if newline is a terminator, so I am inclined to drop JSON support. If newline is a terminator, semicolon can still be used as a separator or terminator, which is useful when writing postorder notation or declaring multiple variables.

#### Optional semicolons

The LES parser needs a way to detect the end of each expression, of course. If we decide newline isn't a terminator, semicolons will inevitably be forgotten. While novice programmers often forget semicolons everywhere, experienced ones forget them specifically after closing braces.

To solve this problem, semicolons could be generally optional after a closing brace, provided that the next token after the closing brace _cannot_ continue the expression. This doesn't always work though, especially if we use dot-keywords as introduced below; the old `#hash-keyword` idea was more compatible with this plan. One thing that might help further is to assume if there is a newline after the closing brace, the user intended to end the statement, except if there is a "continuator" (a syntax element that is introduced below).

### Keywords ###

LESv2 has no keywords. Although Wasm doesn't need this, in LESv3 I decided to introduce exactly three keywords, `true`, `false` and `null`, because I found that writing a special notation like `@true` was cumbersome for end-users, while relying on a postprocessing step to translate `true` into a true _literal_ made it difficult or impossible to make an _identifier_ that happened to be named `true`. In LESv3, `true` is a literal and `` `true` `` is an identifier named `true`.

### Fancy identifiers ###

Normal identifiers have letters, digits, underscores (`_`) and/or apostrophes (`'`). The first character must be a letter or `_`.

In LES it has always been possible to use any arbitrary string as an identifier. In LESv3 strings will be interpreted as UTF-8, and invalid-UTF8 escapes like `\?ff` (or `\xff`?) for the byte 0xFF will be allowed so that arbitrary bytes can be allowed in identifiers.

Dan Gohman's prototype used backslashes to escape individual characters in an identifier. It looks like the current prototype doesn't support standard escapes like `$line1\nline2` for an identifier with a newline in the middle, whereas when I designed LES I assumed that identifiers would be escaped similarly to strings, except that you could write, for instance, `fun\ fact\!` to get an identifier with a space and exclamation mark in it.

In LESv3, my plan had a problem. If `true` is a keyword then we need a way to write an _identifier_ called `true`. Using `\true` to represent the _identifier_ called `true` is a no-go because `\t` conventionally represents a tab character, so this particular identifier wouldn't have its obvious meaning of "tab character followed by `rue`".

Instead, I decided that rather than escaping each individual character, identifiers can be enclosed in backquotes and parsed exactly as a string, e.g. `` `I'm a whole sentence!` `` is an identifier. Compared to the alternative, this rule gives longer identifiers in some cases and shorter ones in others. It's good for representing C++ mangled names like `` `?Fmyclass_v@@YAXVmyclass@@@Z` ``, and less efficient when there's single strange byte like `` `\x1B` ``.

### Keyword expressions ###

LES has no keywords for creating functions, classes or other things. In lieu of keywords, LESv2 used two different kinds of left parenthesis (one with a space in front, the other without), which help distinguish "superexpressions" (such as a function declaration) from normal expressions. 

~~~js
// LESv2
if x { f(); };   // Superexpression equivalent to `if(x, {f();});`
if (x) { f(); }; // Superexpression equivalent to `if((x), {f();});`
if(x, y);        // Call a function named `if`
if (x, y);       // Syntax error with suggestion to remove the space
fn Foo(x: i32);  // Superexpression equivalent to `fn(Foo(x: i32));`
fn Foo (x: i32); // Syntax error with suggestion to remove the space
~~~

Although no one from the Wasm group objected to this, no one supported it either. I decided to use a different design for v3 that avoids possible confusion. At first I planned to use `#` to denote "keywords":

~~~
#fn Foo(x: int32) {...}
~~~

But `#` is a relatively noisy character to look at, and you need the shift key to make one. My new plan is to use a dot, which I think is a little easier on the eyes and hands:

~~~
.fn Foo(x: int32) {...}
~~~

This makes some sense, as `.` is used for "directives" in some assembly languages. Following the dot, a "keyword statement", as this is called, has a single expression followed by an optional braced block with optional "continuator clauses". Here are some examples of keyword statements, a.k.a. "dot expressions":

~~~
// Dot expression with expression
.return $x + 1;
// Dot expression with expression and braced block
.struct Point { x: f64; y: f64; };
// Dot expression with expression, braced block and continuator
.if x > 0 { f(); } else { g(); };
~~~

A "continuator" is an identifier from a predefined set that includes `else`, `catch` and `finally`, that allows an extra clause to be added. The exact set of words allowed as continuators is not finalized, and the syntax of a continuator clause is not finalized either.

Potentially, a keyword statement could take comma-separated arguments:

~~~
.memory initial=1, maximum=10, exported=true;
~~~

However, LESv3 is a flexible language, so a keyword-expression is allowed anywhere that a normal expression is allowed, including as a function argument, which makes this ambiguous:

~~~
foo(bar, .baz a, b, c);
~~~

Does this function take four arguments, or two? The same issue arises if we want to keep JSON compatibility, since you could write 

~~~
{"key":"value", .baz a, b, c}
~~~

The dot in the keyword-expression becomes part of the name of the identifier that is called.

### Block calls ###

LESv3 borrows a feature from [Enhanced C#](http://ecsharp.net/) and takes it one step further. In Enhanced C#, you can write a "block call" which is a function call with a pair of braces afterward:

    unless(IsKitchenClean) { CleanKitchen(); }

This allows users to add new constructs that look built-in but are not. In both Enhanced C# and LES, the resulting syntax tree is a normal call with the braced block added as an extra parameter:

    unless(IsKitchenClean, { CleanKitchen(); }); // equivalent

(The braced block itself, if you were wondering, is a call whose target is `` `'{}` ``.)

A block call can appear in the middle of an expression:

~~~csharp
    str = "<" + switch(x) { A => "Eh?"; B => "Bee"; ... } + ">";
~~~

In Wasm, this feature could be used for loops:

~~~
   loop (label) { infinite(); br label; }
~~~

LESv3 adds a new feature: the braced block can be followed by a continuator clause. These clauses will have the same syntax as the continuator clauses on the keyword-expressions you saw earlier:

~~~csharp
    x = if (c) { a; } else { b; };
~~~

When building a language on top of LES you can choose between two styles:

~~~csharp
    // C style (parens and braces required)
    if (c) { a; } else { b; };
    // Rust style (only braces required)
    .if c { a; } else { b; };
~~~

For Wasm, I've chosen to prefer the first syntax style for the contents of function bodies, and the second style outside functions. Note that the two forms have different call targets (the first one calls `if` while the second calls `` `.if` ``.)

Currently, keyword-expressions cannot start mid-expression, i.e. you can't write `x = .foo y {...}` and must write `x = (.foo y {...})` instead, but this restriction could be lifted.

Just so we're clear, continuators are not keywords, they are just words that you wouldn't expect to see after `}` except for the purpose of continuing the previous statement. The exact syntax and output tree for a continuator clause has not been finalized; here's one possibility:

~~~
    try { a; } catch (e: Exception) { b; };
    // ..could be equivalent to..
    try({ a; }, `'catch`(e: Exception, { b; }));
~~~

### Juxtaposition ###

LESv3 introduces a unary juxtaposition operator, which allows any unadorned identifier (i.e. no `$`, no `.`) to act like a (high-precedence) unary operator:

~~~
// These two lines are equivalent
$x = log sqrt $y * $z;
$x = log(sqrt($y)) * $z;
~~~

Juxtaposition is a handy feature, but it's a fallback. It's not compatible with everything else and if another interpretation applies, the other interpretation will be used instead. The identifier that you want to use as an operator can only be followed by an identifier or a nonnegative literal. For example:

~~~
foo 12     // Juxtaposition:      foo(12)
foo $x     // Juxtaposition:      foo($x)
foo -x     // Normal subtraction: (foo) - x
foo x.y    // Juxtaposition:      foo(x.y)
foo (x).y  // Normal call:        (foo(x)).y
foo $bar() // Juxtaposition:      foo($bar())
~~~

In the postorder code above, you may have noticed that the names of some operators have changed: `i32.lt_s` is now `i32'lt_s` and `i32.add` is now `i32'add`. That's because LES, like most other programming languages, defines dot (`.`) as an operator and not as part of an identifier. In most contexts the dot causes no trouble, but it's not compatible with juxtaposition notation since the left-hand side must be an identifier. The single-quote, on the other hand, is permitted in identifiers (it's treated the same way as a digit.) Alternatively we could use inderscores: `i32_add`. 

Technically it's _possible_ to parse code like this, dots and all:

    i32.eqz i32.clz $x

But I don't think we _should_, because it either increases the complexity of the LES parser's grammar from LL(2) to LL(*), or requires challenging tricks in grammar actions. In fact, my parser already relies on a couple of tricks and I want to avoid adding more (one trick efficiently supports an infinite number of operators with over 20 precedence levels; another is a flag that changes how `{braces}` work inside a `.dot` expression.)

Other options: we could avoid using the juxtaposition feature, or drop it from the language entirely.

What about opcodes like `i32.trunc_s/f64` and `i32.reinterpret/f32`? Naturally, slash is an operator, so it'll have to change. I suggest `i32'trunc_s_f64` and `i32'reinterpret_f32`.

### Labels ###

Remember that `:` is a _binary_ operator, as in `$x: i32`. How is it possible to also use it for labels? The answer is not simple. In fact, I originally planned to use labels with a colon _prefix_, as in `:label`, but if newlines do not act as end-of-statement markers, then this plan doesn't really work as I will explain later. So I gave it some thought and found a combination of novel tricks that will allow the familiar `label:` syntax.

First, I realized that if `:` has a higher precedence on the left side than the right side, it be used for labels and produce a sane syntax tree. This is basically the same trick used in C# so that

    f = a => b = a; // Lambda

parses as

    f = (a => (b = a));

Similarly, we can contrive precedence rules so that

    label1: 
      br loop 'if $length = $length + -1;
    label2:
      x = c ? a : b;

parses like

    label1: ((br loop) 'if ($length = ($length + -1)));
    label2: (x = (c ? (a : b)));

The colon is still treated as a binary operator, which leads us to the second trick: using a postprocessing step to separate the label from the expression that follows it. This shouldn't be a big deal: since labels don't really exist in Wasm anyway (only blocks), a postprocessing step is already needed.

The final problem is that `label:` is currently illegal if there is no expression afterward. To solve this, we can define a special rule that a colon can be used as suffix operator, but only if there is no expression on the right-hand side.

Frankly, this is a lot of trouble to get the familiar syntax. If we drop JSON and use newlines as terminators, I'm inclined to drop the tricks and use `:` as a prefix operator (which the parser already allows): `:label`. This isnt' very workable if newline is _not_ a terminator, though. For starters,

    :label;

is fairly ugly. But there's a deeper problem because `:` is also an infix operator. If the input is

    br stop 'if { $input '<s 1 }
    :label;

Then unless we treat the newline specially, this will get parsed in a silly way:

    (br stop) 'if ({ $input '<s 1 } : label);

Note that labels, unlike local variables, don't need for a `$` to mark them as labels, since they appear in restricted contexts and there's no harm in defining a label named after an opcode (e.g. `grow_memory:`).

**Update:** I should point out that if we use newlines as terminators, only one trick is required: allowing colon as a suffix at the end of an expression.

An uglier alternative is to avoid the colon entirely. For example, if `#` is defined as an ordinary identifier character (like a letter of the alphabet) but is reserved for use by labels, the following code is parsed as two separate statements as we desire:

    br #stop 'if { $input '<s 1 }
    #label;

### Custom/alphanumeric operators ###

LES has always had an "infinite" number of operators, since any combination of operator characters (`[!%^*+=|<>/?:.$&~-]`) can be an operator, whose precedence is chosen by [baked-in rules](https://github.com/qwertie/ecsharp/blob/master/Core/Loyc.Syntax/LES/LesPrecedence.cs).

Outside Wasm, you would expect `r>s` to be parsed as `r > s`. But Wasm integers are not signed or unsigned, so Wasm has an unusually good reason to put letters into operators to indicate their signedness. To resolve this tension between Wasm and "normal" languages, LES does allow letters in operators, but only if the operator is specially marked.

In the new syntax, an operator can contain both operator characters and identifier characters (`[A-Za-Z0-9_]`, excluding the single quote) if it begins with a single quote. Thus `$f() '>s g()` is a signed comparison.

Single quotes can either start a character literal like `'A'` or an operator like `'A`; the difference between them is the lack of a closing quote (three characters of lookahead are sufficient to distinguish between them.)

Alphanumeric operators allow you to construct sentence-like expressions, as you saw in the example:

~~~
    br stop 'if $input '<s 1;
~~~

I recently changed the precedence rules so that an operator like `<=s` has the precedence you would expect.

"Word" operators like `'if`, have two possible precedences depending on whether they start with a lowercase letter. If the operator starts with a lowercase letter, it has very low precedence and is right-associative. This example would be parsed like so:

~~~
    (br stop) 'if ($input '<s 1);
~~~

Upon seeing the `'if` operator, the assembler would look for the `br` call in its first child node to determine that it's a `br_if` operation.

The above example produces no value. Here are some ways that other `br` constructs could be encoded in LES:

    br exit;                     // unconditional branch, no value
    br exit($value);             // unconditional branch with value
    br exit 'if condition;         // conditional branch, no value
    br exit($value) 'if condition; // conditional branch with value
    // Branch tables (syntax is designed to put index after value)
    br         'table $index [a, b, c] | d; // no value
    br($value) 'table $index [a, b, c] | d; // with value

This assumes you can use AST notation. In case AST notation is not possible because of a "void" value between the opcode and its values, alternate syntax(es) are needed. My current idea is to have a pseudo-opcode `pop` and `pop N` that represent one or more values to pop. Then one could write, for instance,

    $result_value();
    $condition();
    drop $do_something_else();
    br exit(pop) 'if pop;

Instead of or in addition to a "friendly" branch syntax, I think there should be a "basic" form, at least one that omits all stack elements:

    br_if exit;
    br_table(1 /*arity*/, [a, b, c], d /*default branch target*/);

Currently, operators that are `'words` cannot be used as prefix or suffix operators, e.g. `'drop $foo()` is illegal. This restriction leaves the door open to having `'` as a prefix for arbitrary continuators in keyword-expressions (to avoid ambiguity between unary `'word-ops` and continuators).

### Attributes ###

"Attributes" are nodes that are independently attached to other nodes. Any node in the syntax tree (whether it's an identifier, literal or call) can have associated attributes. This allows attributes to store "out-of-band" metadata such as debug information and comments, and "attributes" in the C# sense or "annotations" in the Java sense.

Currently, attributes can only appear at the very beginning of an expression and they apply to the whole expression. They consist of an @ sign followed by a "particle" (identifier, literal, braces, brackets or parentheses). The following example shows the kinds of attributes you can write:

~~~java
    // Identifier as attribute
    @Mathy .fn Square(x: i32) { x * x; }
    // Literal as attribute
    @"Increment" $x := $x + 1;
    // Braces or list as attribute
    @{a;b;c} set_local($x, 1);
    @[a,b,c] set_local($x, 1);
    // Arbitrary expression as attribute
    @(f(x, y)) ($x = 1);
    // Multiple attributes on a single expression
    @123 @0x123 @Numeric 
    $x = 1 + $x;
~~~

Currently, in order to apply an attribute to a child node, parentheses are required. Inside parentheses, a new expression is considered to begin:

~~~java
   $x = (@plus (@why $y) + (@zed $z));
~~~

**Note**: Java annotations can have an argument list, e.g. `@Foo(123)`. In LES this is not allowed, because (unless we pay attention to whitespace) there would be an ambiguity between `@Foo(123) (f())()` (`Foo` has an argument list) and `@Foo (f())()` (no argument list intended). You must write `@(Foo(123))` instead.

### Custom Literals ###

This isn't especially important for Wasm, but I'd like to mention my plan for literals. As a format for future programming languages, LES should have a mechanism to allow new kinds of literals to be defined. Meanwhile, LES parsers that don't support a given kind of literal should be able to round-trip it from text to text.

In addition to `true`, `false`, `null` and character literals, LES has three other literal syntaxes:

1. Numbers with optional type suffix, e.g. `-1234`, `0x1234_5678_9ABCi64`
2. Strings with optional type prefix, e.g. `"Hello, world!"`, `s"symbol"`
3. `@@` literals, e.g. `@@nan.f`, which is intended for named literals and includes booleans like `@@false` and inifinities like `@@inf.d`. These literals are parsed the same way as the single-quoted operators introduced above.

LES unifies these three kinds of literals into a single concept. When scanning any of these literals, an LES parser can store the literal in a type like this:

~~~js
// ES6
class CustomLiteral {
    // type prefix or suffix
    public typeMarker;
    // An LES parser is allowed to keep all values in string form, 
    // but if the type marker is known to be numeric or if it was
    // written as a numeric literal, it is acceptable to parse it 
    // into a number and store that here instead.
    public value;
};
~~~

A custom literal need not keep track of whether the literal was originally written as a number or as a string, because numeric and string literals are equivalent. The string `u"0x12345"` is an acceptable way to express the number `0x12345u`, while the number  number `1234.5` is equivalent to the string `number"1234.5"` (i.e. `number` is the default suffix for numbers if one is not present). Finally, an unknown `@@` literal like `@@hello-world!` can be expressed as the string `` `@@`"hello-world!"`` (a type prefix/suffix can be any identifier including a backquoted one.)

While the printer is always allowed to print a literal back out in the form of a string, for readability it is recommended to print it as a numeric literal if the literal is "known" to be a number (e.g. the type marker is recognized as numeric or the LES implementation tracks number-ness. Just be careful not make invalid output by miscategorizing a string as a number; I can imagine security risks from such a mistake.)
 
Proposed literal type markers (not necessarily supported by Wasm):

- `i` and/or `i32`: 32-bit integer (which do you prefer?)
- `L` and/or `i64`: 64-bit integer (which do you prefer?)
- `u` and/or `u32`: 32-bit unsigned integer
- `uL` and/or `u64`: 64-bit unsigned integer
- `f` and/or `f32`: 32-bit floating point
- `d` and/or `f64`: 64-bit floating point
- `z`: unlimited-size integer
- `c`: character (e.g. `c"X"` means `'X'`)
- `s`: symbol (e.g. `s"foo"` would be `:foo` in Ruby and `Symbol.for("foo")` in ES6)
- `re"[a-zA-Z0-9]"`: regular expression
- `m`: decimal (.NET 102-bit number with floating _decimal_ point)
- `` `@@` ``: named literal (multiple data types)

Ordinary strings do not have a type marker, but we could reserve the empty type marker ``` `` ``` for strings.

Other ideas:

- `over`_N_ : rational number with denominator N, e.g. `3over4`
- `ux`, `ua`, `ub`: possible codes for exact [unum](http://motherboard.vice.com/read/a-new-number-format-for-computers-could-nuke-approximation-errors-for-good) and interval [unum](http://www.johngustafson.net/presentations/Multicore2016-JLG.pdf) (`ua` above, `ub` below). 

If the parser recognizes the type marker but the value fails to parse into that type (e.g. `0xFFFF0000i32`, which overflows), the parser may print an error and should store it as a string in a `CustomLiteral` object.

### Room to grow ###

The following syntactic elements are unused, allowing potential future use:

- Non-ASCII characters (currently supported only in strings).
- `\` (backslashes are not used outside strings).
- `'{...}`, `'(...)`, `'[...]`, `@@[...]`, `@@(...)`.

Plus:

- It's not clear what to do with `#`. In LESv2 it was an ordinary identifier character, treated the same as a letter of the alphabet.
- `@` only appears at the beginning of an expression, for attributes. What if it appears later? One idea is to also allow it as a suffix, as in `size = 64@KB + x`. The effect would still be used to attach an attribute, but the suffix version would bind more tightly, equivalent to `size = (@KB 64) + x`.
- Certain pairs of operators are immiscible (cannot be mixed), like `x & 1 == 0`, which illustrates Dennis Ritchie's [C precedence mistake](www.lysator.liu.se/c/dmr-on-or.html). A future version could lift the immiscibility rule while raising the precedence of `&` to what it should have been all along. Another example is `x << 1 + 1`, which a developer might think of as "x times two plus one" but in C is `x << 2`.

### Miscellaneous issues ###

- One possible friendly syntax for loads and stores would be `type[address, offset]` and `type[address, offset] := value` respectively, e.g. `f32[$addr, 4] := 0xFFp0`. This should probably be in addition to an ugly "base" form like `f32'store(12 /*offset*/, 1 /*alignment*/)`.
- In all the examples so far, local variables have started with `$` to avoid all potential _future_ conflicts between variable names and opcode names. The alternative, of course, is to specially mark opcode names instead. This makes some sense, as the text format will rarely use named opcodes, preferring instead to use operators like `=` and `+`. On the other hand, numeric "identifiers" like $2 need to be specially marked anyway to distinguish them from integers, so perhaps removing the `$` has little benefit. Then again, users might often rely on debug information to eliminate such anonymous variables, or even heuristics (e.g. a debugger could detect that `$2` is used as a pointer and name it `ptr2` instead.)
- For readability, LES supports digit separators. Two digit separators are possible: `_` as in `0x6789_ABCD + 123_456_789`, or `'` as in `0x6789'ABCD + 123'456'789`. Which do you prefer?
- Should `/* /* foo */ */` be one nested comment, or one comment plus a `*/` operator?
- Probably `$` should only be allowed at the beginning of an operator, so that `-$x` is a negation of `$x` rather than a single `-$` operator.
- What should the syntax be for an invalid UTF-8 byte? My original idea was `\uNNNN` and `\uNNNNN` for unicode, `\xNN` for latin-1 characters and `\?NN` for a byte that is invalid UTF-8. But now I'm leaning toward `\u` for unicode and `\xNN` for any single byte, including valid UTF-8 control characters and invalid bytes above 127.
- If a numeric literal starts with a dot as in `.125`, this perhaps should be treated as an error, because `x+.125` would parse as `x +. 125`, probably not what the user wanted.

Conclusion
----------

I'm finishing up my parser and unit tests for LESv3 in C#. Before writing/porting parsers for other langauges I'd like to settle some of the open questions:

### Questions for the Community ###

- Are there any elements of LESv3 - or the way I've suggested Wasm be encoded in LES - that you disagree with, or don't understand?
- Newlines: should they terminate statements? If so, how can LES code show its intent for an expression to span multiple lines? (My thoughts in brief: newlines don't count inside `()` or `[]`, or immediately after `{` or an infix operator. For other situations we'll need a line continuation marker such as `\`.)
- If the name of `i32.trunc_s/f64` must be changed to make it into an ordinary identifier, what name should it have instead? `i32_trunc_s_f64`? `i32'trunc_s_f64`?
- Is it important to support non-ascii identifiers like `ThíŝÖnè` in the LES standard? If so, can the standard be written in such a way that all LES parsers recognize and reject the same set of characters as letters? What about [normalization](http://unicode.org/reports/tr15/)? (I'm inclined to say no, because arbitrary identifiers are already supported as backquoted strings.)
- Do you prefer that keyword-expressions begin with an explicit marker as in `.fn X() {}`, or would the implicit design used in LESv2 be better, as in `fn X() {}`? (**Note**: the implicit design is incompatible with juxtaposition expressions, and it produces a syntax error for input like `foo (x, y)`, as explained earlier.)
- Should keyword-expressions support comma-separated arguments despite the ambiguity?
- Labels: should we avoid the extra rule(s) required to support colon-terminated `labels:`? We can use colon as a prefix instead of a suffix if the parser is newline-sensitive. Another option is to use a block statement like `block(label) {...}`, but I am not in favor because it can lead to excessive nesting of braces and puts the label at the top instead of its logical location at the bottom.
- Should the hash sign `#` be treated as a normal identifier character. If not, should it be reserved for future use?
- Continuator clauses: besides `if`, `elsif`, `elseif`, `catch` and `finally`, what should the set of continuators include? Expanding this set in the future would break backward compatibility, so it should be generous from the start. I'm inclined to include `where`, the conjunctions `and or but so then`, and some English prepositions, especially short ones like `to`, `on`, `at` and `via`. Note that the set cannot include `while` and `for` since these already tend to be used at the beginning of expressions.
- Which is better: `123L` or `123i64`? Or should both be supported?
- What infix operators should represent `set_local` and `tee_local`?
- Any comments on the "miscellaneous issues"?
- And the $64,000,000 question: are you in favor of using LES for the Wasm text format? Before or after MVP?
