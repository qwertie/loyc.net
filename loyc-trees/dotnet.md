---
layout: article
title: Using Loyc trees in .NET (C#)
date: 5 Nov 2016 (updated Feb 2019)
toc: true
---

You can create Loyc trees programmatically using the [`LNodeFactory`](http://ecsharp.net/doc/code/classLoyc_1_1Syntax_1_1LNodeFactory.html) class. You have to provide a "source file" object that will be associated with each of the [`LNode`s](http://ecsharp.net/doc/code/classLoyc_1_1Syntax_1_1LNode.html) created by the factory; since you are creating nodes programmatically, just use `EmptySourceFile.Default` or create a `new EmptySourceFile("my source file's name")`. (If you feed your nodes into a compiler programmatically, the source file name may be used by the compiler to display error messages regarding the nodes you created.)

An `LNodeFactory` is often named `F`:

~~~csharp
LNodeFactory F = new LNodeFactory(new EmptySourceFile("Foo.cs"));
 
// Create a call to foo(xyz, 123)
LNode callFoo = F.Call("foo", F.Id("xyz"), F.Literal(123));
 
// Create a function definition: void foo(int x, string y) { return; }
LNode fooDecl = F.Fn(F.Void, F.Id("foo"), 
                     F.Tuple(F.Var(F.Int32, F.Id("x")), F.Var(F.String, F.Id("y"))),
                     F.Braces(F.Call(S.Return)));
~~~

An easier way to create nodes is to parse LES or EC# code, although this can be costly because it happens at runtime. If you're using EC# then you can use [`quote {...}`](http://ecsharp.net/lemp/ref-other.html#quote) to produce code at compile time that will construct a Loyc tree directly.

To parse EC# into a Loyc tree, call `Loyc.Ecs.EcsLanguageService.Value.Parse("your code here;")` (this is an extension method so you'll also need `using Loyc.Syntax;`). To treat a node (or list of nodes) as EC# code and print them with the EC# printer, call `EcsLanguageService.Value.Print(node)`. Don't worry, the EC# printer is fairly reliable at printing Loyc trees that do not represent EC# code; Loyc trees almost always round-trip correctly from a Loyc tree to EC# code and back.

Similarly, to parse LES into a Loyc tree, call `LesLanguageService.Value.Parse("your expression here;")`.

The LES printer is the default when you call `LNode.ToString()`. To use EC# as the default instead, set `LNode.Printer` to `EcsLanguageService.Value.Printer`, or better yet, use code like this:

~~~csharp
    using (LNode.PushPrinter(EcsLanguageService.Value.Printer)) {
      /* print Loyc trees with `tree.ToString()` here */
    }
~~~

Important properties of Loyc nodes
----------------------------------

A Loyc tree node (`LNode`) consists of the following main properties:

* `Range`: tuple of (`ISourceFile Source`, `int StartIndex`, `int Length`)
* `Attrs`: a list of attributes
* One of: `Value` (the value of a literal node), `Name` (the name of an identifier node), or `Target` and `Args`, which are child `LNode`s (e.g. The `Target` of `f(1, a)` is `f` and the `Args` list contains two nodes `1, a`). Note: the `Name` property is special because it works for simple calls in addition to identifiers, e.g. the name of `foo(x)` is `foo`.
* The `Kind` property returns the node type: `NodeKind.Literal`, `NodeKind.Id`, or `NodeKind.Call`. However, it's almost always easier to call one of the three test properties `IsLiteral`, `IsId` or `IsCall`.
* `Range`: indicates the source file that the node came from and location in that source file.
* `Style` an 8-bit flag value that is used as a hint to the node printer about how the node should be printed. For example, a hex literal like `0x10` has the `NodeStyle.Alternate` style to distinguish it from decimal literals such as 16. Custom display styles that do not fit into the `Style` property should be expressed with trivia attributes.

Identifier names are stored in `Symbol`s; a `Symbol` is a singleton string. One purpose of symbols is performance; in order to compare `"foo1" == "foo2"`, the actual characters much be compared one-by-one. `Symbol` comparisons, on the other hand, are lightning-fast reference comparisons. To get a symbol from a string `s` in C#, just use a cast: `(string) s`. To get the string out of a `Symbol`, call the `Symbol.Name` property or `ToString()` which is an alias for `Name`.

Common symbols for keywords and datatypes are defined in the `Loyc.Syntax.CodeSymbols` class in Loyc.Syntax.dll. Throughout the Loyc codebase, a `using S = Loyc.Syntax.CodeSymbols;` statement is often used to abbreviate `CodeSymbols` as `S`, so you can write things like `S.While` (which represents the `#while` symbol), `S.Class` (`#class`) `S.And` (`&&`), `S.AddSet` (`+=`), and so on. See the [source code of CodeSymbols](https://github.com/qwertie/ecsharp/blob/master/Core/Loyc.Syntax/CodeSymbols.cs).

You should also be aware of these helper methods:

* `IsIdNamed(Symbol name)`: returns true if the node is an identifier with the specified name.
* `Calls(Symbol name, int argCount)`: returns true if the node calls the specified name with the specified number of arguments, e.g. if I create a call with `c = F.Call("x", F.Literal(123))` then `c.Calls(GSymbol.Get("x"), 1)` is true.
* `CallsMin(Symbol name, int argCount)`: returns true if the node calls the specified name with the specified _minimum_ number of arguments.
* `HasPAttrs()`: returns true if the node has any "printable", meaning non-trivia, attributes attached to it.
* `IsParenthesizedExpr()`: checks for a `#trivia_inParens` attribute.
* `Descendants()`, `DescendantsAndSelf()`: enumerates the children of a node.
* `ReplaceRecursive(node => {...})` performs a find-and-replace operation, as discussed below.

Node comparisons with `Equals()` test for structural equality rather than reference equality, and tend to be expensive. `GetHashCode()` can be expensive when first called, but the hashcode is cached.

Loyc tree imitators
-------------------

Any .NET class can "pretend" to be a Loyc tree by implementing the `ILNode` interface. By implementing this interface you can print your custom syntax tree as if it were a Loyc tree.

`ILNode` is a read-only interface that omits much of the functionality of `LNode` itself; most notably, it is meant for read-only use and has none of the "With" or "Plus" methods for creating modified nodes. This interface also lacks the `Attrs` and `Args` properties, although they are available as the extension methods `Attrs()` and `Args()`. `ILNode` has an indexer, which allows you to access both arguments and attributes as explained [here](http://loyc.net/loyc-trees/#single-list-perspective).

Most code in the Loyc libraries is designed to work with `LNode`, not `ILNode`, but the LESv2 and LESv3 node printers can print `ILNode` objects directly, and there is a method, `LNodeExt.ToLNode(ILNode)`, to convert any `ILNode` to `LNode`.

Implementing this interface requires a couple of methods that might seem challenging at first. For example, `ILNode` inherits [`INegListSource`](http://ecsharp.net/doc/code/interfaceLoyc_1_1Collections_1_1INegListSource.html) from the [Loyc.Essentials library](http://core.loyc.net/essentials/), which has a `Slice()` method that returns an `IRange` object - where the heck can you get one of those? That's actually easy, you can just use the same implementation as `LNode` itself, which creates a `NegListSlice` adapter that implements `IRange`:

~~~csharp
IRange<ILNode> INegListSource<ILNode>.Slice(int start, int count)
{
    return new NegListSlice<ILNode>(this, start, count);
}
~~~

Another challenge you'll face is the `Range` property which is supposed to return the range of characters in the source file from which the node originated. That's easy if you happen to be using [`Loyc.Syntax.SourceFile<CharSource>`](http://ecsharp.net/doc/code/classLoyc_1_1Syntax_1_1SourceFile.html) to represent your source files, but if not you could fake it using a dummy source file such as `EmptySourceFile.Unknown` or a `new EmptySourceFile`:

~~~csharp
// Assuming you have fields _fileName, _startIndex, _endIndex
public SourceRange Range => 
  new SourceRange(new EmptySourceFile(_fileName),
    _startIndex, _endIndex - _startIndex);
~~~

Let's go through a full example to see how to implement `ILNode`.

Let's say your custom syntax tree is rooted at `Expr`, and among its many nodes is `Number` and `AddOperator`, with base classes `Literal` and `BinaryOperator` respectively. So you have something like this:

~~~csharp
abstract class Expr {
   // Location of the expression in source code
   protected string _filename = "Unknown";
   protected int _startIndex, _endIndex;
   ...
}

abstract class Literal : Expr {
}

class Number : Literal {
   double _n;
   public Number(double n) { _n = n; }
   ...
}

abstract class BinaryOperator : Expr {
   protected BinaryOperator(Expr lhs, Expr rhs) 
      { _lhs = lhs; _rhs = rhs; }
   protected Expr _lhs, _rhs;
   public Expr Left => _lhs;
   public Expr Right => _rhs;
}

class AddOperator : BinaryOperator {
   public AddOperator(Expr lhs, Expr rhs) : base(lhs, rhs) { }
   ...
}
~~~

You should implement most of the interface in your base class(es), with just a few overrides in derived classes as necessary:

~~~csharp
using Loyc;
using Loyc.Syntax;

abstract class Expr : ILNode
{
   protected string _filename = "Unknown";
   protected int _startIndex, _endIndex;

   #region ILNode implementation

   public abstract LNodeKind Kind { get; }
   public abstract Symbol Name { get; }
   public abstract ILNode Target { get; }
   public abstract object Value { get; }
   public abstract int Min { get; }
   public abstract int Max { get; }
   public abstract ILNode TryGet(int index, out bool fail);

   public ILNode this[int index] {
      get {
         bool fail;
         var r = TryGet(index, out fail);
         if (fail) throw new ArgumentOutOfRangeException("index");
         return r;
      }
   }
   public IEnumerator<ILNode> GetEnumerator()
   {
      for (int i = Min; i <= Max; i++)
         yield return this[i];
   }

   public int Count => Max - Min + 1;
   public SourceRange Range 
      => new SourceRange(new EmptySourceFile(_filename), 
             _startIndex, _endIndex - _startIndex);
   NodeStyle ILNode.Style { get => NodeStyle.Default; set {} }
   object IHasLocation.Location => Range;
   public bool Calls(Symbol name, int argCount)
      => Name == name && Max + 1 == argCount;
   public bool CallsMin(Symbol name, int argCount)
      => Name == name && Max + 1 > argCount;
   public bool Equals(ILNode other) 
      => other == this;
   IEnumerator IEnumerable.GetEnumerator() 
      => GetEnumerator();
   public IRange<ILNode> Slice(int start, int count = int.MaxValue)
      => new NegListSlice<ILNode>(this, start, count);
   public LNode ToLNode() => LNodeExt.ToLNode(this);

   #endregion
}

abstract class Literal : Expr
{
   #region ILNode implementation

   public override LNodeKind Kind => LNodeKind.Literal;
   public override Symbol Name => GSymbol.Empty;
   public override ILNode Target => null;
   public override abstract object Value { get; }
   // Literals always have Max==-2 since they have no Target nor Args
   // (Target would have index -1, Args start at index 0). Assuming 
   // no attributes, Min should be -1 so that the number of children
   // is zero (given that Count is defined as Max - Min + 1).
   public override int Min => -1;
   public override int Max => -2;
   public override ILNode TryGet(int index, out bool fail) {
      fail = true; // No children so always fail
      return null;
   }
      
   #endregion
}

class Number : Literal
{
   double _n;
   public Number(double n) { _n = n; }

   public override object Value => _n;
}

abstract class BinaryOperator : Expr
{
   protected BinaryOperator(Expr lhs, Expr rhs)
      { _lhs = lhs; _rhs = rhs; }
   protected Expr _lhs, _rhs;
   public Expr Left => _lhs;
   public Expr Right => _rhs;

   #region ILNode implementation

   public override LNodeKind Kind => LNodeKind.Call;
   public override abstract Symbol Name { get; }
   // Assuming your syntax tree has identifiers, you could return 
   // one of those instead. Note: Range is wrong here, as the Target's
   // range should be the range of the target alone, e.g. the + sign
   // in an AddOperator. But for printing as LES, Range doesn't matter.
   public override ILNode Target => LNode.Id(Name, Range);
   // A Call has no value, not even null
   public override object Value => NoValue.Value;
   // If the node has no attributes, Min must be -1
   public override int Min => -1;
   public override int Max => 1;
   public override ILNode TryGet(int index, out bool fail)
   {
      fail = false;
      if (index == 0) return Left;
      if (index == 1) return Right;
      if (index == -1) return Target;
      fail = true;
      return null;
   }

   #endregion
}

class AddOperator : BinaryOperator
{
   public AddOperator(Expr lhs, Expr rhs) : base(lhs, rhs) { }

   public override Symbol Name => CodeSymbols.Add;
}
~~~

As you can see, although implementing `ILNode` requires a nontrivial amount of code, the "leaf" classes in your class hierarchy often require very little code per-class.

You should then be able to print your custom syntax tree like you would an `LNode`:

~~~csharp
var expr = new AddOperator(new AddOperator(
    new Number(1), new Number(2)), new Number(3));
Console.WriteLine(Les3LanguageService.Value.Print(expr));
// Output: 1.0 + 2.0 + 3.0
~~~

And, of course, you can call `expr.ToLNode()` to convert it to a real Loyc tree.

Modifying nodes
---------------

Since `LNode`s are immutable, you don't modify them directly. Instead you'll typically use one of the "`With`" (or `Plus`) methods to create modified nodes:

~~~csharp
 // For modifying Id nodes (WithName(x) can also be used with call
 // nodes; in that case it means WithTarget(Target.WithName(x))).
 public virtual  LNode WithName(Symbol name)
 
 // For modifying Literal nodes
 public abstract LiteralNode WithValue(object value);
 
 // For modifying Call nodes (note: you can add arguments to a non-call node,
 // which produces a call node.)
 public virtual  CallNode WithTarget(LNode target);
 public virtual  CallNode WithTarget(Symbol name);
 public abstract CallNode WithArgs(RVList<LNode> args);
 public virtual  CallNode With(LNode target, RVList<LNode> args);
 public          CallNode With(Symbol target, params LNode[] args);
 public          LNode PlusArg(LNode arg); // add one parameter
 public          LNode PlusArgs(RVList<LNode> args);
 public          LNode PlusArgs(IEnumerable<LNode> args);
 public          LNode PlusArgs(params LNode[] args);
 public          LNode WithArgChanged(int index, LNode newValue);
 
 // For modifying the attribute list
 public virtual  LNode WithoutAttrs()
 public abstract LNode WithAttrs(RVList<LNode> attrs);
 public          LNode WithAttr(LNode attr)
 public          LNode WithAttrs(params LNode[] attrs)
 public          LNode WithAttrChanged(int index, LNode newValue)
 public          CallNode WithArgs(params LNode[] args)
 public          LNode PlusAttr(LNode attr); // add one attribute
 public          LNode PlusAttrs(RVList<LNode> attrs);
 public          LNode PlusAttrs(IEnumerable<LNode> attrs);
 public          LNode PlusAttrs(params LNode[] attrs);
 
 // Other
 public         LNode WithRange(SourceRange range) { return With(range, Style); }
 public         LNode WithStyle(NodeStyle style)   { return With(Range, style); }
 public virtual LNode With(SourceRange range, NodeStyle style)
~~~

Argument lists are stored in [VList data structures](http://www.codeproject.com/Articles/26171/VList-data-structures-in-C).

### Find, find-and-replace, and pattern matching ###

`ReplaceRecursive(node => {...})` performs a recursive find-and-replace operation; see the [documentation](http://ecsharp.net/doc/code/classLoyc_1_1Syntax_1_1LNode.html#ad887a82823c62a4b4fc5c8bbc51b0603) for full details, but here is an example that replaces all instances of `0xFFFF` or `65535` with `ushort.MaxValue`:

~~~csharp
code = code.ReplaceRecursive(node => {
    if (node.Value is int && ((int)node.Value) == 0xFFFF)
        return LNode.Call(CodeSymbols.Dot, 
           LNode.List(LNode.Id(CodeSymbols.UInt16), LNode.Id("MaxValue")));
    return null;
});
~~~

If all you want to do is search for something, you can still use `ReplaceRecursive`; just return `null` to avoid creating new trees.

When using `ReplaceRecursive`, the `MatchesPattern` method is sometimes useful. This allows you to use a Loyc tree (e.g. supplied by an end-user) to specify what to search for. For example, the Loyc tree represented by this LES code:

    $x * $y + $z;

Lets you find multiplications followed by additions. For example:

~~~csharp
using Loyc;
using Loyc.Syntax;
using Loyc.Syntax.Les;

var pattern = LesLanguageService.Value.Parse("$x * $y + $z").First();
var code = LesLanguageService.Value.Parse(@"{
  x = a * b + c; 
  y = A * B + c * (p * q + r); 
}").First();

Symbol x = (Symbol)"x", y = (Symbol)"y", z = (Symbol)"z";
code = code.ReplaceRecursive(node => {
    IDictionary<Symbol, LNode> captures;
    if (node.MatchesPattern(pattern, out captures))
        return LNode.Call((Symbol)"MulDiv", LNode.List(captures[x], captures[y], captures[z]));
    return null;
});
Les3PrettyPrinter.PrintToConsole(code);
~~~

This produces the following output:
~~~csharp
{
  x = MulDiv(a, b, c);
  y = MulDiv(A, B, c * (p * q + r));
};
~~~
Notice that `p * q + r` has not been changed. That's because when the lambda returns a changed node, `ReplaceRecursive` does not continue searching into the changed node. Currently, the only way to accomplish that is to factor out the lambda into a variable and manually call `ReplaceRecursive` like this:

~~~csharp
Symbol x = (Symbol)"x", y = (Symbol)"y", z = (Symbol)"z";
Func<LNode, LNode> lambda = null; lambda = node => {
    IDictionary<Symbol, LNode> captures;
    if (node.MatchesPattern(pattern, out captures))
        return LNode.Call((Symbol)"MulDiv", LNode.List(captures[x], captures[y], captures[z]))
            .ReplaceRecursive(lambda);
    return null;
};
code = code.ReplaceRecursive(lambda);
~~~

### List/node duality ###

Often, a single node can be treated as a list. For example, in C-like languages you can have a single statement attached to a `while` loop, or a braced block:

~~~csharp
while (c) single_statement();
while (c) { zero_or_more_statements(); ... }
~~~

The `AsList` extension method helps you treat that single statement as a list:

~~~csharp
public static VList<LNode> AsList(this LNode block, Symbol listIdentifier)
~~~

If `block` is the second argument to `#while` then `block.AsList(CodeSymbols.Braces)` would return a one-item list containing `single_statement()` in the first example, and in the second example it would return a list containing the contents of the braced block. The inverse operation is `AsLNode`:

~~~csharp
public static LNode AsLNode(this VList<LNode> list, Symbol listIdentifier)
~~~

This returns `list[0]` if the list has one item; otherwise creates a node that calls `listIdentifier` with the specified argument list.

### Splicing ###

Occasionally a "splicing" operation is useful:

~~~csharp
// in LNodeExt class
public static VList<LNode> WithSpliced(this VList<LNode> list, LNode node, Symbol listName = CodeSymbols.Splice)
public static VList<LNode> WithSpliced(this VList<LNode> list, int index, LNode node, Symbol listName = CodeSymbols.Splice)
~~~

"Splicing" refers to conditionally inserting the arguments of one node into another node, if the node calls an identifier with a particular `Name`. For example, if you have a list `10, 11, 12` and a node `#splice(1, 2, 3)` then `list.WithSpliced(0, node)` returns the list `1, 2, 3, 10, 11, 12`. But if the node does not call `#splice` then it is simply added to the list, e.g. if the `node` is `foo(1, 2, 3)` then `list.WithSpliced(0, node)` returns the list `foo(1, 2, 3), 10, 11, 12`.

There are helper methods in `LNode` to do splicing:

~~~csharp
public CallNode WithSplicedArgs(int index, LNode from, Symbol listName)
{
    return WithArgs(LNodeExt.WithSpliced(Args, index, from, listName));
}
public LNode WithSplicedAttrs(int index, LNode from, Symbol listName)
{
    return WithAttrs(LNodeExt.WithSpliced(Attrs, index, from, listName));
}
~~~
