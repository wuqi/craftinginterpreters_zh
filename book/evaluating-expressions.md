> 你是我的创造者，但我是你的主人；服从我!
>
> <cite>Mary Shelley, <em>弗兰肯斯坦</em></cite>

如果你想为这一章营造一个合适的氛围，试着想象一场雷雨，那种在故事高潮时喜欢猛然拉开百叶窗的漩涡式暴风雨。也许再加上几道闪电。在这一章中，我们的解释器将呼吸，睁开眼睛，并执行一些代码。

<span name="spooky"></span>

<img src="image/evaluating-expressions/lightning.png" alt="A bolt of lightning strikes a Victorian mansion. Spooky!" />

<aside name="spooky">

一座破旧的维多利亚式豪宅是可选项，不过可以增加些氛围。

</aside>

对于语言实现来说，有各种方式可以使计算机执行用户的源代码命令。它们可以将其编译为机器代码，将其翻译为另一种高级语言，或者将其还原为某种字节码格式，以便在虚拟机中执行。不过对于我们的第一个解释器，我们要选择最简单、最短的一条路，也就是执行语法树本身。

现在，我们的解释器只支持表达式。因此，为了“执行”代码，我们要计算一个表达式时并生成一个值。对于我们可以解析的每一种表达式语法——字面量，操作符等——我们都需要相应的代码块，该代码块知道如何计算该语法树并产生结果。这也就引出了两个问题：

1. 我们要生成什么类型的值？

2. 我们如何组织这些代码块？

让我们来逐个击破...

## 描述值

在Lox中，<span name="value">值</span>由字面量创建，由表达式计算，并存储在变量中。用户将其视作Lox对象，但它们是用编写解释器的底层语言实现的。这意味着要在Lox的动态类型和Java的静态类型之间架起桥梁。Lox中的变量可以存储任何（Lox）类型的值，甚至可以在不同时间存储不同类型的值。我们可以用什么Java类型来表示？

<aside name="value">

在这里，我基本可以互换 "值 "和 "对象"。

稍后在C解释器中，我们会对它们稍作区分，但这主要是针对实现的两个不同方面（本地数据和堆分配数据）使用不同的术语。从用户的角度来看，这些术语是同义的。

</aside>

给定一个具有该静态类型的Java变量，我们还必须能够在运行时确定它持有哪种类型的值。当解释器执行 `+` 运算符时，它需要知道它是在将两个数字相加还是在拼接两个字符串。有没有一种Java类型可以容纳数字、字符串、布尔值等等？有没有一种类型可以告诉我们它的运行时类型是什么？有的! 就是老牌的java.lang.Object。

在解释器中需要存储Lox值的地方，我们可以使用Object作为类型。Java已经将其基本类型的所有子类对象装箱了，因此我们可以将它们用作Lox的内置类型：

<table>
<thead>
<tr>
  <td>Lox type</td>
  <td>Java representation</td>
</tr>
</thead>
<tbody>
<tr>
  <td>Any Lox value</td>
  <td>Object</td>
</tr>
<tr>
  <td><code>nil</code></td>
  <td><code>null</code></td>
</tr>
<tr>
  <td>Boolean</td>
  <td>Boolean</td>
</tr>
<tr>
  <td>number</td>
  <td>Double</td>
</tr>
<tr>
  <td>string</td>
  <td>String</td>
</tr>
</tbody>
</table>

给定一个静态类型为Object的值，我们可以使用Java内置的 `instanceof` 操作符来确定运行时的值是数字、字符串还是其他类型。换句话说，<span name="jvm">JVM</span>自己的对象表示方便地为我们提供了实现Lox内置类型所需的一切。当稍后添加 `Lox` 的函数、类和实例等概念时，我们还必须做更多的工作，但Object和基本类型的包装类足以满足我们现在的需要。

<aside name="jvm">

我们需要对值进行的另一项操作是管理它们的内存，而Java也可以做到这一点。方便的对象表示和优秀的垃圾回收器是我们选择用Java编写第一个解释器的主要原因。

</aside>

## 表达式求值

