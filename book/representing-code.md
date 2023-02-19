> 对于居住在树林里的人来说，几乎每一种树都有它的声音和特点。
> <cite>Thomas Hardy, <em>Under the Greenwood Tree</em></cite>

在[上一章][scanning]中，我们将原始的源代码作为字符串，并将其转换为一个稍高层次的表示：一系列的 token 。我们将在[下一章][parsing]中编写的解析器，将这些 token 再次转化为更丰富、更复杂的表达。

[scanning]: scanning.html
[parsing]: parsing-expressions.html

在我们能够产生这种表示法之前，我们需要定义它。这就是本章的主题。在这一过程中，我们将<span name="boring">介绍</span>一些围绕形式化语法的理论，感受函数式编程和面向对象编程之间的区别，讨论几个设计模式，并实现一些元编程。


<aside name="boring">

我非常担心这是本书中最无聊的一章，所以我不断地把更多有趣的想法塞进去，直到没有空间了。

</aside>

在我们做所有这些之前，让我们关注一下主要目标——代码的表示。
句法分析器应该便于生产，解释器也应该易于消费。如果你还没有编写一个句法分析器或解释器，那么这些需求并不明确。也许你的直觉能帮上忙。当你扮演*人工*翻译的角色时，你的大脑在做什么？你如何心算这样一个算术表达式:

```lox
1 + 2 * 3 - 4
```

因为你了解运算顺序--  像[Please Excuse My Dear Aunt Sally][sally]的<span name="PEMDAS">口诀</span>中说的--你知道乘法要在加法或减法之前计算。有一种方法是用一棵树来说明这种优先顺序。叶子结点是数字，内部结点是运算符，每个运算符都有分支。

[sally]: https://en.wikipedia.org/wiki/Order_of_operations#Mnemonics

<aside name="PEMDAS">

译注: 美国记运算的顺序的口诀：括号（Parenthesis）、指数(Exponent)、乘除（multiplication and division）、加减（addition and subtraction），编成了Please Excuse My Dear Aunt Sally （请原谅我亲爱的阿姨莎莉）。

</aside>

为了计算一个算术节点，你需要知道其子节点的数值，所以你必须先计算这些子节点。这意味着你要从叶节点向根部遍历--即*后序遍历*。

<span name="tree-steps"></span>

<img src="image/representing-code/tree-evaluate.png" alt="Evaluating the tree from the bottom up." />

<aside name="tree-steps">

A. 从整个树开始，首先计算最底部的操作, `2 * 3`.

B. 然后我们做加法 `+` .

C. 继续, 做减法 `-`.

D. 得到最终结果！

</aside>

如果我给你一个算术表达式，你可以很容易地画出这些树。给定一棵树，你可以毫不费力地计算它。因此，从直觉上看，我们代码的可以表示为一棵与语言的语法结构(运算符嵌套)相匹配的<span name="only">树</span>。

<aside name="only">

这并不是说树是我们代码的*唯一*的表示方法。在[Part III][]中，我们将生成字节码，这是另一种不太友好但更接近机器的表示方法。

[part iii]: a-bytecode-virtual-machine.html

</aside>

那么我们需要更精确地了解语法是什么。就像上一章中的词法语法一样，围绕着句法语法有大量的理论。我们对这个理论的研究比扫描时多一点，因为它在解释器中是一个有用的工具。我们从[Chomsky hierarchy][]层次结构的第一层开始...

[chomsky hierarchy]: https://en.wikipedia.org/wiki/Chomsky_hierarchy

## 上下文无关文法

在上一章中，我们用来定义词汇语法的样式——即字符如何组合成 token 的规则——被称为*正则语言*。这对我们的扫描器来说很完美，它发射出死板的 token 序列。但是正则语言没有强大到足以处理可以任意深度嵌套的表达式。

我们需要一把更大的锤子，那把锤子就是 **上下文无关文法** (**CFG**)。它是 **[形式语法][]** (formal grammars)工具箱中第二重的工具。一种形式语法由一组原子片段组成，称之为“字母表”(alphabet)。然后，它定义一组(通常是无限的)语法"中"的“字符串”。每个字符串都是字母表中的“字母”序列。

[形式语法]: https://en.wikipedia.org/wiki/Formal_grammar

我之所以使用所有这些引号，是因为从词法到句法的语法时，这些术语会变得有点混乱。在我们的扫描器语法中，字母表由单个字符组成，字符串是有效的词组--大致是 "单词"。在我们现在讨论的句法语法中，我们处于不同的粒度水平。现在，字母表中的每个 "字母" 都是一个完整的 token ，而 "字符串 "是一个 *tokens* 的序列--一个完整的表达式。

