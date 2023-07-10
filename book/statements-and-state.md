> 我一生都在渴求一种无法名状的东西。
>
> ——安德烈·布勒东，《疯狂的爱》

我们目前使用的解释器感觉更像是在计算器上按按钮，而不是编写真正的语言。对我来说，“编程”意味着从小的部分构建一个系统。但我们现在无法完成这个过程，因为没有办法将名称绑定到一些数据或函数上。如果没有办法引用这些片段，我们就无法组合成软件系统。

为了支持绑定，我们的解释器需要内部状态。当你在程序的开头定义一个变量并在结尾处使用它时，解释器必须在此期间保持该变量的值。因此，在本章中，我们将为我们的解释器赋予一个可以处理和记忆的大脑。

<img src="image/statements-and-state/brain.png" alt="A brain, presumably remembering stuff." />

状态和 <span name="expr">语句</span> 是相辅相成的。由于语句按定义不会计算出一个具体值，所以它们需要做一些其他的事情来发挥作用。这些事情被称为副作用。它可能意味着产生用户可见的输出或修改解释器中的某些状态，以便稍后可以检测到。后者使它们非常适合定义变量或其他命名实体。

<aside name="expr">

你可以创建一种语言，将变量声明视为既创建绑定又产生值的表达式。Tcl是其中唯一我知道的一个这样做的语言。虽然Scheme也有类似的语法，但需要注意的是，在  `let` 表达式被求值之后，它所绑定的变量就被遗忘了。而 `define` 语法则不是一个表达式。

</aside>

在本章中，我们将完成这些任务。我们将定义产生输出（`print`）和创建状态（`var`）的语句，并添加表达式来访问和赋值变量。最后，我们会引入代码块和局部作用域。虽然内容很多，但我们会逐步学习，一步一步地消化它们。

## 语句

我们首先扩展Lox的语法以支持语句。 语句与表达式并没有很大的不同，我们从两种最简单的类型开始：
    
1.   **表达式语句 ** 可以让您将表达式放在需要语句的位置。它们的存在是为了计算有副作用的表达式。您可能没有注意到它们，但其实你在<span   name="expr-stmt">C</span>、Java和其他语言中一直在使用表达式语句。如果你看到一个函数或方法调用后面跟着一个 `;`，您看到的其实就是一个表达式语句。

    <aside name="expr-stmt">

    Pascal是一个例外。它区分*过程*和*函数*。函数返回值，但过程没有返回值。有一种调用过程的语句形式，但只能在需要表达式的地方调用函数。Pascal中没有表达式语句。

    </aside>

2.  `print` 语句会计算表达式，并将结果展示给用户。我承认,将 `print` 直接嵌入语言中，而不是把它变成一个库函数的做法有些奇怪。这样做是基于本书的编排策略的让步，即我们会以章节为单位逐步构建这个解释器，并希望能够在完成解释器的所有功能<span name="print">之前</span>能够使用它。如果让print成为一个标准库函数，我们必须等到拥有了定义和调用函数的所有机制之后，才能看到它发挥作用。

    <aside name="print">
    
    这里提一下(原文中的防杠提示...)，BASIC和Python都有专门的 `print` 语句，并且它们都是真正的编程语言。当然，在Python 3.0中，他们删除了 `print` 语句……

    </aside>

新的语法意味着新的语法规则。在本章中，我们终于可以解析整个Lox脚本。由于Lox是一种命令式、动态类型的语言，脚本的“顶层”只是一个语句列表。新的规则如下：

```ebnf
program        → statement* EOF ;

statement      → exprStmt
               | printStmt ;

exprStmt       → expression ";" ;
printStmt      → "print" expression ";" ;
```

第一条规则是 `program`，它是语法的起点，代表一个完整的Lox脚本或REPL输入。一个程序是一系列语句，后面跟着特殊的“文件结束”(EOF)标记。这个结束标记是必须的，它确保解析器消费整个输入，而不会在脚本末尾忽略错误的未消耗的标记。

目前，`statement` 只有两种情况，分别对应我们描述过的两种语句。在本章的后面和接下来的章节中，我们将填充更多的内容。下一步是将这个语法转换成我们可以存储在内存中的东西——语法树。

### Statement语法树

在语法中，没有同时允许表达式和语句的地方。例如，操作符，例如 `+`  的操作数始终是表达式，而不是语句。`while` 循环的主体始终是一个语句。 

因为这两种语法是不相干的，所以我们不需要提供一个它们都继承的基类。将表达式和语句拆分为单独的类结构，可使Java编译器帮助我们发现一些愚蠢的错误，例如将语句传递给需要表达式的Java方法。 

这意味着要为语句创建一个新的基类。我们将像我们的前辈一样使用神秘的名称“Stmt”。我很有<span name="foresight">远见</span>，在设计我们的AST元编程脚本时就已经预见到了这一点。这就是我们将“Expr”作为参数传给了 `defineAst()` 的原因。现在我们添加另一个方法调用来定义Stmt和它的<span name="stmt-ast">子类</span>。

<aside name="foresight">

不完全是因为远见：我在将代码分割成章节之前就已经编写了整本书的所有代码。

</aside>

^code stmt-ast (2 before, 1 after)

<aside name="stmt-ast">

新节点的生成代码在 [附录 II][appendix-ii]中： [Expression语句][Expression statement], [Print 语句][Print statement]。

[appendix-ii]: appendix-ii.html
[expression statement]: appendix-ii.html#expression-statement
[print statement]: appendix-ii.html#print-statement

</aside>

运行AST生成器脚本，然后查看生成的 `Stmt.java` 文件，其中包含我们需要的表达式和 `print` 语句的语法树类。不要忘记将该文件添加到您的IDE项目或makefile或其他相关文件中。

###  解析语句