接下来，我们需要编写大量的代码来实现对每种可解析表达式的求值逻辑。我们可以将这些代码放入语法树类中，例如添加一个 `interpret()` 方法。然后，我们可以告诉每个语法树节点：“解释你自己”。这就是《设计模式》中的 [解释器模式][Interpreter design pattern]。这是一个整洁的模式，但正如我之前提到的，如果我们将各种逻辑都塞进语法树类中，它会变得混乱不堪。

[interpreter design pattern]: https://en.wikipedia.org/wiki/Interpreter_pattern

相反，我们将重用我们之前使用的[访问者][Visitor pattern]模式。在前一章中，我们创建了一个AstPrinter类。它接收一个语法树，并递归地遍历它，构建一个字符串，并最终返回它。这几乎就是一个真正的解释器所做的事情，只是解释器不是将字符串连接起来，而是计算值。

[visitor pattern]: representing-code.html#the-visitor-pattern

我们先创建一个新类。

^code interpreter-class

这个类声明自己是一个访问者。visit方法的返回类型将是Object，这是我们在Java代码中用来表示Lox值的根类。为了实现Visitor接口，我们需要为解析器生成的四个表达式树类中分别定义访问方法。我们将从最简单的开始...

### 字面量求值

表达式树的叶子节点是构成其他表达式的基本语法单元，它们被称为<span name="leaf">字面量</span>。字面量几乎已经是值了，但两者的区别很重要。字面量是一种产生值的语法单元。字面量总是出现在用户的源代码中的某个位置。而许多值是通过计算产生的，并不在代码本身的任何地方存在。它们不是字面量。字面量来自于解析器的领域。而值是一个解释器的概念，属于运行时的世界的一部分。

<aside name="leaf">

在[下一章节][vars]中，当我们实现变量时，我们将添加标识符表达式，它也是叶子节点。

[vars]: statements-and-state.html

</aside>

所以，就像我们在解析器中将字面量*标记*转换为字面量*语法树节点*一样，现在我们将字面量树节点转换为运行时值。这其实非常简单。

^code visit-literal

我们早在扫描过程中就即时生成了运行时的值，并把它放进了语法标记中。解析器获取该值并将其插入字面量语法树节点中，所以要对字面量求值，我们只需把它存的值取出来。



### 括号求值

The next simplest node to evaluate is grouping -- the node you get as a result
of using explicit parentheses in an expression.

下一个最简单的节点是分组节点——当你在表达式中使用显式括号时得到的节点。
下一个需要求值的节点是分组节点——当你在表达式中显式使用括号时产生的语法树节点。



^code visit-grouping

A <span name="grouping">grouping</span> node has a reference to an inner node
for the expression contained inside the parentheses. To evaluate the grouping
expression itself, we recursively evaluate that subexpression and return it.

We rely on this helper method which simply sends the expression back into the
interpreter's visitor implementation:

<aside name="grouping">

Some parsers don't define tree nodes for parentheses. Instead, when parsing a
parenthesized expression, they simply return the node for the inner expression.
We do create a node for parentheses in Lox because we'll need it later to
correctly handle the left-hand sides of assignment expressions.

</aside>

^code evaluate

### Evaluating unary expressions

Like grouping, unary expressions have a single subexpression that we must
evaluate first. The difference is that the unary expression itself does a little
work afterwards.

^code visit-unary

First, we evaluate the operand expression. Then we apply the unary operator
itself to the result of that. There are two different unary expressions,
identified by the type of the operator token.

Shown here is `-`, which negates the result of the subexpression. The
subexpression must be a number. Since we don't *statically* know that in Java,
we <span name="cast">cast</span> it before performing the operation. This type
cast happens at runtime when the `-` is evaluated. That's the core of what makes
a language dynamically typed right there.

<aside name="cast">

You're probably wondering what happens if the cast fails. Fear not, we'll get
into that soon.

</aside>

You can start to see how evaluation recursively traverses the tree. We can't
evaluate the unary operator itself until after we evaluate its operand
subexpression. That means our interpreter is doing a **post-order traversal** --
each node evaluates its children before doing its own work.

The other unary operator is logical not.

^code unary-bang (1 before, 1 after)