哦，也许一张表格会有些帮助:

<table>
<thead>
<tr>
  <td>术语</td>
  <td></td>
  <td>词法语法</td>
  <td>句法语法</td>
</tr>
</thead>
<tbody>
<tr>
  <td> &ldquo;字母表&rdquo; 是<span class="ellipse">&thinsp;.&thinsp;.&thinsp;.</span></td>
  <td>&rarr;&ensp;</td>
  <td>字符</td>
  <td>Tokens</td>
</tr>
<tr>
  <td> &ldquo;字符串&rdquo; 是<span class="ellipse">&thinsp;.&thinsp;.&thinsp;.</span></td>
  <td>&rarr;&ensp;</td>
  <td>Lexeme 或 token</td>
  <td>表达式</td>
</tr>
<tr>
  <td>由什么实现<span class="ellipse">&thinsp;.&thinsp;.&thinsp;.</span></td>
  <td>&rarr;&ensp;</td>
  <td>扫描器</td>
  <td>句法分析器</td>
</tr>
</tbody>
</table>

形式语法的工作是指定哪些字符串是有效的，哪些是无效的。如果我们为英语句子定义一个语法，"鸡蛋是美味的早餐 "是正确的语法，但 "美味的早餐是鸡蛋 "则不是。

### 语法规则

我们如何描述一个包含无限个有效字符串的语法？显然，我们不能把他们全部列出来。我们创建了一个有限的规则集作为替代，你可以把它想象成一个二选一的游戏。

如果你从规则开始，你可以用它们来 *生成* 语法中的字符串。以这种方式创建的字符串被称为 **派生** ，因为每个字符串都是从语法的规则中派生出来的。在游戏的每一步中，你选择一条规则，然后按照它告诉你的去做。形式语法的大部分行话都是从这里来的。规则被称为 **产生式** ，因为它们在 *语法* 中产生字符串。

上下文无关文法中的每个产生式都有一个 **头** -- 包含<span name="name">名称</span>--和 **主体**，用于描述它生成的内容。在其纯粹的形式中，主体只是一个符号的列表。符号有两种可选口味:

<aside name="name">

将头限制在一个符号上是上下文无关文法的一个决定性特征。更强大的形式主义，如 **[无限制文法][]** ，允许在头和主体中有一系列符号。

[无限制文法]: https://en.wikipedia.org/wiki/Unrestricted_grammar

</aside>

*   **终结符** 是语法字母表中的一个字符。你可以把它看成是一个字面值。在我们定义的句法语法中，终结符是来自扫描器的单个 lexemes--token ，如 `if` 或 `1234` 。

    这些被称为 "终结符"，取 "终点" 之意，因为它们不会导致游戏中的任何进一步 "行动"。你只是产生一个符号。

*   **非终结符** 是对语法中另一条规则的命名引用。它意味着 "执行该规则并在这里插入它产生的任何内容"。这样一来，语法就构成了。

还有最后一项改进：你可以有多个同名的规则。当你到达一个具有该名称的非终结符时，你可以为它选择任何一条规则，以你的想法为准。

为了使之具体化，我们需要一种<span name="turtles">方法</span>来写下这些产生式规则。人们一直在试图将语法具体化，这可以追溯到古印度语法学家波你尼撰写的《声明论》(亦称《八章书》) ，他在几千年前就将梵文语法编成了法典。直到John Backus和其公司需要一个用于指定ALGOL 58语言的记法，并提出了[**Backus-Naur form**][bnf]（ **BNF** 巴科斯范式），才有了很大的进展。从那时起，几乎每个人都使用某种形式的BNF，并根据自己的口味进行调整。

[bnf]: https://en.wikipedia.org/wiki/Backus%E2%80%93Naur_form

我试图想出一些简洁的方法。每条规则都是一个名字，后面跟一个箭头（ `→` ），然后是一连串的符号，最后以分号（`;`）结束。终结符是带引号的字符串，非终结符是小写的单词。

<aside name="turtles">

是的，我们需要定义一种语法，用于定义我们的语法的规则。我们是否也应该指定这种 *元语法* ？我们用什么记法来表示 *它* ？一路下来都是语言!

</aside>

利用这种方式，下面是<span name="breakfast">早餐</span>菜单的语法。

<aside name="breakfast">

是的，我真的要在这整本书中使用早餐的例子。对不起。

</aside>