解析器的parse()方法会解析并返回一个表达式，这只是一个临时的hack，用于让上一章的代码能启动并运行起来。现在，我们的语法已经有了正确的起始规则，`program`，我们可以正式编写 `parse()` 方法了。

^code parse

<aside name="parse-error-handling">

我们之前在这里添加的捕获 `ParseError` 异常的代码怎么办？当我们添加对其他语句类型的支持时，我们很快就会放置更好的解析错误处理代码。

</aside>

该方法会尽可能多地解析一系列语句，直到命中输入内容的结尾为止。这是一种非常直接的将program规则转换为递归下降风格的方式。由于我们现在使用ArrayList，所以我们还必须向Java的冗长之神祈祷一番。

^code parser-imports (2 before, 1 after)

一个程序就是一系列的语句，而我们可以通过下面的方法解析每一条语句：

^code parse-statement

看起来有些简陋，不过我们稍后会填充更多的语句类型。我们通过查看当前标记来确定匹配的具体语句规则。例如 `print`标记，显然对应 `print` 语句。

如果下一个标记看起来不像任何已知类型的语句，我们就假设它一定是一个表达式语句。这是解析语句时典型的最终失败分支，因为很难从第一个标记主动识别出一个表达式。

每种语句类型都有自己的方法。首先是 `print`：

^code parse-print-statement

由于我们已经匹配和消耗了 `print` 标记本身，所以这里不需要再次进行匹配。我们解析后续的表达式，消耗结束的分号，并生成语法树。

如果我们没有匹配到print语句，那一定是一条表达式语句：

^code parse-expression-statement

与前面的方法类似，我们解析一个后面带分号的表达式。我们将该表达式 Expr 包装在正确类型的 Stmt 语句中并返回。

### 执行语句

我们正在以微观的方式回顾前面的几章，逐步完成解释器的前端工作。我们的解析器现在可以生成语句的语法树，所以下一步也是最后一步就是解释它们。与表达式一样，我们使用访问者模式，但是我们需要实现一个新的访问者接口 Stmt.Visitor 需要实现，因为语句有自己的基类。

我们将其添加到 Interpreter 实现的接口列表中。

^code interpreter (1 after)

<aside name="void">

出于与类型擦除和堆栈有关的晦涩原因，Java不允许使用小写的 "void" 作为泛型类型参数。相反，有一个专门用于此用途的单独的 "Void" 类型。可以将其视为 "void" 的 "包装类"，类似于 "Integer" 是 "int" 的包装类。

</aside>

与表达式不同，语句不产生任何值，因此 visit 方法的返回类型是 Void，而不是 Object。我们有两种语句类型，每种类型都需要一个 visit 方法。最简单的是表达式语句。

^code visit-expression-stmt

我们使用现有的 `evaluate()` 方法计算内部表达式，并<span name="discard">丢弃</span>其结果值。然后我们返回`null`，因为Java要求为特殊的大写Void返回类型返回该值。虽然有点奇怪，但你能做什么呢？


<aside name="discard">

刚巧，我们通过将 `evaluate()` 的调用放在一个 Java 表达式语句中来丢弃返回的值。

</aside>

`print` 语句的visit方法没有太大的区别。

^code visit-print

在丢弃表达式的值之前，我们使用上一章介绍的 `stringify()` 方法将其转换为字符串，然后将其输出到 stdout。

我们的解释器现在能够处理语句了，但是我们还需要做一些工作来将它们传递给解释器。首先，修改 Interpreter 类中的旧 `interpret()` 方法，以接受语句列表，即一段程序。

^code interpret

这将替换掉原来处理单个表达式的旧代码。新代码依赖于下面的辅助小方法：

^code execute

这里处理语句的方法类似处理表达式的 `evaluate()` 方法。因为要使用列表，所以我们需要在Java中引入一下。

^code import-list (2 before, 2 after)

Lox主类中仍然是只解析单个表达式并将其传给解释器。我们将其修正如下：

^code parse-statements (1 before, 2 after)

然后将对解释器的调用替换为以下内容：

^code interpret-statements (2 before, 1 after)

基本就是对新语法的遍历。好了，启动解释器并尝试一下。此时，有必要在文本文件中草拟一个小的Lox程序作为脚本运行。类似于：

```lox
print "one";
print true;
print 2 + 1;
```

它几乎看起来像一个真正的程序！请注意，REPL现在也要求您输入完整的语句，而不仅仅是简单的表达式。不要忘记分号。

## 全局变量

现在我们有了语句，我们可以开始处理状态。在我们深入研究词法作用域的所有复杂性之前，我们将从最简单的变量--<span name="globals">全局变量</span>开始。我们需要两个新的结构。

1.  **变量声明** 语句用于创建一个新变量。


    ```lox
    var beverage = "espresso";
    ```

    该语句将创建一个新的绑定，将一个名称（这里是 "beverage"）和一个值（这里是字符串 `"espresso"`）关联起来。

2.  一旦声明完成，**变量表达式** 就可以访问该绑定。当标识符“beverage”被用作一个表达式时，程序会查找与该名称绑定的值并返回。


    ```lox
    print beverage; // "espresso".
    ```

稍后，我们将添加赋值和块作用域，但现在这已经足够让我们继续前进了。

<aside name="globals">

全局状态的名声不好。确实，大量的全局状态，特别是可变状态，使得大型程序难以维护。在软件工程中，应该尽量减少使用全局状态。

但是，当你在拼凑一个简单的编程语言，甚至是学习你的第一门语言时，全局变量的简单性是有帮助的。我的第一门语言是BASIC，尽管我最终不再使用它了，但是在我能让计算机做有趣的事情之前，不必理解作用域规则，挺好。

</aside>

### 变量语法

与前面一样，我们将从语法开始，从前到后依次完成实现。变量声明是一种语句，但它们不同于其他语句，我们把statement语法一分为二来处理该情况。这是因为语法限制了某些类型的语句可以出现的位置。

