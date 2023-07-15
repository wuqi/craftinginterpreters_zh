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

还记得 `statement()` 会在没有其它语句匹配时会尝试解析一个表达式语句吗？而 `expression()` 如果无法在当前语法标记处解析表达式，则会抛出一个语法错误？这一系列调用链可以保证在解析无效的声明或语句时能报告错误。

当解析器匹配到一个 `var` 标记时，它会进入相应的处理分支：

^code parse-var-declaration

一如既往，递归下降代码遵循语法规则。解析器已经匹配了 `var` 标记，因此接下来需要并消耗一个标识符标记作为变量名。

然后，如果遇到 `=` 标记，解析器就知道后面有一个初始化表达式，并对其进行解析。否则，它将初始化为 `null`。最后，解析器会消耗语句末尾必需的分号。所有这些都被包装在一个Stmt.Var语法树节点中，然后我们就完成解析了。

解析变量表达式更加简单。在 `primary()` 函数中，我们需要查找一个标识符标记。

^code parse-identifier (2 before, 2 after)

这样我们就有了一个可以声明和使用变量的工作前端。剩下的就是将其接入解释器。在此之前，我们需要讨论变量在内存中的位置。

## 环境

将变量与值关联的绑定需要存储在某个地方。自从Lisp发明了圆括号以来，这个数据结构就被称为<span name="env">**环境**</span>。


<img src="image/statements-and-state/environment.png" alt="An environment containing two bindings." />

<aside name="env">

我喜欢将环境想象成一个真实的仙境，变量和值在其中嬉戏玩耍。

</aside>

你可以将其想象成一个<span name="map">映射</span>，其中键是变量名，值是变量的值。实际上，这就是我们在Java中实现的方式。我们可以将这个映射和管理它的代码直接放入解释器中，但由于它形成了一个清晰的概念，我们可以将其提取到一个独立的类中。

创建一个新文件并添加以下内容：

<aside name="map">

Java称之为**maps**或**hashmaps**。其他语言称之为**hash tables**、**dictionaries**（Python和C#）、**hashes**（Ruby和Perl）、**tables**（Lua）或**associative arrays**（PHP）。在很久以前，它们被称为**scatter tables**。

</aside>

^code environment-class

在其中有一个Java Map用于存储绑定。它使用裸字符串作为键，而不是标记（tokens）。标记表示源文本中特定位置的代码单元，但在查找变量时，所有具有相同名称的标识符标记应该引用同一个变量（暂时忽略作用域）。使用裸字符串可以确保所有这些标记都指向同一个映射键。

我们需要支持两个操作。首先，是变量定义，将一个新的名称与一个值绑定在一起。

^code environment-define

虽然不是特别复杂，但是我们这里也做出了一个有趣的语义抉择。当我们向映射中添加键时，没有检查该键是否已存在。这意味着下面的程序可以正常工作：

```lox
var a = "before";
print a; // "before".
var a = "after";
print a; // "after".
```

一个变量声明不仅仅定义一个*新*变量，还可以用来*重新*定义一个已有变量。我们可以<span name="scheme">选择</span>将这视为错误来处理。用户可能并不打算重新定义一个已有变量（如果他们确实想这样做，他们可能会使用赋值语句而不是 `var` ）。将重新定义视为错误可以帮助用户找到这个bug。

但是，这样做会对REPL的使用造成不良影响。在REPL会话中，我们不需要费心去记住已经定义的变量。我们可以在REPL中允许重新定义变量，但在脚本中不允许。但这样一来，用户就需要学习两套规则，而且从一个地方复制粘贴到另一个地方的代码可能无法正常工作。

<aside name="scheme">

我关于变量和作用域的原则是：“当你不确定的时候，就按照Scheme的做法来”。Scheme的开发者花了很多时间思考变量的作用范围，他们的目标就是让变量的使用更清晰明了。所以，如果你遵循他们的方式，就不容易出错。