```ebnf
breakfast  → protein "with" breakfast "on the side" ;
breakfast  → protein ;
breakfast  → bread ;

protein    → crispiness "crispy" "bacon" ;
protein    → "sausage" ;
protein    → cooked "eggs" ;

crispiness → "really" ;
crispiness → "really" crispiness ;

cooked     → "scrambled" ;
cooked     → "poached" ;
cooked     → "fried" ;

bread      → "toast" ;
bread      → "biscuits" ;
bread      → "English muffin" ;
```

我们可以使用这个语法来生成随机早餐。我们来玩一轮，看看效果如何。按照古老的惯例，游戏从语法中的第一条规则开始，在这里是 `breakfast`。有三个产生式，我们随机选择第一个。我们得到的字符串如下所示:

```text
protein "with" breakfast "on the side"
```

我们需要扩展第一个非终结符，即 `protein` ，所以我们为它选择一个产生式。我们来选一个:

```ebnf
protein → cooked "eggs" ;
```

接下来，我们需要为 `cooked` 选一个产生式，所以我们选择  `"poached"`。那是一个终结符，所以我们把它加进去。现在我们的字符串看起来像:

```text
"poached" "eggs" "with" breakfast "on the side"
```

下一个非终结符又是 `breakfast` 。我们选择的第一个 `breakfast` 产生式递归引用了 `breakfast` 规则。语法中的递归是一个很好的标志，说明被定义的语言是上下文无关的，而不是正则的。特别是，递归非终结符 <span name="nest">两边</span>都有产生式的递归，意味着这种语言不是正则的。

<aside name="nest">

想象一下，我们在这里把 `breakfast` 规则递归扩展了好几次，比如"bacon with bacon with bacon with..." 。为了正确地完成这个字符串，我们需要在结尾处添加*同等* 数量的 "on the side" 。跟踪所需尾部的数量超出了正则语法的能力。正则语法可以表达重复，但它们不能统计有多少次重复，而这对于确保字符串有相同数量的 `with` 和 `on the side` 部分是必要的。

</aside>

我们可以一次又一次地挑选第一种产生式作为 `breakfast` ，产生各种形式的早餐，如 "bacon with sausage with scrambled eggs with bacon..."。但我们不会这样做。这一次，我们将选择 `bread`。`bread` 有三条规则，每条都只包含一个终结符。我们将选择 "English muffin"。

这样，字符串中的每个非终结符都被展开，直到最后只包含终结符，我们只剩下:

<img src="image/representing-code/breakfast.png" alt='"Playing" the grammar to generate a string.' />

扔进去一些火腿和荷兰酱，你就有了班尼迪克蛋。

每当我们碰到一个包含多个产生式的规则时，我们都会任意选择一个。正是这种灵活性允许少量的语法规则编码组合上更大的字符串集。一个规则可以直接或间接地引用自身，这一事实使它更加灵活，让我们可以把无限多的字符串装进一个有限的语法中。

### 强化我们的记法

把无限的字符串集塞进少量的规则中非常神奇，但是我们还可以更进一步。我们的记法是有效的，但它很乏味。因此，像任何一个好的语言设计者一样，我们将在上面撒上一点语法糖--一些额外的便利记法。除了终结符和非终结符之外，我们还允许在规则的主体中出现一些其他类型的表达式：

*   我们不需要每次都重复规则的名称，而是允许使用一系列用竖线（`|`）隔开产生式。

    ```ebnf
    bread → "toast" | "biscuits" | "English muffin" ;
    ```

*   此外，我们将允许用圆括号进行分组，然后允许 `|` 在其中从一系列的选项中选择一个产生式。

    ```ebnf
    protein → ( "scrambled" | "poached" | "fried" ) "eggs" ;
    ```

*   使用递归来支持重复的符号序列有一定的<span name="purity">吸引力</span>，但每次我们想要循环时，都要创建一个单独的命名子规则，这有点麻烦。因此，我们还使用后缀 `*` 来允许前面的符号或组重复零次或多次。

    ```ebnf
    crispiness → "really" "really"* ;
    ```

<aside name="purity">

这就是Scheme编程语言的工作方式。它根本就没有内置的循环功能。反之，所有的重复都是用递归来表达的。

</aside>

*   后缀 `+` 与此类似，但要求前面的产生式至少出现一次。

    ```ebnf
    crispiness → "really"+ ;
    ```

*   后缀"？"是指一个可选的产生式。它前面的东西可以出现零次或一次，但不能更多。

    ```ebnf
    breakfast → protein ( "with" breakfast "on the side" )? ;
    ```

有了所有这些语法细节，我们的早餐语法概括为：

```ebnf
breakfast → protein ( "with" breakfast "on the side" )?
          | bread ;

protein   → "really"+ "crispy" "bacon"
          | "sausage"
          | ( "scrambled" | "poached" | "fried" ) "eggs" ;

bread     → "toast" | "biscuits" | "English muffin" ;
```