控制流语句中的子句（比如if语句的then和else分支或while循环的循环体）都是单个语句。但是，这个语句不能是一个声明变量的语句。下面的代码是OK的：


```lox
if (monday) print "Ugh, already?";
```

但是这个不行：

```lox
if (monday) var beverage = "espresso";
```

我们也*可以*允许后者，但这会令人困惑。那个 `beverage` 变量的作用域是什么？在 `if` 语句之后是否仍然存在？如果是的话，在除了星期一以外的其他日子，它的值是什么？在那些日子里，变量是否存在？

像这样的代码很奇怪，所以C、Java等语言都不允许这种写法。就像存在两个级别的<span name="brace">“优先级”</span>一样，语句的一些位置（比如在块内或顶层）允许任何类型的语句，包括声明语句。而其他位置只允许“更高级别”的语句，不允许声明语句。

<aside name="brace">

代码块语句的工作方式类似于表达式中的括号。“块”本身处于“较高”的优先级，并且可以在任何地方使用，如if语句的子语句中。而其中*包含*的可以是优先级较低的语句。你可以在块中声明变量或其它名称。通过大括号，你可以在只允许某些语句的位置书写完整的语句语法。

</aside>

为了适应这种区别，我们为声明语句添加了另一条规则。

```ebnf
program        → declaration* EOF ;

declaration    → varDecl
               | statement ;

statement      → exprStmt
               | printStmt ;
```

声明语句属于新的 `declaration` 规则。目前，它只包括变量，但以后还会包括函数和类。任何允许声明的地方也允许非声明语句，所以 `declaration` 规则会转到 `statement` 规则。显然，你可以在脚本的顶层声明变量，所以 `program` 会遵循新的规则。

声明一个变量的规则如下：

```ebnf
varDecl        → "var" IDENTIFIER ( "=" expression )? ";" ;
```

和大多数语句一样，声明语句以一个关键字开头。这里是`var`关键字。然后是一个标识符令牌，表示正在声明的变量的名称，后面是可选的初始化表达式。最后，我们用分号结束它。

为了访问一个变量，我们定义了一种新的基本表达式类型：

```ebnf
primary        → "true" | "false" | "nil"
               | NUMBER | STRING
               | "(" expression ")"
               | IDENTIFIER ;
```

`IDENTIFIER` 子句匹配一个单独的标识符令牌，该令牌被理解为要访问的变量的名称。

These new grammar rules get their corresponding syntax trees. Over in the AST
generator, we add a <span name="var-stmt-ast">new statement</span> node for a
variable declaration.
这些新的语法规则需要其相应的语法树。在AST生成器中，我们为变量声明添加一个<span name="var-stmt-ast">新的</span>语句树。

^code var-stmt-ast (1 before, 1 after)

<aside name="var-stmt-ast">

生成的新节点的代码位于[Appendix II][appendix-var-stmt]中。

[appendix-var-stmt]: appendix-ii.html#variable-statement

</aside>

这里存储了名称标记，以便我们知道该语句声明了什么，此外还有初始化表达式（如果没有，字段就是null）。

然后我们添加一个表达式节点用于访问变量。

^code var-expr (1 before, 1 after)

<span name="var-expr-ast">这</span>只是对变量名称标记的简单包装，就是这样。一如既往，别忘了运行AST生成器脚本，以便获取更新的 "Expr.java "和 "Stmt.java "文件。

<aside name="var-expr-ast">

生成的新节点的代码位于[Appendix II][appendix-var-expr]中。

[appendix-var-expr]: appendix-ii.html#variable-expression

</aside>

### 解析变量


在解析变量语句之前，我们需要修改一些代码，为语法中的新规则`declaration`腾出一些空间。现在，程序的最顶层是声明语句的列表，所以解析器方法的入口需要更改

^code parse-declaration (3 before, 4 after)

这里会调用下面的新方法：

^code declaration

嘿，你还记得在 [前面的章节][parsing] 中我们建立了[错误恢复][error recovery]的手脚架吗？现在我们终于准备好跟它串起来了。

[parsing]: parsing-expressions.html
[error recovery]: parsing-expressions.html#panic-mode-error-recovery

`declaration()`  方法在解析块或脚本中的一系列语句时会被重复调用，因此当解析器进入恐慌模式时，它就是进行同步的正确位置。该方法的整个体被包裹在一个try块中，以捕获解析器开始错误恢复时抛出的异常。这使得解析器可以尝试解析下一条语句或声明的开头。

真正的解析发生在try块内部。首先，它通过查找前导的`var` 关键字来判断是否为变量声明。如果不是，它将继续执行现有的`statement()`方法，该方法解析`print`和语句表达式。

Remember how `statement()` tries to parse an expression statement if no other
statement matches? And `expression()` reports a syntax error if it can't parse
an expression at the current token? That chain of calls ensures we report an
error if a valid declaration or statement isn't parsed.

When the parser matches a `var` token, it branches to:

^code parse-var-declaration

As always, the recursive descent code follows the grammar rule. The parser has
already matched the `var` token, so next it requires and consumes an identifier
token for the variable name.

Then, if it sees an `=` token, it knows there is an initializer expression and
parses it. Otherwise, it leaves the initializer `null`. Finally, it consumes the
required semicolon at the end of the statement. All this gets wrapped in a
Stmt.Var syntax tree node and we're groovy.

Parsing a variable expression is even easier. In `primary()`, we look for an
identifier token.

^code parse-identifier (2 before, 2 after)

That gives us a working front end for declaring and using variables. All that's
left is to feed it into the interpreter. Before we get to that, we need to talk
about where variables live in memory.

## Environments