在Scheme中，你可以在顶层重新定义变量。（这就像是在一个程序的最外层给变量起了一个新的名字。这样做的好处是，你可以在不同的地方使用相同的变量名，而不会造成混淆。）

</aside>

所以，为了保持这两种模式的一致性，我们将允许重定义——至少对于全局变量来说是如此。一旦一个变量存在，我们就需要可以查找该变量的方法。

^code environment-get (2 before, 1 after)

这个情况在语义上有点意思。如果找到了这个变量，它就会直接返回与之绑定的值。但是如果没有找到呢？这时候我们又有几种选择：

* 抛出语法错误

* 抛出运行时错误.

* 允许该操作并返回默认值，比如 `nil`.

Lox是相当宽松的，但是最后一个选项对我来说有点太宽松了。把它作为一个语法错误，也就是编译时的错误，听起来是个聪明的选择。使用未定义的变量是一个bug，而且你越早发现这个错误，越好。

问题在于*使用*变量并不等同于*引用*变量。如果一段代码块被包裹在一个函数内部，你可以在这段代码中引用一个变量，而不需要立即对其进行求值。如果我们把*引用*未声明的变量定义为静态错误，那么定义递归函数将变得更加困难。

我们可以通过在检查函数体之前先声明函数自己的名字来处理单一递归——一个调用自身的函数。但是这对于互相递归调用的函数无效。考虑以下情况：

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

确实，这可能不是判断一个数字是偶数还是奇数的最好方法（更不用说如果传入一个非整数或负数会发生一些不可控的事情）。不过请忍耐一下。

</aside>

在我们查看`isOdd()`函数体的<span name="declare">时候</span>，`isEven()`函数还没有被定义。如果我们交换这两个函数的顺序，那么当我们查看`isEven()`函数体的时候，`isOdd()`函数也还没有被定义。

<aside name="declare">

有些静态类型的编程语言，比如Java和C#，为了解决这个问题，规定程序的顶层不是一连串的命令式语句。相反，它们规定一个程序是一组同时存在的声明。在实际执行时，会先声明*所有*的名字，然后再去看函数的具体内容。

而像C和Pascal这样的老式编程语言不是这样工作的。它们要求在一个名称完全定义之前，必须先添加明确的*前向声明*。这是为了适应当时计算机性能有限的情况而做出的妥协。它们希望能够通过一次遍历源代码的方式，在处理函数具体内容之前就能够编译整个源文件。
</aside>

因为把它作为*静态*错误会让递归声明变得太难，所以我们将错误推迟到运行时。只要不对变量引用进行*求值*，就可以在变量定义之前引用它。这样程序就可以正常工作，但是在以下情况下会出现运行时错误：

```lox
print a;
var a = "too late!";
```

就像表达式求值代码中的类型错误一样，我们通过抛出异常来报告运行时错误。异常中包含变量的标记，这样我们就可以告诉用户在他们的代码在哪里搞砸了。

### 解释全局变量

Interpreter类会获得一个新的Environment类的实例。

^code environment-field (2 before, 1 after)

我们将其直接存储为Interpreter中的字段，这样只要解释器还在运行，变量就会一直保留在内存中。

我们有两个新的语法树，因此需要两个新的visit方法。第一个是用于声明语句的。

^code visit-var

如果变量有初始化式，我们就对其求值。如果没有，我们就需要做一个选择。我们可以通过在解析器中*要求*必须有初始化式来将其视为语法错误。但是，大多数语言并不这么做，所以在Lox中这样做可能有点太过苛刻。

我们可以把它当作运行时错误。我们允许你定义一个未初始化的变量，但是如果在给它赋值之前访问它，就会发生一个运行时错误。这个想法不错，但是大多数动态类型的语言并不这么做。相反，我们将保持简单，规定在Lox中，如果变量没有被显式初始化，它就会被设置为`nil`。

```lox
var a;
print a; // "nil".
```