我希望不是太糟糕。如果您习惯于在文本编辑器中使用grep或[正则表达式][regex]，那么大多数标点符号应该很熟悉。主要区别在于这里的符号表示整个 token ，而不是单个字符。

[regex]: https://en.wikipedia.org/wiki/Regular_expression#Standards

在本书的其余部分，我们将使用这个记法来精确描述Lox的语法。当你研究编程语言时，你会发现上下文无关文法（使用这个或 [EBNF][] 或其他记法）可以帮助你将你的非正式语法设计想法具体化。它们也是与其他语言黑客交流语法的一个方便的媒介。

[ebnf]: https://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_form

我们为Lox定义的规则和产生式也是我们要实现的树状数据结构的指南，以表示内存中的代码。在做到这一点之前，我们需要一个实际的Lox语法，或者至少有足够的语法让我们开始工作。

### Lox表达式的语法

在上一章中，我们一举完成了Lox的整个词法语法。每个关键词和标点符号都在那里。句法语法更大，在我们真正启动和运行我们的解释器之前，从头到尾研究一遍实在是一件无聊的事情。

因此，我们将在接下来的几章中学习这门语言的子集。一旦我们有了这种小型语言的表示、解析和解释，那么后面的章节将逐步为它增加新的特性，包括新的语法。现在，我们只关注少数几个表达式:

*   **字面值.** 数字, 字符串, 布尔值, 以及 `nil`.

*   **一元表达式.** 前缀 `!` 用于执行逻辑非，而 `-` 用于对一个数求反。

*   **二元表达式.** 我们熟知并热爱的中缀算术运算符(`+`, `-`, `*`, `/`)和逻辑运算符(`==`, `!=`, `<`, `<=`, `>`, `>=`)。

*   **圆括号.** 一对 `(` 和 `)` 包裹着一个表达式。

这为我们提供了足够的语法来表达如下表达式：

```lox
1 - (2 * 3) < 4 == false
```

使用我们方便时髦的新记法，语法可以表示为:

```ebnf
expression     → literal
               | unary
               | binary
               | grouping ;

literal        → NUMBER | STRING | "true" | "false" | "nil" ;
grouping       → "(" expression ")" ;
unary          → ( "-" | "!" ) expression ;
binary         → expression operator expression ;
operator       → "==" | "!=" | "<" | "<=" | ">" | ">="
               | "+"  | "-"  | "*" | "/" ;
```

这里有一点额外的<span name="play">元语法</span>。除了对匹配精确 lexeme 的终结符使用加引号的字符串外，我们还对单一 lexeme 的终结符使用大写字母，这些 lexeme 的文本表示可能有所不同。 `NUMBER` 是任意数字直接量，`STRING` 是任意字符串直接量。稍后，我们将对 `IDENTIFIER` 做同样的处理。

这个语法实际上是不明确的，我们在解析它的时候会看到这一点。但是现在已经足够好了。

<aside name="play">

如果你愿意，可以试着用这个语法生成一些表达式，就像我们之前用早餐语法做的那样。你觉得结果表达式看起来对吗？你能让它产生像 `1 + / 3` 这样的错误吗？

</aside>

## 实现语法树

最后，我们可以写一些代码了。那个小小的表达式语法就是我们的骨架。由于语法是递归的--注意`grouping`, `unary`和`binary`都是指 `expression` --我们的数据结构将形成一棵树。由于这个结构代表了我们语言的语法，所以它被称为<span name="ast">**语法树**</span>。

<aside name="ast">

In particular, we're defining an **abstract syntax tree** (**AST**). In a
**parse tree**, every single grammar production becomes a node in the tree. An
AST elides productions that aren't needed by later phases.

</aside>

Our scanner used a single Token class to represent all kinds of lexemes. To
distinguish the different kinds -- think the number `123` versus the string
`"123"` -- we included a simple TokenType enum. Syntax trees are not so <span
name="token-data">homogeneous</span>. Unary expressions have a single operand,
binary expressions have two, and literals have none.

We *could* mush that all together into a single Expression class with an
arbitrary list of children. Some compilers do. But I like getting the most out
of Java's type system. So we'll define a base class for expressions. Then, for
each kind of expression -- each production under `expression` -- we create a
subclass that has fields for the nonterminals specific to that rule. This way,
we get a compile error if we, say, try to access the second operand of a unary
expression.

<aside name="token-data">

Tokens aren't entirely homogeneous either. Tokens for literals store the value,
but other kinds of lexemes don't need that state. I have seen scanners that use
different classes for literals and other kinds of lexemes, but I figured I'd
keep things simpler.