The bindings that associate variables to values need to be stored somewhere.
Ever since the Lisp folks invented parentheses, this data structure has been
called an <span name="env">**environment**</span>.

<img src="image/statements-and-state/environment.png" alt="An environment containing two bindings." />

<aside name="env">

I like to imagine the environment literally, as a sylvan wonderland where
variables and values frolic.

</aside>

You can think of it like a <span name="map">map</span> where the keys are
variable names and the values are the variable's, uh, values. In fact, that's
how we'll implement it in Java. We could stuff that map and the code to manage
it right into Interpreter, but since it forms a nicely delineated concept, we'll
pull it out into its own class.

Start a new file and add:

<aside name="map">

Java calls them **maps** or **hashmaps**. Other languages call them **hash
tables**, **dictionaries** (Python and C#), **hashes** (Ruby and Perl),
**tables** (Lua), or **associative arrays** (PHP). Way back when, they were
known as **scatter tables**.

</aside>

^code environment-class

There's a Java Map in there to store the bindings. It uses bare strings for the
keys, not tokens. A token represents a unit of code at a specific place in the
source text, but when it comes to looking up variables, all identifier tokens
with the same name should refer to the same variable (ignoring scope for now).
Using the raw string ensures all of those tokens refer to the same map key.

There are two operations we need to support. First, a variable definition binds
a new name to a value.

^code environment-define

Not exactly brain surgery, but we have made one interesting semantic choice.
When we add the key to the map, we don't check to see if it's already present.
That means that this program works:

```lox
var a = "before";
print a; // "before".
var a = "after";
print a; // "after".
```

A variable statement doesn't just define a *new* variable, it can also be used
to *re*define an existing variable. We could <span name="scheme">choose</span>
to make this an error instead. The user may not intend to redefine an existing
variable. (If they did mean to, they probably would have used assignment, not
`var`.) Making redefinition an error would help them find that bug.

However, doing so interacts poorly with the REPL. In the middle of a REPL
session, it's nice to not have to mentally track which variables you've already
defined. We could allow redefinition in the REPL but not in scripts, but then
users would have to learn two sets of rules, and code copied and pasted from one
form to the other might not work.

<aside name="scheme">

My rule about variables and scoping is, "When in doubt, do what Scheme does".
The Scheme folks have probably spent more time thinking about variable scope
than we ever will -- one of the main goals of Scheme was to introduce lexical
scoping to the world -- so it's hard to go wrong if you follow in their
footsteps.

Scheme allows redefining variables at the top level.

</aside>

So, to keep the two modes consistent, we'll allow it -- at least for global
variables. Once a variable exists, we need a way to look it up.

^code environment-get (2 before, 1 after)

This is a little more semantically interesting. If the variable is found, it
simply returns the value bound to it. But what if it's not? Again, we have a
choice:

* Make it a syntax error.

* Make it a runtime error.

* Allow it and return some default value like `nil`.

Lox is pretty lax, but the last option is a little *too* permissive to me.
Making it a syntax error -- a compile-time error -- seems like a smart choice.
Using an undefined variable is a bug, and the sooner you detect the mistake, the
better.

The problem is that *using* a variable isn't the same as *referring* to it. You
can refer to a variable in a chunk of code without immediately evaluating it if
that chunk of code is wrapped inside a function. If we make it a static error to
*mention* a variable before it's been declared, it becomes much harder to define
recursive functions.

We could accommodate single recursion -- a function that calls itself -- by
declaring the function's own name before we examine its body. But that doesn't
help with mutually recursive procedures that call each other. Consider:

<span name="contrived"></span>

```lox
fun isOdd(n) {
  if (n == 0) return false;
  return isEven(n - 1);
}

fun isEven(n) {
  if (n == 0) return true;
  return isOdd(n - 1);
}
```

<aside name="contrived">

Granted, this is probably not the most efficient way to tell if a number is even
or odd (not to mention the bad things that happen if you pass a non-integer or
negative number to them). Bear with me.

</aside>

The `isEven()` function isn't defined by the <span name="declare">time</span> we
are looking at the body of `isOdd()` where it's called. If we swap the order of
the two functions, then `isOdd()` isn't defined when we're looking at
`isEven()`'s body.

<aside name="declare">

Some statically typed languages like Java and C# solve this by specifying that
the top level of a program isn't a sequence of imperative statements. Instead, a
program is a set of declarations which all come into being simultaneously. The
implementation declares *all* of the names before looking at the bodies of *any*
of the functions.

Older languages like C and Pascal don't work like this. Instead, they force you
to add explicit *forward declarations* to declare a name before it's fully
defined. That was a concession to the limited computing power at the time. They
wanted to be able to compile a source file in one single pass through the text,
so those compilers couldn't gather up all of the declarations first before
processing function bodies.

</aside>

Since making it a *static* error makes recursive declarations too difficult,
we'll defer the error to runtime. It's OK to refer to a variable before it's
defined as long as you don't *evaluate* the reference. That lets the program
for even and odd numbers work, but you'd get a runtime error in:

```lox
print a;
var a = "too late!";
```

As with type errors in the expression evaluation code, we report a runtime error
by throwing an exception. The exception contains the variable's token so we can
tell the user where in their code they messed up.

### Interpreting global variables

The Interpreter class gets an instance of the new Environment class.

^code environment-field (2 before, 1 after)

We store it as a field directly in Interpreter so that the variables stay in
memory as long as the interpreter is still running.

We have two new syntax trees, so that's two new visit methods. The first is for
declaration statements.

^code visit-var

If the variable has an initializer, we evaluate it. If not, we have another
choice to make. We could have made this a syntax error in the parser by
*requiring* an initializer. Most languages don't, though, so it feels a little
harsh to do so in Lox.

We could make it a runtime error. We'd let you define an uninitialized variable,
but if you accessed it before assigning to it, a runtime error would occur. It's
not a bad idea, but most dynamically typed languages don't do that. Instead,
we'll keep it simple and say that Lox sets a variable to `nil` if it isn't
explicitly initialized.

```lox
var a;
print a; // "nil".
```

Thus, if there isn't an initializer, we set the value to `null`, which is the
Java representation of Lox's `nil` value. Then we tell the environment to bind
the variable to that value.

Next, we evaluate a variable expression.

^code visit-variable

This simply forwards to the environment which does the heavy lifting to make
sure the variable is defined. With that, we've got rudimentary variables
working. Try this out:

```lox
var a = 1;
var b = 2;
print a + b;
```

We can't reuse *code* yet, but we can start to build up programs that reuse
*data*.

## Assignment

It's possible to create a language that has variables but does not let you
reassign -- or **mutate** -- them. Haskell is one example. SML supports only
mutable references and arrays -- variables cannot be reassigned. Rust steers you
away from mutation by requiring a `mut` modifier to enable assignment.

Mutating a variable is a side effect and, as the name suggests, some language
folks think side effects are <span name="pure">dirty</span> or inelegant. Code
should be pure math that produces values -- crystalline, unchanging ones -- like
an act of divine creation. Not some grubby automaton that beats blobs of data
into shape, one imperative grunt at a time.

<aside name="pure">

I find it delightful that the same group of people who pride themselves on
dispassionate logic are also the ones who can't resist emotionally loaded terms
for their work: "pure", "side effect", "lazy", "persistent", "first-class",
"higher-order".

</aside>

Lox is not so austere. Lox is an imperative language, and mutation comes with
the territory. Adding support for assignment doesn't require much work. Global
variables already support redefinition, so most of the machinery is there now.
Mainly, we're missing an explicit assignment notation.

### Assignment syntax

That little `=` syntax is more complex than it might seem. Like most C-derived
languages, assignment is an <span name="assign">expression</span> and not a
statement. As in C, it is the lowest precedence expression form. That means the
rule slots between `expression` and `equality` (the next lowest precedence
expression).

<aside name="assign">

In some other languages, like Pascal, Python, and Go, assignment is a statement.

</aside>

```ebnf
expression     → assignment ;
assignment     → IDENTIFIER "=" assignment
               | equality ;
```

This says an `assignment` is either an identifier followed by an `=` and an
expression for the value, or an `equality` (and thus any other) expression.
Later, `assignment` will get more complex when we add property setters on
objects, like:

```lox
instance.field = "value";
```

The easy part is adding the <span name="assign-ast">new syntax tree node</span>.

^code assign-expr (1 before, 1 after)

<aside name="assign-ast">

The generated code for the new node is in [Appendix II][appendix-assign].

[appendix-assign]: appendix-ii.html#assign-expression

</aside>

It has a token for the variable being assigned to, and an expression for the new
value. After you run the AstGenerator to get the new Expr.Assign class, swap out
the body of the parser's existing `expression()` method to match the updated
rule.

^code expression (1 before, 1 after)

Here is where it gets tricky. A single token lookahead recursive descent parser
can't see far enough to tell that it's parsing an assignment until *after* it
has gone through the left-hand side and stumbled onto the `=`. You might wonder
why it even needs to. After all, we don't know we're parsing a `+` expression
until after we've finished parsing the left operand.

The difference is that the left-hand side of an assignment isn't an expression
that evaluates to a value. It's a sort of pseudo-expression that evaluates to a
"thing" you can assign to. Consider:

```lox
var a = "before";
a = "value";
```

On the second line, we don't *evaluate* `a` (which would return the string
"before"). We figure out what variable `a` refers to so we know where to store
the right-hand side expression's value. The [classic terms][l-value] for these
two <span name="l-value">constructs</span> are **l-value** and **r-value**. All
of the expressions that we've seen so far that produce values are r-values. An
l-value "evaluates" to a storage location that you can assign into.