因此，如果没有初始化式，我们将值设为`null`，这也是Lox中的`nil`值的Java表示形式。然后，我们告诉环境上下文将变量与该值进行绑定。

接下来，我们要对变量表达式求值。

^code visit-variable

这里只是简单地将操作转发到环境上下文中，环境做了一些繁重的工作保证变量已被定义。这样，我们就可以支持基本的变量操作了。试试这个：

```lox
var a = 1;
var b = 2;
print a + b;
```

我们目前还不能重用*代码*，但我们可以开始构建重用*数据*的程序。

## 赋值

有些语言可以使用变量，但不允许你重新赋值或**修改**它们。比如Haskell就是这样的一种语言。SML只支持可变引用和数组，变量不能重新赋值。而Rust则通过要求使用`mut`修饰符来启用赋值，以避免变量的修改。

修改变量给它添加了副作用，有些语言专家认为这样的副作用<span name="pure">不够优雅</span>或者不够干净。他们认为代码应该像纯粹的数学一样，创造出一系列值，就像是神圣的创世一样，这些值是晶莹剔透、永恒不变的。而不是像一个肮脏的机器人，一次又一次地将数据进行改造，发出命令式的咕哝声。

<aside name="pure">

我发现有趣的是，那些以冷静逻辑为傲的人群，却无法抵挡对自己工作使用情感负荷的术语的诱惑：“纯净”，“副作用”，“惰性”，“持久”，“一流”，“高阶”（"pure", "side effect", "lazy", "persistent", "first-class",
"higher-order"）。

</aside>

Lox并不那么严肃。Lox是一种命令式语言，变量的修改是其中的一部分。添加对赋值的支持并不需要太多的工作。全局变量已经支持重新定义，所以大部分机制已经存在了。主要的问题是缺少显式的赋值表示法。

### Assignment syntax

这个小小的 `=` 语法比它看起来要复杂。和大多数C衍生语言一样，赋值是一个<span name="assign">表达式</span>而不是一个语句。就像在C语言中一样，它是最低优先级的表达式形式。这意味着这个规则位于表达式 `expression` 和等式  `equality` 之间（下一个最低优先级的表达式）。

<aside name="assign">

在其他一些语言中，比如Pascal、Python和Go，赋值是一个语句。

</aside>

```ebnf
expression     → assignment ;
assignment     → IDENTIFIER "=" assignment
               | equality ;
```

这就是说，一个赋值操作要么是一个标识符后面跟着一个等号和一个表达式来表示赋给该标识符的值，要么是一个等式（或其他）表达式。稍后，当我们在对象上添加属性设置器时，赋值操作将变得更加复杂，比如：

```lox
instance.field = "value";
```

最简单的部分就是添加<span name="assign-ast">新的语法树节点</span>。

^code assign-expr (1 before, 1 after)

<aside name="assign-ast">

新节点的生成代码在以下位置： [Appendix II][appendix-assign].

[appendix-assign]: appendix-ii.html#assign-expression

</aside>

它包含了用于赋值的变量的标记，以及表示新值的表达式。当你运行AstGenerator生成新的Expr.Assign类后，需要将解析器现有的 `expression()` 方法的代码替换为与更新规则相匹配的代码。

^code expression (1 before, 1 after)

这里开始变得棘手。在单个标记的前瞻递归下降解析器中，直到解析完左侧并遇到 `=` *之后*，它才能确定是否正在解析一个赋值语句。你可能会好奇为什么需要这样做。毕竟，我们在解析左操作数之后才能确定是否解析的是一个 `+` 加法表达式。

左侧赋值语句与普通表达式的区别在于，它不是一个可以求值的表达式，而是一种伪表达式，它的求值结果是一个可以赋值的"对象"。让我们来看一个例子：

```lox
var a = "before";
a = "value";
```