</aside>

Something like this:

```java
package com.craftinginterpreters.lox;

abstract class Expr { // [expr]
  static class Binary extends Expr {
    Binary(Expr left, Token operator, Expr right) {
      this.left = left;
      this.operator = operator;
      this.right = right;
    }

    final Expr left;
    final Token operator;
    final Expr right;
  }

  // Other expressions...
}
```

<aside name="expr">

I avoid abbreviations in my code because they trip up a reader who doesn't know
what they stand for. But in compilers I've looked at, "Expr" and "Stmt" are so
ubiquitous that I may as well start getting you used to them now.

</aside>

Expr is the base class that all expression classes inherit from. As you can see
from `Binary`, the subclasses are nested inside of it. There's no technical need
for this, but it lets us cram all of the classes into a single Java file.

### Disoriented objects

You'll note that, much like the Token class, there aren't any methods here. It's
a dumb structure. Nicely typed, but merely a bag of data. This feels strange in
an object-oriented language like Java. Shouldn't the class *do stuff*?

The problem is that these tree classes aren't owned by any single domain. Should
they have methods for parsing since that's where the trees are created? Or
interpreting since that's where they are consumed? Trees span the border between
those territories, which means they are really owned by *neither*.

In fact, these types exist to enable the parser and interpreter to
*communicate*. That lends itself to types that are simply data with no
associated behavior. This style is very natural in functional languages like
Lisp and ML where *all* data is separate from behavior, but it feels odd in
Java.

Functional programming aficionados right now are jumping up to exclaim "See!
Object-oriented languages are a bad fit for an interpreter!" I won't go that
far. You'll recall that the scanner itself was admirably suited to
object-orientation. It had all of the mutable state to keep track of where it
was in the source code, a well-defined set of public methods, and a handful of
private helpers.

My feeling is that each phase or part of the interpreter works fine in an
object-oriented style. It is the data structures that flow between them that are
stripped of behavior.

### Metaprogramming the trees

Java can express behavior-less classes, but I wouldn't say that it's
particularly great at it. Eleven lines of code to stuff three fields in an
object is pretty tedious, and when we're all done, we're going to have 21 of
these classes.

I don't want to waste your time or my ink writing all that down. Really, what is
the essence of each subclass? A name, and a list of typed fields. That's it.
We're smart language hackers, right? Let's <span
name="automate">automate</span>.

<aside name="automate">

Picture me doing an awkward robot dance when you read that. "AU-TO-MATE."

</aside>

Instead of tediously handwriting each class definition, field declaration,
constructor, and initializer, we'll hack together a <span
name="python">script</span> that does it for us. It has a description of each
tree type -- its name and fields -- and it prints out the Java code needed to
define a class with that name and state.

This script is a tiny Java command-line app that generates a file named
"Expr.java":

<aside name="python">

I got the idea of scripting the syntax tree classes from Jim Hugunin, creator of
Jython and IronPython.

An actual scripting language would be a better fit for this than Java, but I'm
trying not to throw too many languages at you.

</aside>

^code generate-ast

Note that this file is in a different package, `.tool` instead of `.lox`. This
script isn't part of the interpreter itself. It's a tool *we*, the people
hacking on the interpreter, run ourselves to generate the syntax tree classes.
When it's done, we treat "Expr.java" like any other file in the implementation.
We are merely automating how that file gets authored.

To generate the classes, it needs to have some description of each type and its
fields.

^code call-define-ast (1 before, 1 after)

For brevity's sake, I jammed the descriptions of the expression types into
strings. Each is the name of the class followed by `:` and the list of fields,
separated by commas. Each field has a type and a name.

The first thing `defineAst()` needs to do is output the base Expr class.

^code define-ast

When we call this, `baseName` is "Expr", which is both the name of the class and
the name of the file it outputs. We pass this as an argument instead of
hardcoding the name because we'll add a separate family of classes later for
statements.

Inside the base class, we define each subclass.

^code nested-classes (2 before, 1 after)

<aside name="robust">

This isn't the world's most elegant string manipulation code, but that's fine.
It only runs on the exact set of class definitions we give it. Robustness ain't
a priority.

</aside>

That code, in turn, calls:

^code define-type

There we go. All of that glorious Java boilerplate is done. It declares each
field in the class body. It defines a constructor for the class with parameters
for each field and initializes them in the body.

Compile and run this Java program now and it <span name="longer">blasts</span>
out a new &ldquo;.java" file containing a few dozen lines of code. That file's
about to get even longer.

<aside name="longer">