[l-value]: https://en.wikipedia.org/wiki/Value_(computer_science)#lrvalue

<aside name="l-value">

In fact, the names come from assignment expressions: *l*-values appear on the
*left* side of the `=` in an assignment, and *r*-values on the *right*.

</aside>

We want the syntax tree to reflect that an l-value isn't evaluated like a normal
expression. That's why the Expr.Assign node has a *Token* for the left-hand
side, not an Expr. The problem is that the parser doesn't know it's parsing an
l-value until it hits the `=`. In a complex l-value, that may occur <span
name="many">many</span> tokens later.

```lox
makeList().head.next = node;
```

<aside name="many">

Since the receiver of a field assignment can be any expression, and expressions
can be as long as you want to make them, it may take an *unbounded* number of
tokens of lookahead to find the `=`.

</aside>

We have only a single token of lookahead, so what do we do? We use a little
trick, and it looks like this:

^code parse-assignment

Most of the code for parsing an assignment expression looks similar to that of
the other binary operators like `+`. We parse the left-hand side, which can be
any expression of higher precedence. If we find an `=`, we parse the right-hand
side and then wrap it all up in an assignment expression tree node.

<aside name="no-throw">

We *report* an error if the left-hand side isn't a valid assignment target, but
we don't *throw* it because the parser isn't in a confused state where we need
to go into panic mode and synchronize.

</aside>

One slight difference from binary operators is that we don't loop to build up a
sequence of the same operator. Since assignment is right-associative, we instead
recursively call `assignment()` to parse the right-hand side.

The trick is that right before we create the assignment expression node, we look
at the left-hand side expression and figure out what kind of assignment target
it is. We convert the r-value expression node into an l-value representation.

This conversion works because it turns out that every valid assignment target
happens to also be <span name="converse">valid syntax</span> as a normal
expression. Consider a complex field assignment like:

<aside name="converse">