在第二行，我们不对变量`a`进行*求值*（这将返回字符串"before"）。我们通过确定变量`a`引用的内容，来确定在哪里存储右侧表达式的值。对这两个<span name="l-value">概念</span>的[经典术语][l-value]是**左值**和**右值**。到目前为止，我们所见到的所有产生值的表达式都是右值。而左值则"求值"为一个可以进行赋值的存储位置。

[l-value]: https://en.wikipedia.org/wiki/Value_(computer_science)#lrvalue

<aside name="l-value">

实际上，这些术语来源于赋值表达式：左值*l*-value出现在赋值语句中的*左侧*，而右值*r*-value出现在*右侧*。

</aside>

We want the syntax tree to reflect that an l-value isn't evaluated like a normal
expression. That's why the Expr.Assign node has a *Token* for the left-hand
side, not an Expr. The problem is that the parser doesn't know it's parsing an
l-value until it hits the `=`. In a complex l-value, that may occur <span
name="many">many</span> tokens later.

我们希望语法树能够准确反映出左值与普通表达式的求值方式不同。这就是为什么Expr.Assign节点的左侧使用了一个*Token*，而不是一个Expr。然而，问题在于解析器在遇到等号（=）之前并不知道它正在解析一个左值。在复杂的左值中，可能在出现<span
name="many">很多</span>标记之后才能识别到。

```lox
makeList().head.next = node;
```

<aside name="many">

由于字段赋值的接收者可以是任何表达式，并且表达式可以任意长，因此可能需要*无限*数量的前瞻标记才能找到等号（`=`）。

</aside>

我们只会前瞻一个标记，那么我们应该如何处理呢？我们可以使用一个小技巧，具体如下：

^code parse-assignment

大部分解析赋值表达式的代码与其他二元操作符（如`+`）的代码非常相似。我们首先解析左侧表达式，该表达式可以是任何优先级较高的表达式。如果我们找到一个等号（`=`），则解析右侧表达式，并将其封装在一个赋值表达式的语法树节点中。

<aside name="no-throw">

如果左侧不是一个有效的赋值目标，我们会*报告*一个错误，但我们不会*抛出*异常。因为解析器没有陷入混乱状态，所以我们不需要进入紧急模式并进行同步。

</aside>

与二元操作符稍有不同的是，我们不会循环构建相同操作符的序列。由于赋值是右结合的，我们会通过递归调用 `assignment()` 来解析右侧表达式。

关键是在创建赋值表达式节点之前，我们会查看左侧表达式，并确定它是何种类型的赋值目标。我们将右侧表达式节点转换为左侧值的表示形式。


This conversion works because it turns out that every valid assignment target
happens to also be <span name="converse">valid syntax</span> as a normal
expression. Consider a complex field assignment like:
这种转换有效的原因是，每个有效的赋值目标正好也是符合普通表达式的<span name="converse">有效语法</span>。考虑一个复杂的字段赋值，例如：

<aside name="converse">

即使存在一些不是有效表达式的赋值目标，您仍然可以使用这个技巧。定义一个*包容性的语法*，它可以接受所有有效的表达式和赋值目标的语法。当遇到一个等号（=）时，如果左侧*不符合*有效的赋值目标的语法，则报告错误。反之，如果*没有*遇到等号（=），则如果左侧不是一个有效的*表达式*，也报告错误。
</aside>

```lox
newPoint(x + 2, 0).y = 3;
```

该赋值表达式的左侧也是一个有效的表达式。


```lox
newPoint(x + 2, 0).y;
```

第一个示例是设置字段的值，第二个示例是获取字段的值。

这意味着我们可以将左侧*视为*一个表达式进行解析，然后在解析完成后生成一个语法树，将其转换为赋值目标。如果左侧的表达式不是一个<span name="paren">有效</span>的赋值目标，我们将报告语法错误。这样可以确保在遇到类似下面的代码时会报告错误：

```lox
a + b = c;
```

<aside name="paren">

早在解析那章，我就说过我们要在语法树中表示圆括号表达式，因为我们将在后续使用它们。这就是原因。我们需要能够区分以下情况：

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