[Appendix II][] contains the code generated by this script once we've finished
implementing jlox and defined all of its syntax tree nodes.

[appendix ii]: appendix-ii.html

</aside>

## Working with Trees

Put on your imagination hat for a moment. Even though we aren't there yet,
consider what the interpreter will do with the syntax trees. Each kind of
expression in Lox behaves differently at runtime. That means the interpreter
needs to select a different chunk of code to handle each expression type. With
tokens, we can simply switch on the TokenType. But we don't have a "type" enum
for the syntax trees, just a separate Java class for each one.

We could write a long chain of type tests:

```java
if (expr instanceof Expr.Binary) {
  // ...
} else if (expr instanceof Expr.Grouping) {
  // ...
} else // ...
```

But all of those sequential type tests are slow. Expression types whose names
are alphabetically later would take longer to execute because they'd fall
through more `if` cases before finding the right type. That's not my idea of an
elegant solution.

We have a family of classes and we need to associate a chunk of behavior with
each one. The natural solution in an object-oriented language like Java is to
put those behaviors into methods on the classes themselves. We could add an
abstract <span name="interpreter-pattern">`interpret()`</span> method on Expr
which each subclass would then implement to interpret itself.

<aside name="interpreter-pattern">

This exact thing is literally called the ["Interpreter pattern"][interp] in
*Design Patterns: Elements of Reusable Object-Oriented Software*, by Erich
Gamma, et al.

[interp]: https://en.wikipedia.org/wiki/Interpreter_pattern

</aside>