You can still use this trick even if there are assignment targets that are not
valid expressions. Define a **cover grammar**, a looser grammar that accepts
all of the valid expression *and* assignment target syntaxes. When you hit
an `=`, report an error if the left-hand side isn't within the valid assignment
target grammar. Conversely, if you *don't* hit an `=`, report an error if the
left-hand side isn't a valid *expression*.

</aside>

```lox
newPoint(x + 2, 0).y = 3;
```

The left-hand side of that assignment could also work as a valid expression.

```lox
newPoint(x + 2, 0).y;
```

The first example sets the field, the second gets it.

This means we can parse the left-hand side *as if it were* an expression and
then after the fact produce a syntax tree that turns it into an assignment
target. If the left-hand side expression isn't a <span name="paren">valid</span>
assignment target, we fail with a syntax error. That ensures we report an error
on code like this:

```lox
a + b = c;
```

<aside name="paren">

Way back in the parsing chapter, I said we represent parenthesized expressions
in the syntax tree because we'll need them later. This is why. We need to be
able to distinguish these cases:

```lox
a = 3;   // OK.
(a) = 3; // Error.
```

</aside>

Right now, the only valid target is a simple variable expression, but we'll add
fields later. The end result of this trick is an assignment expression tree node
that knows what it is assigning to and has an expression subtree for the value
being assigned. All with only a single token of lookahead and no backtracking.

### Assignment semantics

We have a new syntax tree node, so our interpreter gets a new visit method.

^code visit-assign

For obvious reasons, it's similar to variable declaration. It evaluates the
right-hand side to get the value, then stores it in the named variable. Instead
of using `define()` on Environment, it calls this new method:

^code environment-assign

The key difference between assignment and definition is that assignment is not
<span name="new">allowed</span> to create a *new* variable. In terms of our
implementation, that means it's a runtime error if the key doesn't already exist
in the environment's variable map.

<aside name="new">

Unlike Python and Ruby, Lox doesn't do [implicit variable declaration][].

[implicit variable declaration]: #design-note

</aside>

The last thing the `visit()` method does is return the assigned value. That's
because assignment is an expression that can be nested inside other expressions,
like so:

```lox
var a = 1;
print a = 2; // "2".
```

Our interpreter can now create, read, and modify variables. It's about as
sophisticated as early <span name="basic">BASICs</span>. Global variables are
simple, but writing a large program when any two chunks of code can accidentally
step on each other's state is no fun. We want *local* variables, which means
it's time for *scope*.

<aside name="basic">

Maybe a little better than that. Unlike some old BASICs, Lox can handle variable
names longer than two characters.

</aside>

## Scope

A **scope** defines a region where a name maps to a certain entity. Multiple
scopes enable the same name to refer to different things in different contexts.
In my house, "Bob" usually refers to me. But maybe in your town you know a
different Bob. Same name, but different dudes based on where you say it.

<span name="lexical">**Lexical scope**</span> (or the less commonly heard
**static scope**) is a specific style of scoping where the text of the program
itself shows where a scope begins and ends. In Lox, as in most modern languages,
variables are lexically scoped. When you see an expression that uses some
variable, you can figure out which variable declaration it refers to just by
statically reading the code.

<aside name="lexical">

"Lexical" comes from the Greek "lexikos" which means "related to words". When we
use it in programming languages, it usually means a thing you can figure out
from source code itself without having to execute anything.

Lexical scope came onto the scene with ALGOL. Earlier languages were often
dynamically scoped. Computer scientists back then believed dynamic scope was
faster to execute. Today, thanks to early Scheme hackers, we know that isn't
true. If anything, it's the opposite.

Dynamic scope for variables lives on in some corners. Emacs Lisp defaults to
dynamic scope for variables. The [`binding`][binding] macro in Clojure provides
it. The widely disliked [`with` statement][with] in JavaScript turns properties
on an object into dynamically scoped variables.

[binding]: http://clojuredocs.org/clojure.core/binding
[with]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/with

</aside>

For example:

```lox
{
  var a = "first";
  print a; // "first".
}

{
  var a = "second";
  print a; // "second".
}
```

Here, we have two blocks with a variable `a` declared in each of them. You and
I can tell just from looking at the code that the use of `a` in the first
`print` statement refers to the first `a`, and the second one refers to the
second.

<img src="image/statements-and-state/blocks.png" alt="An environment for each 'a'." />

This is in contrast to **dynamic scope** where you don't know what a name refers
to until you execute the code. Lox doesn't have dynamically scoped *variables*,
but methods and fields on objects are dynamically scoped.

```lox
class Saxophone {
  play() {
    print "Careless Whisper";
  }
}

class GolfClub {
  play() {
    print "Fore!";
  }
}

fun playIt(thing) {
  thing.play();
}
```

When `playIt()` calls `thing.play()`, we don't know if we're about to hear
"Careless Whisper" or "Fore!" It depends on whether you pass a Saxophone or a
GolfClub to the function, and we don't know that until runtime.