The implementation is simple, but what is this "truthy" thing about? We need to
make a little side trip to one of the great questions of Western philosophy:
*What is truth?*

### Truthiness and falsiness

OK, maybe we're not going to really get into the universal question, but at
least inside the world of Lox, we need to decide what happens when you use
something other than `true` or `false` in a logic operation like `!` or any
other place where a Boolean is expected.

We *could* just say it's an error because we don't roll with implicit
conversions, but most dynamically typed languages aren't that ascetic. Instead,
they take the universe of values of all types and partition them into two sets,
one of which they define to be "true", or "truthful", or (my favorite) "truthy",
and the rest which are "false" or "falsey". This partitioning is somewhat
arbitrary and gets <span name="weird">weird</span> in a few languages.

<aside name="weird" class="bottom">

In JavaScript, strings are truthy, but empty strings are not. Arrays are truthy
but empty arrays are... also truthy. The number `0` is falsey, but the *string*
`"0"` is truthy.

In Python, empty strings are falsey like in JS, but other empty sequences are
falsey too.

In PHP, both the number `0` and the string `"0"` are falsey. Most other
non-empty strings are truthy.

Get all that?

</aside>

Lox follows Ruby's simple rule: `false` and `nil` are falsey, and everything else
is truthy. We implement that like so:

^code is-truthy

### Evaluating binary operators

On to the last expression tree class, binary operators. There's a handful of
them, and we'll start with the arithmetic ones.

^code visit-binary

<aside name="left">

Did you notice we pinned down a subtle corner of the language semantics here?
In a binary expression, we evaluate the operands in left-to-right order. If
those operands have side effects, that choice is user visible, so this isn't
simply an implementation detail.

If we want our two interpreters to be consistent (hint: we do), we'll need to
make sure clox does the same thing.

</aside>

I think you can figure out what's going on here. The main difference from the
unary negation operator is that we have two operands to evaluate.

I left out one arithmetic operator because it's a little special.

^code binary-plus (3 before, 1 after)

The `+` operator can also be used to concatenate two strings. To handle that, we
don't just assume the operands are a certain type and *cast* them, we
dynamically *check* the type and choose the appropriate operation. This is why
we need our object representation to support `instanceof`.

<aside name="plus">

We could have defined an operator specifically for string concatenation. That's
what Perl (`.`), Lua (`..`), Smalltalk (`,`), Haskell (`++`), and others do.

I thought it would make Lox a little more approachable to use the same syntax as
Java, JavaScript, Python, and others. This means that the `+` operator is
**overloaded** to support both adding numbers and concatenating strings. Even in
languages that don't use `+` for strings, they still often overload it for
adding both integers and floating-point numbers.

</aside>

Next up are the comparison operators.

^code binary-comparison (1 before, 1 after)

They are basically the same as arithmetic. The only difference is that where the
arithmetic operators produce a value whose type is the same as the operands
(numbers or strings), the comparison operators always produce a Boolean.

The last pair of operators are equality.

^code binary-equality

Unlike the comparison operators which require numbers, the equality operators
support operands of any type, even mixed ones. You can't ask Lox if 3 is *less*
than `"three"`, but you can ask if it's <span name="equal">*equal*</span> to
it.

<aside name="equal">

Spoiler alert: it's not.

</aside>

Like truthiness, the equality logic is hoisted out into a separate method.

^code is-equal

This is one of those corners where the details of how we represent Lox objects
in terms of Java matter. We need to correctly implement *Lox's* notion of
equality, which may be different from Java's.

Fortunately, the two are pretty similar. Lox doesn't do implicit conversions in
equality and Java does not either. We do have to handle `nil`/`null` specially
so that we don't throw a NullPointerException if we try to call `equals()` on
`null`. Otherwise, we're fine. Java's <span name="nan">`equals()`</span> method
on Boolean, Double, and String have the behavior we want for Lox.

<aside name="nan">

What do you expect this to evaluate to:

```lox
(0 / 0) == (0 / 0)
```

According to [IEEE 754][], which specifies the behavior of double-precision
numbers, dividing a zero by zero gives you the special **NaN** ("not a number")
value. Strangely enough, NaN is *not* equal to itself.

