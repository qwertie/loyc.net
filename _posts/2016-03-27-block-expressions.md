---
title: Adding "quick binding" to Enhanced C#, part 1
layout: post
---

The Common Subexpression Problem
--------------------------------

Many years ago I noticed a common pattern with my "if" statements. I would use a complex or computed expression of some sort - usually a method or property call - and then, soon afterward, I would want to use the same expression again. For example:

~~~csharp
if (list[i].SubItems.Count > _threshold)
   Foo(list[i].SubItems);
else {
   ...
}
~~~

When I'm coding, this pattern seems to happen multiple times a day. And so I have to rewrite it as

~~~csharp
var subitems = list[i].SubItems;
if (subitems.Count > _threshold)
   Foo(subitems);
else {
   ...
}
~~~

This takes time. Plus, in my opinion, the new code is not as readable as the original code. The readability problem in this particular case is quite small, but it tends to grow as the complexity of the "if" condition grows (you'll see that soon).

The necessary refactoring is:

1. select the subexpression,
2. cut it to the clipboard, 
3. choose a variable name (`subitems`) and write it in place of the original expression,
4. insert a new line, and
5. add `var subitems = ` and paste the subexpression.

Sometimes it's a lot more complicated, though. Consider a slightly modified version of the original code:

~~~csharp
if (i < list.Count && list[i].SubItems.Count > _threshold)
   Foo(list[i].SubItems);
else {
   ...
}
~~~

The instructions above would give us code that is clearly wrong:

~~~csharp
var subitems = list[i].SubItems; // ArgumentOutOfRangeException!!!
if (i < list.Count && subitems.Count > _threshold)
   Foo(list[i].SubItems);
else {
   ...
}
~~~

Instead we have to "move up" the test for `i < list.Count`. Something like this:

~~~csharp
if (i < list.Count) {
   var subitems = list[i].SubItems;
   if (i < list.Count && subitems.Count > _threshold)
      Foo(list[i].SubItems);
   else {
      ...
   }
}
~~~

This is _still_ wrong, though. See the problem? It's the `else` clause. The `else` clause is supposed to run if _either_ of the `if` conditions are false. So now we have to refactor it again... maybe something like this:

~~~csharp
SubItemType subitems;
bool flag = i < list.Count;
if (flag) {
   subitems = list[i].SubItems;
   flag = subitems.Count > _threshold;
}
if (flag)
   Foo(subitems);
else {
   ...
}
~~~

Wow! That's ugly. Sometimes we have to go to great lengths just to factor out a common subexpression. And there's a compiler error hidden in this refactoring - can you spot it?  The anti-readability of this code is obvious, since one line of code has ballooned to 7.

As a real life example, consider this code, which I managed to find in a matter of seconds when I started looking through my own code:

~~~csharp
static Symbol ChooseFieldName(Symbol propName)
{
   string name = propName.Name;
   char first = name.FirstOrDefault();
   char lower = char.ToLowerInvariant(first);
   if (lower != first)
      name = lower + name.Substring(1);
   return GSymbol.Get("_" + name);
}
~~~

In this function I've explicitly factored out common subexpressions into local variables. If I hadn't done that, the code would have looked like this:

~~~csharp
static Symbol ChooseFieldName(Symbol propName)
{
   string name = propName.Name;
   if (char.ToLowerInvariant(name.FirstOrDefault()) != name.FirstOrDefault())
      name = char.ToLowerInvariant(name.FirstOrDefault()) + name.Substring(1);
   return GSymbol.Get("_" + name);
}
~~~

Even though this code is longer and will run slower, it is, to me at least, slightly easier to understand. I think that the variable declarations for `first` and `lower` are a cognitive burden for the reader because some of the context information has been removed: when you look at `first`, you have no idea what its purpose is. The `if` statement is what gives a purpose to this variable, but the `if` statement doesn't appear until two lines later. Therefore, it takes you longer to see that the code converts the first letter of `name` to lowercase (but only if the first letter changes as a result).

Solution
--------

Some languages, such as Go, have a `:=` operator that [creates and assigns a variable at once](http://stackoverflow.com/questions/16521472/assignment-operator-in-go-language). In C#, the natural equivalent of `:=` would be

~~~csharp
if ((var subitems = list[i].SubItems).Count > _threshold)
   Foo(subitems);
else {
   ...
}
~~~

But with the extra parentheses and everything, it is a little unweildy. Several years ago I found the optimal solution to this problem. It looks like this:

~~~csharp
if (list[i].SubItems::subitems.Count > _threshold)
   Foo(subitems);
else {
   ...
}
~~~

I call `::` the "quick binding operator". The `::` operator already exists in C#, but you could code for years without ever using it; I'm just proposing that we "overload" this operator with a new behavior, whenever its original behavior does not apply.

Using `::` is not just shorter, it has beter workflow, too. As soon as you write `Foo`, you notice that you're using the same expression again. So you simply go to the line above and add `::subitems` to save it to a temporary variable. No cutting and pasting, and the name "subitems" is repeated only twice, not three times as in the original code. Plus, the example above that uses `&&` stays simple:

~~~csharp
if (i < list.Count && list[i].SubItems::subitems.Count > _threshold)
   Foo(subitems);
else {
   ...
}
~~~

In my opinion, `subitems` should (in general) also be available in the `else` clause, and even after the end of the `if` statement. After all, if you were writing the code by hand, it would be, and it's more generally useful if it's accessible afterward. So, should it be scoped to the first part of the `if` statement, the entire `if` statement, or to the outer block? Well, this seems like a decision I can put off until later.

Here's how it looks for the `ChooseFieldName` example above:

~~~csharp
static Symbol ChooseFieldName(Symbol propName)
{
   string name = propName.Name;
   if (char.ToLowerInvariant(name.FirstOrDefault()::first)::lower != first)
      name = lower + name.Substring(1);
   return GSymbol.Get("_" + name);
}
~~~

Implementing this in EC#
------------------------

The motivation to support this feature goes beyond simple variable declarations, and it goes beyond the code you write yourself. Consider the `?.` operator. Today the `?.` is part of C# 6, but before that it was implemented as a LeMP macro for [Enhanced C#](http://ecsharp.net). When you wrote

    Foo(Bar?.Baz);

A macro would convert this to

    Foo(Bar != null ? Bar.Baz : null);

But of course, this translation could be wrong: `Bar` is evaluated twice, but it should only be evaluated once. Originally I planned to solve this using a feature called "block expressions". The `?.` macro would generate code like this:

    Foo({ var Bar_13 = Bar; Bar_13 != null ? Bar_13.Baz : null });

**Note**: `13` would be a global counter used to produce unique variable names.

Notice that the final expression has no semicolon, which is used to mean "return a value from here"; it's an idea copied directly from [Rust](https://www.rust-lang.org/). Originally I was going to use this same syntax to simplify function return values:

    static double Square(double x) { x*x }

However, this syntax is a bit redundant now that C# 6 has lambda-style functions:

    static double Square(double x) => x*x;

So now I'm now thinking it's better to avoid complicating the parser with new syntax that we don't really need.  Instead I'm thiking of using this syntax instead, which re-uses the `in` operator that is already used for other purposes:

    Foo((Bar_13 != null ? Bar_13.Baz : null) in { var Bar_13 = Bar; });

The `in {...}` operator would be a bit like the `where` clause in Haskell (it's just easier to call it `in` because `in` is a keyword, and `where` is not.) The second part (in the braces) runs first, and then the first expression gets the final result. Arguably it's better to have a left-to-right execution order, so we could consider other syntaxes, like

    Foo({ var Bar_13 = Bar; out Bar_13 != null ? Bar_13.Baz : null; });

or

    Foo({ var Bar_13 = Bar; } => Bar_13 != null ? Bar_13.Baz : null);

It may look like these are "new" syntaxes, but they're not. Both of them are re-using pre-existing parsing rules that other macros are already relying on.

All of these possibilities have an... issue. By using braces, the implication is that a new scope is created, so the variable `Bar_13` should exist within that scope and disappear afterward. However, we need to access the value of `Bar_13` after the scope has ended so that we can get its value out.

I had thought of dealing with this problem with "variable renaming". The idea is: "let's eliminate the braces so we can use `Bar_13` in the call to `Foo`". Rather than actually using braces to create a new scope, we'll strip out the braces, but give each variable within the braces a new name so it doesn't conflict with anything outside. C# doesn't allow you to declare anything other than variables in an executable context, so renaming variables is sufficient (we need not watch out for methods and properties, for instance). However, in this particular case, `Bar_13` is already a unique name because it's a compiler-generated variable. This leads me to ask: hang on, do we really need the braces at all?

The braces provide an obvious syntax for saying "I want to execute a statement inside an expression". However, end-users don't really need this feature; it's intended mainly as a mechanism to help macros work. So now I'm thinking, let's forget the braces and just have a pseudo-function called `#runSequence` or something like that:

    Foo(#runSequence(var Bar_13 = Bar, Bar_13 != null ? Bar_13.Baz : null));

Now, how is all this related to our `::` quick binding operator?

Well, given a binding like this:

~~~csharp
if (list[i].SubItems::subitems.Count > _threshold)
~~~

It can be rewritten as 

~~~csharp
if (#runSequence(var subitems = list[i].SubItems, subitems).Count > _threshold)
~~~

Therefore, it can be handled the same way as any other sequence of statements that a macro might produce.

Whatever syntax we use, actually implementing it will be challenging. More on that when I write part 2.