Scope and environments are close cousins. The former is the theoretical concept,
and the latter is the machinery that implements it. As our interpreter works its
way through code, syntax tree nodes that affect scope will change the
environment. In a C-ish syntax like Lox's, scope is controlled by curly-braced
blocks. (That's why we call it **block scope**.)

```lox
{
  var a = "in block";
}
print a; // Error! No more "a".
```

The beginning of a block introduces a new local scope, and that scope ends when
execution passes the closing `}`. Any variables declared inside the block
disappear.

### Nesting and shadowing

A first cut at implementing block scope might work like this:

1.  As we visit each statement inside the block, keep track of any variables
    declared.

2.  After the last statement is executed, tell the environment to delete all of
    those variables.

That would work for the previous example. But remember, one motivation for
local scope is encapsulation -- a block of code in one corner of the program
shouldn't interfere with some other block. Check this out:

```lox
// How loud?
var volume = 11;

// Silence.
volume = 0;

// Calculate size of 3x4x5 cuboid.
{
  var volume = 3 * 4 * 5;
  print volume;
}
```

Look at the block where we calculate the volume of the cuboid using a local
declaration of `volume`. After the block exits, the interpreter will delete the
*global* `volume` variable. That ain't right. When we exit the block, we should
remove any variables declared inside the block, but if there is a variable with
the same name declared outside of the block, *that's a different variable*. It
shouldn't get touched.

When a local variable has the same name as a variable in an enclosing scope, it
**shadows** the outer one. Code inside the block can't see it any more -- it is
hidden in the "shadow" cast by the inner one -- but it's still there.

When we enter a new block scope, we need to preserve variables defined in outer
scopes so they are still around when we exit the inner block. We do that by
defining a fresh environment for each block containing only the variables
defined in that scope. When we exit the block, we discard its environment and
restore the previous one.

We also need to handle enclosing variables that are *not* shadowed.

```lox
var global = "outside";
{
  var local = "inside";
  print global + local;
}
```

Here, `global` lives in the outer global environment and `local` is defined
inside the block's environment. In that `print` statement, both of those
variables are in scope. In order to find them, the interpreter must search not
only the current innermost environment, but also any enclosing ones.

We implement this by <span name="cactus">chaining</span> the environments
together. Each environment has a reference to the environment of the immediately
enclosing scope. When we look up a variable, we walk that chain from innermost
out until we find the variable. Starting at the inner scope is how we make local
variables shadow outer ones.

<img src="image/statements-and-state/chaining.png" alt="Environments for each scope, linked together." />

<aside name="cactus">

While the interpreter is running, the environments form a linear list of
objects, but consider the full set of environments created during the entire
execution. An outer scope may have multiple blocks nested within it, and each
will point to the outer one, giving a tree-like structure, though only one path
through the tree exists at a time.

The boring name for this is a [**parent-pointer tree**][parent pointer], but I
much prefer the evocative **cactus stack**.

[parent pointer]: https://en.wikipedia.org/wiki/Parent_pointer_tree

<img class="above" src="image/statements-and-state/cactus.png" alt="Each branch points to its parent. The root is global scope." />

</aside>

Before we add block syntax to the grammar, we'll beef up our Environment class
with support for this nesting. First, we give each environment a reference to
its enclosing one.

^code enclosing-field (1 before, 1 after)

This field needs to be initialized, so we add a couple of constructors.

^code environment-constructors

The no-argument constructor is for the global scope's environment, which ends
the chain. The other constructor creates a new local scope nested inside the
given outer one.

We don't have to touch the `define()` method -- a new variable is always
declared in the current innermost scope. But variable lookup and assignment work
with existing variables and they need to walk the chain to find them. First,
lookup:

^code environment-get-enclosing (2 before, 3 after)

If the variable isn't found in this environment, we simply try the enclosing
one. That in turn does the same thing <span name="recurse">recursively</span>,
so this will ultimately walk the entire chain. If we reach an environment with
no enclosing one and still don't find the variable, then we give up and report
an error as before.

Assignment works the same way.

<aside name="recurse">

It's likely faster to iteratively walk the chain, but I think the recursive
solution is prettier. We'll do something *much* faster in clox.

</aside>

^code environment-assign-enclosing (4 before, 1 after)

Again, if the variable isn't in this environment, it checks the outer one,
recursively.

### Block syntax and semantics

Now that Environments nest, we're ready to add blocks to the language. Behold
the grammar:

```ebnf
statement      → exprStmt
               | printStmt
               | block ;

block          → "{" declaration* "}" ;
```

A block is a (possibly empty) series of statements or declarations surrounded by
curly braces. A block is itself a statement and can appear anywhere a statement
is allowed. The <span name="block-ast">syntax tree</span> node looks like this:

^code block-ast (1 before, 1 after)

<aside name="block-ast">

The generated code for the new node is in [Appendix II][appendix-block].

[appendix-block]: appendix-ii.html#block-statement

</aside>

<span name="generate">It</span> contains the list of statements that are inside
the block. Parsing is straightforward. Like other statements, we detect the
beginning of a block by its leading token -- in this case the `{`. In the
`statement()` method, we add:

<aside name="generate">

As always, don't forget to run "GenerateAst.java".

</aside>

^code parse-block (1 before, 2 after)

All the real work happens here:

^code block

We <span name="list">create</span> an empty list and then parse statements and
add them to the list until we reach the end of the block, marked by the closing
`}`. Note that the loop also has an explicit check for `isAtEnd()`. We have to
be careful to avoid infinite loops, even when parsing invalid code. If the user
forgets a closing `}`, the parser needs to not get stuck.

<aside name="list">

Having `block()` return the raw list of statements and leaving it to
`statement()` to wrap the list in a Stmt.Block looks a little odd. I did it that
way because we'll reuse `block()` later for parsing function bodies and we don't
want that body wrapped in a Stmt.Block.

</aside>

That's it for syntax. For semantics, we add another visit method to Interpreter.

^code visit-block

To execute a block, we create a new environment for the block's scope and pass
it off to this other method:

^code execute-block

This new method executes a list of statements in the context of a given <span
name="param">environment</span>. Up until now, the `environment` field in
Interpreter always pointed to the same environment -- the global one. Now, that
field represents the *current* environment. That's the environment that
corresponds to the innermost scope containing the code to be executed.

To execute code within a given scope, this method updates the interpreter's
`environment` field, visits all of the statements, and then restores the
previous value. As is always good practice in Java, it restores the previous
environment using a finally clause. That way it gets restored even if an
exception is thrown.

<aside name="param">

Manually changing and restoring a mutable `environment` field feels inelegant.
Another classic approach is to explicitly pass the environment as a parameter to
each visit method. To "change" the environment, you pass a different one as you
recurse down the tree. You don't have to restore the old one, since the new one
lives on the Java stack and is implicitly discarded when the interpreter returns
from the block's visit method.

I considered that for jlox, but it's kind of tedious and verbose adding an
environment parameter to every single visit method. To keep the book a little
simpler, I went with the mutable field.

</aside>

Surprisingly, that's all we need to do in order to fully support local
variables, nesting, and shadowing. Go ahead and try this out:

```lox
var a = "global a";
var b = "global b";
var c = "global c";
{
  var a = "outer a";
  var b = "outer b";
  {
    var a = "inner a";
    print a;
    print b;
    print c;
  }
  print a;
  print b;
  print c;
}
print a;
print b;
print c;
```

Our little interpreter can remember things now. We are inching closer to
something resembling a full-featured programming language.

<div class="challenges">

## Challenges

1.  The REPL no longer supports entering a single expression and automatically
    printing its result value. That's a drag. Add support to the REPL to let
    users type in both statements and expressions. If they enter a statement,
    execute it. If they enter an expression, evaluate it and display the result
    value.

2.  Maybe you want Lox to be a little more explicit about variable
    initialization. Instead of implicitly initializing variables to `nil`, make
    it a runtime error to access a variable that has not been initialized or
    assigned to, as in:

    ```lox
    // No initializers.
    var a;
    var b;
    
    a = "assigned";
    print a; // OK, was assigned first.
    
    print b; // Error!
    ```

3.  What does the following program do?

    ```lox
    var a = 1;
    {
      var a = a + 2;
      print a;
    }
    ```

    What did you *expect* it to do? Is it what you think it should do? What
    does analogous code in other languages you are familiar with do? What do
    you think users will expect this to do?

</div>

<div class="design-note">

## Design Note: Implicit Variable Declaration

Lox has distinct syntax for declaring a new variable and assigning to an
existing one. Some languages collapse those to only assignment syntax. Assigning
to a non-existent variable automatically brings it into being. This is called
**implicit variable declaration** and exists in Python, Ruby, and CoffeeScript,
among others. JavaScript has an explicit syntax to declare variables, but can
also create new variables on assignment. Visual Basic has [an option to enable
or disable implicit variables][vb].

[vb]: https://msdn.microsoft.com/en-us/library/xe53dz5w(v=vs.100).aspx

When the same syntax can assign or create a variable, each language must decide
what happens when it isn't clear about which behavior the user intends. In
particular, each language must choose how implicit declaration interacts with
shadowing, and which scope an implicitly declared variable goes into.

*   In Python, assignment always creates a variable in the current function's
    scope, even if there is a variable with the same name declared outside of
    the function.

*   Ruby avoids some ambiguity by having different naming rules for local and
    global variables. However, blocks in Ruby (which are more like closures than
    like "blocks" in C) have their own scope, so it still has the problem.
    Assignment in Ruby assigns to an existing variable outside of the current
    block if there is one with the same name. Otherwise, it creates a new
    variable in the current block's scope.

*   CoffeeScript, which takes after Ruby in many ways, is similar. It explicitly
    disallows shadowing by saying that assignment always assigns to a variable
    in an outer scope if there is one, all the way up to the outermost global
    scope. Otherwise, it creates the variable in the current function scope.

*   In JavaScript, assignment modifies an existing variable in any enclosing
    scope, if found. If not, it implicitly creates a new variable in the
    *global* scope.

The main advantage to implicit declaration is simplicity. There's less syntax
and no "declaration" concept to learn. Users can just start assigning stuff and
the language figures it out.

Older, statically typed languages like C benefit from explicit declaration
because they give the user a place to tell the compiler what type each variable
has and how much storage to allocate for it. In a dynamically typed,
garbage-collected language, that isn't really necessary, so you can get away
with making declarations implicit. It feels a little more "scripty", more "you
know what I mean".

But is that a good idea? Implicit declaration has some problems.

*   A user may intend to assign to an existing variable, but may have misspelled
    it. The interpreter doesn't know that, so it goes ahead and silently creates
    some new variable and the variable the user wanted to assign to still has
    its old value. This is particularly heinous in JavaScript where a typo will
    create a *global* variable, which may in turn interfere with other code.

*   JS, Ruby, and CoffeeScript use the presence of an existing variable with the
    same name -- even in an outer scope -- to determine whether or not an
    assignment creates a new variable or assigns to an existing one. That means
    adding a new variable in a surrounding scope can change the meaning of
    existing code. What was once a local variable may silently turn into an
    assignment to that new outer variable.

*   In Python, you may *want* to assign to some variable outside of the current
    function instead of creating a new variable in the current one, but you
    can't.

Over time, the languages I know with implicit variable declaration ended up
adding more features and complexity to deal with these problems.

*   Implicit declaration of global variables in JavaScript is universally
    considered a mistake today. "Strict mode" disables it and makes it a compile
    error.

*   Python added a `global` statement to let you explicitly assign to a global
    variable from within a function. Later, as functional programming and nested
    functions became more popular, they added a similar `nonlocal` statement to
    assign to variables in enclosing functions.

*   Ruby extended its block syntax to allow declaring certain variables to be
    explicitly local to the block even if the same name exists in an outer
    scope.

Given those, I think the simplicity argument is mostly lost. There is an
argument that implicit declaration is the right *default* but I personally find
that less compelling.

My opinion is that implicit declaration made sense in years past when most
scripting languages were heavily imperative and code was pretty flat. As
programmers have gotten more comfortable with deep nesting, functional
programming, and closures, it's become much more common to want access to
variables in outer scopes. That makes it more likely that users will run into
the tricky cases where it's not clear whether they intend their assignment to
create a new variable or reuse a surrounding one.

So I prefer explicitly declaring variables, which is why Lox requires it.

</div>