In Java, the `==` operator on primitive doubles preserves that behavior, but the
`equals()` method on the Double class does not. Lox uses the latter, so doesn't
follow IEEE. These kinds of subtle incompatibilities occupy a dismaying fraction
of language implementers' lives.

[ieee 754]: https://en.wikipedia.org/wiki/IEEE_754

</aside>

And that's it! That's all the code we need to correctly interpret a valid Lox
expression. But what about an *invalid* one? In particular, what happens when a
subexpression evaluates to an object of the wrong type for the operation being
performed?

## Runtime Errors

I was cavalier about jamming casts in whenever a subexpression produces an
Object and the operator requires it to be a number or a string. Those casts can
fail. Even though the user's code is erroneous, if we want to make a <span
name="fail">usable</span> language, we are responsible for handling that error
gracefully.

<aside name="fail">

We could simply not detect or report a type error at all. This is what C does if
you cast a pointer to some type that doesn't match the data that is actually
being pointed to. C gains flexibility and speed by allowing that, but is
also famously dangerous. Once you misinterpret bits in memory, all bets are off.

Few modern languages accept unsafe operations like that. Instead, most are
**memory safe** and ensure -- through a combination of static and runtime checks
-- that a program can never incorrectly interpret the value stored in a piece of
memory.

</aside>

It's time for us to talk about **runtime errors**. I spilled a lot of ink in the
previous chapters talking about error handling, but those were all *syntax* or
*static* errors. Those are detected and reported before *any* code is executed.
Runtime errors are failures that the language semantics demand we detect and
report while the program is running (hence the name).

Right now, if an operand is the wrong type for the operation being performed,
the Java cast will fail and the JVM will throw a ClassCastException. That
unwinds the whole stack and exits the application, vomiting a Java stack trace
onto the user. That's probably not what we want. The fact that Lox is
implemented in Java should be a detail hidden from the user. Instead, we want
them to understand that a *Lox* runtime error occurred, and give them an error
message relevant to our language and their program.

The Java behavior does have one thing going for it, though. It correctly stops
executing any code when the error occurs. Let's say the user enters some
expression like:

```lox
2 * (3 / -"muffin")
```

You can't negate a <span name="muffin">muffin</span>, so we need to report a
runtime error at that inner `-` expression. That in turn means we can't evaluate
the `/` expression since it has no meaningful right operand. Likewise for the
`*`. So when a runtime error occurs deep in some expression, we need to escape
all the way out.

<aside name="muffin">

I don't know, man, *can* you negate a muffin?

<img src="image/evaluating-expressions/muffin.png" alt="A muffin, negated." />

</aside>

We could print a runtime error and then abort the process and exit the
application entirely. That has a certain melodramatic flair. Sort of the
programming language interpreter equivalent of a mic drop.

Tempting as that is, we should probably do something a little less cataclysmic.
While a runtime error needs to stop evaluating the *expression*, it shouldn't
kill the *interpreter*. If a user is running the REPL and has a typo in a line
of code, they should still be able to keep the session going and enter more code
after that.

### Detecting runtime errors

Our tree-walk interpreter evaluates nested expressions using recursive method
calls, and we need to unwind out of all of those. Throwing an exception in Java
is a fine way to accomplish that. However, instead of using Java's own cast
failure, we'll define a Lox-specific one so that we can handle it how we want.

Before we do the cast, we check the object's type ourselves. So, for unary `-`,
we add:

^code check-unary-operand (1 before, 1 after)

The code to check the operand is:

^code check-operand

When the check fails, it throws one of these:

^code runtime-error-class

Unlike the Java cast exception, our <span name="class">class</span> tracks the
token that identifies where in the user's code the runtime error came from. As
with static errors, this helps the user know where to fix their code.

<aside name="class">

I admit the name "RuntimeError" is confusing since Java defines a
RuntimeException class. An annoying thing about building interpreters is your
names often collide with ones already taken by the implementation language. Just
wait until we support Lox classes.

</aside>

We need similar checking for the binary operators. Since I promised you every
single line of code needed to implement the interpreters, I'll run through them
all.

Greater than:

^code check-greater-operand (1 before, 1 after)

Greater than or equal to:

^code check-greater-equal-operand (1 before, 1 after)

Less than:

^code check-less-operand (1 before, 1 after)

Less than or equal to:

^code check-less-equal-operand (1 before, 1 after)

Subtraction:

^code check-minus-operand (1 before, 1 after)

Division:

^code check-slash-operand (1 before, 1 after)

Multiplication:

^code check-star-operand (1 before, 1 after)

All of those rely on this validator, which is virtually the same as the unary
one:

^code check-operands

<aside name="operand">

Another subtle semantic choice: We evaluate *both* operands before checking the
type of *either*. Imagine we have a function `say()` that prints its argument
then returns it. Using that, we write:

```lox
say("left") - say("right");
```

Our interpreter prints "left" and "right" before reporting the runtime error. We
could have instead specified that the left operand is checked before even
evaluating the right.

</aside>

The last remaining operator, again the odd one out, is addition. Since `+` is
overloaded for numbers and strings, it already has code to check the types. All
we need to do is fail if neither of the two success cases match.

^code string-wrong-type (3 before, 1 after)

That gets us detecting runtime errors deep in the innards of the evaluator. The
errors are getting thrown. The next step is to write the code that catches them.
For that, we need to wire up the Interpreter class into the main Lox class that
drives it.

## Hooking Up the Interpreter

The visit methods are sort of the guts of the Interpreter class, where the real
work happens. We need to wrap a skin around them to interface with the rest of
the program. The Interpreter's public API is simply one method.

^code interpret

This takes in a syntax tree for an expression and evaluates it. If that
succeeds, `evaluate()` returns an object for the result value. `interpret()`
converts that to a string and shows it to the user. To convert a Lox value to a
string, we rely on:

^code stringify

This is another of those pieces of code like `isTruthy()` that crosses the
membrane between the user's view of Lox objects and their internal
representation in Java.

It's pretty straightforward. Since Lox was designed to be familiar to someone
coming from Java, things like Booleans look the same in both languages. The two
edge cases are `nil`, which we represent using Java's `null`, and numbers.

Lox uses double-precision numbers even for integer values. In that case, they
should print without a decimal point. Since Java has both floating point and
integer types, it wants you to know which one you're using. It tells you by
adding an explicit `.0` to integer-valued doubles. We don't care about that, so
we <span name="number">hack</span> it off the end.

<aside name="number">

Yet again, we take care of this edge case with numbers to ensure that jlox and
clox work the same. Handling weird corners of the language like this will drive
you crazy but is an important part of the job.

Users rely on these details -- either deliberately or inadvertently -- and if
the implementations aren't consistent, their program will break when they run it
on different interpreters.

</aside>

### Reporting runtime errors

If a runtime error is thrown while evaluating the expression, `interpret()`
catches it. This lets us report the error to the user and then gracefully
continue. All of our existing error reporting code lives in the Lox class, so we
put this method there too:

^code runtime-error-method

We use the token associated with the RuntimeError to tell the user what line of
code was executing when the error occurred. Even better would be to give the
user an entire call stack to show how they *got* to be executing that code. But
we don't have function calls yet, so I guess we don't have to worry about it.

After showing the error, `runtimeError()` sets this field:

^code had-runtime-error-field (1 before, 1 after)

That field plays a small but important role.

^code check-runtime-error (4 before, 1 after)

If the user is running a Lox <span name="repl">script from a file</span> and a
runtime error occurs, we set an exit code when the process quits to let the
calling process know. Not everyone cares about shell etiquette, but we do.

<aside name="repl">

If the user is running the REPL, we don't care about tracking runtime errors.
After they are reported, we simply loop around and let them input new code and
keep going.

</aside>

### Running the interpreter

Now that we have an interpreter, the Lox class can start using it.

^code interpreter-instance (1 before, 1 after)

We make the field static so that successive calls to `run()` inside a REPL
session reuse the same interpreter. That doesn't make a difference now, but it
will later when the interpreter stores global variables. Those variables should
persist throughout the REPL session.

Finally, we remove the line of temporary code from the [last chapter][] for
printing the syntax tree and replace it with this:

[last chapter]: parsing-expressions.html

^code interpreter-interpret (3 before, 1 after)

We have an entire language pipeline now: scanning, parsing, and
execution. Congratulations, you now have your very own arithmetic calculator.

As you can see, the interpreter is pretty bare bones. But the Interpreter class
and the Visitor pattern we've set up today form the skeleton that later chapters
will stuff full of interesting guts -- variables, functions, etc. Right now, the
interpreter doesn't do very much, but it's alive!

<img src="image/evaluating-expressions/skeleton.png" alt="A skeleton waving hello." />

<div class="challenges">

## Challenges

1.  Allowing comparisons on types other than numbers could be useful. The
    operators might have a reasonable interpretation for strings. Even
    comparisons among mixed types, like `3 < "pancake"` could be handy to enable
    things like ordered collections of heterogeneous types. Or it could simply
    lead to bugs and confusion.

    Would you extend Lox to support comparing other types? If so, which pairs of
    types do you allow and how do you define their ordering? Justify your
    choices and compare them to other languages.

2.  Many languages define `+` such that if *either* operand is a string, the
    other is converted to a string and the results are then concatenated. For
    example, `"scone" + 4` would yield `scone4`. Extend the code in
    `visitBinaryExpr()` to support that.

3.  What happens right now if you divide a number by zero? What do you think
    should happen? Justify your choice. How do other languages you know handle
    division by zero, and why do they make the choices they do?

    Change the implementation in `visitBinaryExpr()` to detect and report a
    runtime error for this case.

</div>

<div class="design-note">

## Design Note: Static and Dynamic Typing

Some languages, like Java, are statically typed which means type errors are
detected and reported at compile time before any code is run. Others, like Lox,
are dynamically typed and defer checking for type errors until runtime right
before an operation is attempted. We tend to consider this a black-and-white
choice, but there is actually a continuum between them.

It turns out even most statically typed languages do *some* type checks at
runtime. The type system checks most type rules statically, but inserts runtime
checks in the generated code for other operations.

For example, in Java, the *static* type system assumes a cast expression will
always safely succeed. After you cast some value, you can statically treat it as
the destination type and not get any compile errors. But downcasts can fail,
obviously. The only reason the static checker can presume that casts always
succeed without violating the language's soundness guarantees, is because the
cast is checked *at runtime* and throws an exception on failure.

A more subtle example is [covariant arrays][] in Java and C#. The static
subtyping rules for arrays allow operations that are not sound. Consider:

[covariant arrays]: https://en.wikipedia.org/wiki/Covariance_and_contravariance_(computer_science)#Covariant_arrays_in_Java_and_C.23

```java
Object[] stuff = new Integer[1];
stuff[0] = "not an int!";
```

This code compiles without any errors. The first line upcasts the Integer array
and stores it in a variable of type Object array. The second line stores a
string in one of its cells. The Object array type statically allows that
-- strings *are* Objects -- but the actual Integer array that `stuff` refers to
at runtime should never have a string in it! To avoid that catastrophe, when you
store a value in an array, the JVM does a *runtime* check to make sure it's an
allowed type. If not, it throws an ArrayStoreException.

Java could have avoided the need to check this at runtime by disallowing the
cast on the first line. It could make arrays *invariant* such that an array of
Integers is *not* an array of Objects. That's statically sound, but it prohibits
common and safe patterns of code that only read from arrays. Covariance is safe
if you never *write* to the array. Those patterns were particularly important
for usability in Java 1.0 before it supported generics. James Gosling and the
other Java designers traded off a little static safety and performance -- those
array store checks take time -- in return for some flexibility.

There are few modern statically typed languages that don't make that trade-off
*somewhere*. Even Haskell will let you run code with non-exhaustive matches. If
you find yourself designing a statically typed language, keep in mind that you
can sometimes give users more flexibility without sacrificing *too* many of the
benefits of static safety by deferring some type checks until runtime.

On the other hand, a key reason users choose statically typed languages is
because of the confidence the language gives them that certain kinds of errors
can *never* occur when their program is run. Defer too many type checks until
runtime, and you erode that confidence.

</div>