This works alright for tiny projects, but it scales poorly. Like I noted before,
these tree classes span a few domains. At the very least, both the parser and
interpreter will mess with them. As [you'll see later][resolution], we need to
do name resolution on them. If our language was statically typed, we'd have a
type checking pass.

[resolution]: resolving-and-binding.html

If we added instance methods to the expression classes for every one of those
operations, that would smush a bunch of different domains together. That
violates [separation of concerns][] and leads to hard-to-maintain code.

[separation of concerns]: https://en.wikipedia.org/wiki/Separation_of_concerns

### The expression problem

This problem is more fundamental than it may seem at first. We have a handful of
types, and a handful of high-level operations like "interpret". For each pair of
type and operation, we need a specific implementation. Picture a table:

<img src="image/representing-code/table.png" alt="A table where rows are labeled with expression classes, and columns are function names." />

Rows are types, and columns are operations. Each cell represents the unique
piece of code to implement that operation on that type.

An object-oriented language like Java assumes that all of the code in one row
naturally hangs together. It figures all the things you do with a type are
likely related to each other, and the language makes it easy to define them
together as methods inside the same class.

<img src="image/representing-code/rows.png" alt="The table split into rows for each class." />

This makes it easy to extend the table by adding new rows. Simply define a new
class. No existing code has to be touched. But imagine if you want to add a new
*operation* -- a new column. In Java, that means cracking open each of those
existing classes and adding a method to it.

Functional paradigm languages in the <span name="ml">ML</span> family flip that
around. There, you don't have classes with methods. Types and functions are
totally distinct. To implement an operation for a number of different types, you
define a single function. In the body of that function, you use *pattern
matching* -- sort of a type-based switch on steroids -- to implement the
operation for each type all in one place.

<aside name="ml">

ML, short for "metalanguage" was created by Robin Milner and friends and forms
one of the main branches in the great programming language family tree. Its
children include SML, Caml, OCaml, Haskell, and F#. Even Scala, Rust, and Swift
bear a strong resemblance.

Much like Lisp, it is one of those languages that is so full of good ideas that
language designers today are still rediscovering them over forty years later.

</aside>

This makes it trivial to add new operations -- simply define another function
that pattern matches on all of the types.

<img src="image/representing-code/columns.png" alt="The table split into columns for each function." />

But, conversely, adding a new type is hard. You have to go back and add a new
case to all of the pattern matches in all of the existing functions.

Each style has a certain "grain" to it. That's what the paradigm name literally
says -- an object-oriented language wants you to *orient* your code along the
rows of types. A functional language instead encourages you to lump each
column's worth of code together into a *function*.

A bunch of smart language nerds noticed that neither style made it easy to add
*both* rows and columns to the <span name="multi">table</span>. They called this
difficulty the "expression problem" because -- like we are now -- they first ran
into it when they were trying to figure out the best way to model expression
syntax tree nodes in a compiler.

<aside name="multi">

Languages with *multimethods*, like Common Lisp's CLOS, Dylan, and Julia do
support adding both new types and operations easily. What they typically
sacrifice is either static type checking, or separate compilation.

</aside>

People have thrown all sorts of language features, design patterns, and
programming tricks to try to knock that problem down but no perfect language has
finished it off yet. In the meantime, the best we can do is try to pick a
language whose orientation matches the natural architectural seams in the
program we're writing.

Object-orientation works fine for many parts of our interpreter, but these tree
classes rub against the grain of Java. Fortunately, there's a design pattern we
can bring to bear on it.

### The Visitor pattern

The **Visitor pattern** is the most widely misunderstood pattern in all of
*Design Patterns*, which is really saying something when you look at the
software architecture excesses of the past couple of decades.

The trouble starts with terminology. The pattern isn't about "visiting", and the
"accept" method in it doesn't conjure up any helpful imagery either. Many think
the pattern has to do with traversing trees, which isn't the case at all. We
*are* going to use it on a set of classes that are tree-like, but that's a
coincidence. As you'll see, the pattern works as well on a single object.

The Visitor pattern is really about approximating the functional style within an
OOP language. It lets us add new columns to that table easily. We can define all
of the behavior for a new operation on a set of types in one place, without
having to touch the types themselves. It does this the same way we solve almost
every problem in computer science: by adding a layer of indirection.

Before we apply it to our auto-generated Expr classes, let's walk through a
simpler example. Say we have two kinds of pastries: <span
name="beignet">beignets</span> and crullers.

<aside name="beignet">

A beignet (pronounced "ben-yay", with equal emphasis on both syllables) is a
deep-fried pastry in the same family as doughnuts. When the French colonized
North America in the 1700s, they brought beignets with them. Today, in the US,
they are most strongly associated with the cuisine of New Orleans.

My preferred way to consume them is fresh out of the fryer at Café du Monde,
piled high in powdered sugar, and washed down with a cup of café au lait while I
watch tourists staggering around trying to shake off their hangover from the
previous night's revelry.

</aside>

^code pastries (no location)

We want to be able to define new pastry operations -- cooking them, eating them,
decorating them, etc. -- without having to add a new method to each class every
time. Here's how we do it. First, we define a separate interface.

^code pastry-visitor (no location)

<aside name="overload">

In *Design Patterns*, both of these methods are confusingly named `visit()`, and
they rely on overloading to distinguish them. This leads some readers to think
that the correct visit method is chosen *at runtime* based on its parameter
type. That isn't the case. Unlike over*riding*, over*loading* is statically
dispatched at compile time.

Using distinct names for each method makes the dispatch more obvious, and also
shows you how to apply this pattern in languages that don't support overloading.

</aside>

Each operation that can be performed on pastries is a new class that implements
that interface. It has a concrete method for each type of pastry. That keeps the
code for the operation on both types all nestled snugly together in one class.

Given some pastry, how do we route it to the correct method on the visitor based
on its type? Polymorphism to the rescue! We add this method to Pastry:

^code pastry-accept (1 before, 1 after, no location)

Each subclass implements it.

^code beignet-accept (1 before, 1 after, no location)

And:

^code cruller-accept (1 before, 1 after, no location)

To perform an operation on a pastry, we call its `accept()` method and pass in
the visitor for the operation we want to execute. The pastry -- the specific
subclass's overriding implementation of `accept()` -- turns around and calls the
appropriate visit method on the visitor and passes *itself* to it.

That's the heart of the trick right there. It lets us use polymorphic dispatch
on the *pastry* classes to select the appropriate method on the *visitor* class.
In the table, each pastry class is a row, but if you look at all of the methods
for a single visitor, they form a *column*.

<img src="image/representing-code/visitor.png" alt="Now all of the cells for one operation are part of the same class, the visitor." />

We added one `accept()` method to each class, and we can use it for as many
visitors as we want without ever having to touch the pastry classes again. It's
a clever pattern.

### Visitors for expressions

OK, let's weave it into our expression classes. We'll also <span
name="context">refine</span> the pattern a little. In the pastry example, the
visit and `accept()` methods don't return anything. In practice, visitors often
want to define operations that produce values. But what return type should
`accept()` have? We can't assume every visitor class wants to produce the same
type, so we'll use generics to let each implementation fill in a return type.

<aside name="context">

Another common refinement is an additional "context" parameter that is passed to
the visit methods and then sent back through as a parameter to `accept()`. That
lets operations take an additional parameter. The visitors we'll define in the
book don't need that, so I omitted it.

</aside>

First, we define the visitor interface. Again, we nest it inside the base class
so that we can keep everything in one file.

^code call-define-visitor (2 before, 1 after)

That function generates the visitor interface.

^code define-visitor

Here, we iterate through all of the subclasses and declare a visit method for
each one. When we define new expression types later, this will automatically
include them.

Inside the base class, we define the abstract `accept()` method.

^code base-accept-method (2 before, 1 after)

Finally, each subclass implements that and calls the right visit method for its
own type.

^code accept-method (1 before, 2 after)

There we go. Now we can define operations on expressions without having to muck
with the classes or our generator script. Compile and run this generator script
to output an updated "Expr.java" file. It contains a generated Visitor
interface and a set of expression node classes that support the Visitor pattern
using it.

Before we end this rambling chapter, let's implement that Visitor interface and
see the pattern in action.

## A (Not Very) Pretty Printer

When we debug our parser and interpreter, it's often useful to look at a parsed
syntax tree and make sure it has the structure we expect. We could inspect it in
the debugger, but that can be a chore.

Instead, we'd like some code that, given a syntax tree, produces an unambiguous
string representation of it. Converting a tree to a string is sort of the
opposite of a parser, and is often called "pretty printing" when the goal is to
produce a string of text that is valid syntax in the source language.

That's not our goal here. We want the string to very explicitly show the nesting
structure of the tree. A printer that returned `1 + 2 * 3` isn't super helpful
if what we're trying to debug is whether operator precedence is handled
correctly. We want to know if the `+` or `*` is at the top of the tree.

To that end, the string representation we produce isn't going to be Lox syntax.
Instead, it will look a lot like, well, Lisp. Each expression is explicitly
parenthesized, and all of its subexpressions and tokens are contained in that.

Given a syntax tree like:

<img src="image/representing-code/expression.png" alt="An example syntax tree." />

It produces:

```text
(* (- 123) (group 45.67))
```

Not exactly "pretty", but it does show the nesting and grouping explicitly. To
implement this, we define a new class.

^code ast-printer

As you can see, it implements the visitor interface. That means we need visit
methods for each of the expression types we have so far.

^code visit-methods (2 before, 1 after)

Literal expressions are easy -- they convert the value to a string with a little
check to handle Java's `null` standing in for Lox's `nil`. The other expressions
have subexpressions, so they use this `parenthesize()` helper method:

^code print-utilities

It takes a name and a list of subexpressions and wraps them all up in
parentheses, yielding a string like:

```text
(+ 1 2)
```

Note that it calls `accept()` on each subexpression and passes in itself. This
is the <span name="tree">recursive</span> step that lets us print an entire
tree.

<aside name="tree">

This recursion is also why people think the Visitor pattern itself has to do
with trees.

</aside>

We don't have a parser yet, so it's hard to see this in action. For now, we'll
hack together a little `main()` method that manually instantiates a tree and
prints it.

^code printer-main

If we did everything right, it prints:

```text
(* (- 123) (group 45.67))
```

You can go ahead and delete this method. We won't need it. Also, as we add new
syntax tree types, I won't bother showing the necessary visit methods for them
in AstPrinter. If you want to (and you want the Java compiler to not yell at
you), go ahead and add them yourself. It will come in handy in the next chapter
when we start parsing Lox code into syntax trees. Or, if you don't care to
maintain AstPrinter, feel free to delete it. We won't need it again.

<div class="challenges">

## Challenges

1.  Earlier, I said that the `|`, `*`, and `+` forms we added to our grammar
    metasyntax were just syntactic sugar. Take this grammar:

    ```ebnf
    expr → expr ( "(" ( expr ( "," expr )* )? ")" | "." IDENTIFIER )+
         | IDENTIFIER
         | NUMBER
    ```

    Produce a grammar that matches the same language but does not use any of
    that notational sugar.

    *Bonus:* What kind of expression does this bit of grammar encode?

1.  The Visitor pattern lets you emulate the functional style in an
    object-oriented language. Devise a complementary pattern for a functional
    language. It should let you bundle all of the operations on one type
    together and let you define new types easily.

    (SML or Haskell would be ideal for this exercise, but Scheme or another Lisp
    works as well.)

1.  In [reverse Polish notation][rpn] (RPN), the operands to an arithmetic
    operator are both placed before the operator, so `1 + 2` becomes `1 2 +`.
    Evaluation proceeds from left to right. Numbers are pushed onto an implicit
    stack. An arithmetic operator pops the top two numbers, performs the
    operation, and pushes the result. Thus, this:

    ```lox
    (1 + 2) * (4 - 3)
    ```

    in RPN becomes:

    ```lox
    1 2 + 4 3 - *
    ```

    Define a visitor class for our syntax tree classes that takes an expression,
    converts it to RPN, and returns the resulting string.

[rpn]: https://en.wikipedia.org/wiki/Reverse_Polish_notation

</div>
