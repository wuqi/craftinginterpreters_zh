> 对于森林中的居民来说来说，几乎每一种树都有它的声音和特点。
> <cite>Thomas Hardy, <em>Under the Greenwood Tree</em></cite>

在[上一章][scanning]中，我们以字符串形式接收原始源代码，并将其转换为一个稍高层次的表示：一系列的词法标记。我们将在[下一章][parsing]中编写的解析器，将这些词法标记再次转化为更丰富、更复杂的表示形式。

[scanning]: scanning.html
[parsing]: parsing-expressions.html

在我们能够输出这种表示法之前，我们需要定义它。这就是本章的主题。在这一过程中，我们将围绕形式化语法<span name="boring">进行</span>一些理论讲解，感受函数式编程和面向对象编程之间的区别，讨论几种设计模式，并进行一些元编程。


<aside name="boring">

我非常担心这是本书中最无聊的一章，所以我不断地把更多有趣的想法塞进去，直到空间不足了。

</aside>

在我们做所有这些之前，让我们关注一下主要目标——代码的表示形式。它应该易于解析器生成，也易于解释器使用。如果您还没有编写过解析器或解释器，那么这样的需求描述并不能很好地描述问题。也许你的直觉能帮上忙。当你扮演*人工*翻译的角色时，你的大脑在做什么？你如何心算这样一个算术表达式:

```lox
1 + 2 * 3 - 4
```

因为你了解运算顺序--  像[Please Excuse My Dear Aunt Sally][sally]的<span name="PEMDAS">口诀</span>中说的--你知道乘法要在加法或减法之前计算。有一种方法是用一棵树来说明这种优先顺序。叶子结点是数字，内部结点是运算符，每个运算符都对应一个分支。

[sally]: https://en.wikipedia.org/wiki/Order_of_operations#Mnemonics

<aside name="PEMDAS">

译注: 美国记运算的顺序的口诀：括号（Parenthesis）、指数(Exponent)、乘除（multiplication and division）、加减（addition and subtraction），编成了Please Excuse My Dear Aunt Sally （请原谅我亲爱的阿姨莎莉）。

</aside>

为了计算一个算术节点，你需要知道其子树的数值，所以你必须先计算出这些子树。这意味着你要从叶节点向根部遍历--即*后序遍历*。

<span name="tree-steps"></span>

<img src="image/representing-code/tree-evaluate.png" alt="Evaluating the tree from the bottom up." />

<aside name="tree-steps">

A. 从整个树开始，首先计算最底部的操作, `2 * 3`.

B. 然后我们做加法 `+` .

C. 继续, 做减法 `-`.

D. 得到最终结果！

</aside>

如果我给你一个算术表达式，你可以很容易地画出这样的树。给定一棵树，你可以毫不费力地计算它。因此，直观来看，我们代码的可以表示为一棵与语言的语法结构(运算符嵌套)相匹配的<span name="only">树</span>。

<aside name="only">

这并不是说树是我们代码的*唯一*的表示方法。在[Part III][]中，我们将生成字节码，这是另一种不太友好但更接近机器的表示方法。

[part iii]: a-bytecode-virtual-machine.html

</aside>

那么我们需要更精确地了解语法是什么。就像上一章中的词法语法一样，围绕着句法语法有大量的理论。我们要比之前处理扫描时投入更多精力去研究这个理论，因为它在整个解释器的很多地方都是一个有用的工具。我们从[Chomsky hierarchy][]乔姆斯基层次结构的表层结构开始...

[chomsky hierarchy]: https://en.wikipedia.org/wiki/Chomsky_hierarchy

## 上下文无关文法

在上一章中，我们用来定义词法语法的形式体系——即字符如何组合成词法标记的规则——被称为*正则语言*。这对我们的扫描器来说很完美，因为它输出的是死板的词法标记序列。但正则语言还不够强大，无法处理可以任意深度嵌套的表达式。

我们需要一把更大的锤子，这就是 **上下文无关文法** (**CFG**)。它是 **[形式语法][]** (formal grammars)工具箱中第二重的工具。个形式语法由一组原子片段组成，称之为“字母表”(alphabet)。然后，它定义一组(通常是无限的)语法"中"的“字符串”。每个字符串都是字母表中的“字母”序列。

[形式语法]: https://en.wikipedia.org/wiki/Formal_grammar

我之所以使用这些引号，是因为从词法到句法的语法时，这些术语会变得有点混乱。在我们的扫描器语法中，字母表由单个字符组成，字符串是有效的词组--粗略的说，就是 "单词"。在我们现在讨论的句法语法中，我们处于不同的粒度水平。现在，字母表中的每个 "字母" 都是一个完整的词法标记，而 "字符串 "是一个 *词法标记* 的序列--一个完整的表达式。

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
  <td>词法标记</td>
</tr>
<tr>
  <td> &ldquo;字符串&rdquo; 是<span class="ellipse">&thinsp;.&thinsp;.&thinsp;.</span></td>
  <td>&rarr;&ensp;</td>
  <td>词素或词法标记</td>
  <td>表达式</td>
</tr>
<tr>
  <td>由什么实现<span class="ellipse">&thinsp;.&thinsp;.&thinsp;.</span></td>
  <td>&rarr;&ensp;</td>
  <td>扫描器</td>
  <td>解析器</td>
</tr>
</tbody>
</table>

形式语法的工作是指定哪些字符串是有效的，哪些是无效的。如果我们为英语句子定义一个语法，"鸡蛋是美味的早餐 "是正确的语法，但 "美味的早餐是鸡蛋 "则不是。

### 语法规则

我们如何描述一个包含无限个有效字符串的语法？显然，我们不能把他们全部列出来。我们创建了一个有限的规则集作为替代，你可以把它想象成一个二选一的游戏。

如果从规则入手，你可以用它们来 *生成* 语法中的字符串。以这种方式创建的字符串被称为 **派生式**（推导式） ，因为每个字符串都是从语法的规则中派生（推导）出来的。在游戏的每一步中，你都要选择一条规则，然后按照它告诉你的去做。围绕形式化语法的大部分术语都从这个方向来的。规则被称为 **产生式** （生成式），因为它们在 *语法* 中产生字符串。

上下文无关文法中的每个产生式都有一个 **头** -- 包含<span name="name">名称</span>--和 **主体**，用于描述它生成的内容。在其纯粹的形式中，主体只是一个符号的列表。符号有两种可选口味:

<aside name="name">

将头部限制在一个符号上是上下文无关文法的一个决定性特征。更强大的形式主义，如 **[无限制文法][]** ，允许在头和主体中都有一系列符号。

[无限制文法]: https://en.wikipedia.org/wiki/Unrestricted_grammar

</aside>

*   **终结符** 是语法字母表中的一个字母。你可以把它看成是一个字面值。在我们定义的句法语法中，终结符是独立的词素--来自扫描器的词法标记 ，如 `if` 或 `1234` 。

    这些被称为 "终结符"，取 "终点" 之意，因为它们不会导致游戏中的任何进一步 "行动"。你只是简单地产生了那一个符号。

*   **非终结符** 是对语法中另一条规则的命名引用。它意味着 "执行该规则并在这里插入它产生的任何内容"。这样一来，语法就构成了。

还有最后一项改进：你可以有多个同名的规则。当你到达一个具有该名称的非终结符时，你可以为它选择任何一条规则，以你的想法为准。

为了使之具体化，我们需要一种<span name="turtles">方法</span>来写下这些产生式规则。人们一直在试图将语法具体化，这可以追溯到古印度语法学家波你尼撰写的《声明论》(亦称《八章书》) ，他在几千年前就将梵文语法编成了法典。直到John Backus和其公司需要指定ALGOL 58语言的记法，并提出了[**Backus-Naur form**][bnf]（ **BNF** 巴科斯范式），才有了很大的进展。从那时起，几乎每个人都使用BNF的某种变形，并根据自己的口味进行调整。

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

我们可以使用这个语法来生成随机早餐。我们来玩一轮，看看它是如何工作的。按照老规矩，游戏从语法中的第一条规则开始，在这里是 `breakfast`。有三个产生式，我们随机选择第一个。我们得到的字符串如下所示:

```text
protein "with" breakfast "on the side"
```

我们需要扩展第一个非终结符，即 `protein` ，所以我们为它选择一个产生式。我们来选一个:

```ebnf
protein → cooked "eggs" ;
```

接下来，我们需要为 `cooked` 选一个产生式，所以我们选择  `"poached"`。那是一个终结符，所以我们把它加进去。现在我们的字符串是这样的:

```text
"poached" "eggs" "with" breakfast "on the side"
```

下一个非终结符又是 `breakfast` 。我们选择的第一个 `breakfast` 产生式递归引用了 `breakfast` 规则。语法中的递归是一个很好的标志，说明被定义的语言是上下文无关的，而不是正则的。特别是，递归非终结符 <span name="nest">两边</span>都有产生式的递归，意味着这种语言不是正则的。

<aside name="nest">

想象一下，我们在这里把 `breakfast` 规则递归扩展了好几次，比如"bacon with bacon with bacon with..." 。为了正确地完成这个字符串，我们需要在结尾处添加*同等* 数量的 "on the side" 。跟踪所需尾部的数量超出了正则语法的能力。正则语法可以表达重复，但它们不能统计有多少次重复，而这对于确保字符串有相同数量的 `with` 和 `on the side` 部分是必要的。

</aside>

我们可以不断选择第一种产生式作为 `breakfast` ，产生各种形式的早餐，如 "bacon with sausage with scrambled eggs with bacon..."。但我们不会这样做。这一次，我们将选择 `bread`。`bread` 有三条规则，每条都只包含一个终结符。我们将选择 "English muffin"。

这样，字符串中的每个非终结符都被展开，直到最后只包含终结符，我们只剩下:

<img src="image/representing-code/breakfast.png" alt='"Playing" the grammar to generate a string.' />

再加上一些火腿和荷兰酱，你就有了班尼迪克蛋。

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

我希望不是太糟糕。如果您习惯于在文本编辑器中使用grep或[正则表达式][regex]，那么大多数标点符号应该很熟悉。主要区别在于这里的符号表示整个标记，而不是单个字符。

[regex]: https://en.wikipedia.org/wiki/Regular_expression#Standards

在本书的其余部分，我们将使用这个记法来精确描述Lox的语法。当你研究编程语言时，你会发现上下文无关文法（使用这个或 [EBNF][] 或其他记法）可以帮助你将你的非正式语法设计想法具体化。它们也是与其他语言黑客交流语法的一个方便的媒介。

[ebnf]: https://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_form

我们为Lox定义的规则和产生式也是我们要实现的树状数据结构的指南，以表示内存中的代码。在做到这一点之前，我们需要一个实际的Lox语法，或者至少有足够的语法让我们开始工作。

### Lox表达式的语法

在上一章中，我们一气呵成地完成了Lox所有的词法语法。包括每个关键词和标点符号。但句法语法规模更大，在我们真正启动和运行我们的解释器之前，从头到尾研究一遍实在是一件无聊的事情。

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

这里有一点额外的<span name="play">元语法</span>。除了对匹配精确词素的终结符使用加引号的字符串外，我们还对表示单一词素的终结符使用大写字母，这些表示的文本表示可能有所不同。 `NUMBER` 是任意数字字面量，`STRING` 是任意字符串字面量。稍后，我们将对 `IDENTIFIER` 做同样的处理。

这个语法实际上是有歧义的，我们在解析它的时候会看到这一点。但是现在已经够用了。

<aside name="play">

如果你愿意，可以试着用这个语法生成一些表达式，就像我们之前用早餐语法做的那样。你觉得结果表达式看起来对吗？你能让它产生像 `1 + / 3` 这样的错误吗？

</aside>

## 实现语法树

最后，我们可以写一些代码了。那个小小的表达式语法就是我们的骨架。由于语法是递归的--注意`grouping`, `unary`和`binary`都是指回 `expression` 的--我们的数据结构将形成一棵树。由于这个结构代表了我们语言的语法，所以它被称为<span name="ast">**语法树**</span>。

<aside name="ast">

特别地，我们定义了一个 **抽象语法树** ( **AST** )。在一个 **语法分析树** 中，每一个语法产生式都成为树中的一个节点。AST删除后面阶段不需要的产生式。

</aside>

我们的扫描器使用单个 Token 类来表示所有的类型的词素。为了区分不同的种类- -想想数字 `123`  和字符串 `"123"` - -我们加了一个简单的 TokenType 枚举。句法树并不是那么<span name="token-data">同质化</span>。一元表达式只有一个操作数，二元表达式有两个操作数，字面量没有操作数。

我们 *可以* 把这些都混在一起，整合成一个具有任意子类列表的单个Expression类中。有些编译器就是这样做的。但是我喜欢充分的利用Java的类型系统。因此我们将为表达式定义一个基类。然后，对于每一种表达式- -每个表达式( `expression` )下的产生式- -我们创建一个子类，该子类具有该规则所特有的非终结符字段。这样，如果我们试图访问一元表达式的第二个操作数，我们就会得到一个编译错误。

<aside name="token-data">

Tokens 也不是完全同质的。字面值的 Token 保存了值，但其他类型的 lexeme 不需要这种状态。我见过一些扫描器对字面值和其他种类的 lexeme 使用不同的类，但我想我应该让事情更简单。

</aside>

大概是这样的:

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

我在代码中避免使用缩写，因为它们会让不知道缩写代表什么的读者犯错误。但是在我所观察的编译器中，"Expr" 和 "Stmt" 是如此普遍，以至于我现在也可以开始让你习惯它们了。

</aside>

Expr 是所有表达式类都继承自的基类。正如你在 `Binary` 中所看到的，子类都嵌套在它里面。这在技术上没有必要，但它让我们把所有的类都压缩到一个 Java 文件中。

### 不面向对象的类

您会注意到，表达式类与Token类非常相似，其中没有任何方法。这是一个愚蠢的结构。巧妙的类型封装，但只是一包数据。这在Java这样的面向对象语言中感觉很奇怪。类内不应该 *做一些事情* 吗？

问题是这些树相关类不属于任何单个领域。既然树是在语法分析时创建的，那么它们应该有语法分析方法吗？或者提供解释相关的方法，因为这是他们被消费的地方？树横跨这些领土之间的边界，这意味着它们实际上都不属于 *任何一方*。

事实上，这些类型的存在是为了使解析器和解释器能够进行通信。这使它适合于简单的没有关联行为的数据类型。这种风格在函数式语言中非常自然，比如Lisp和ML，在这些语言中，所有的数据都与行为都是分离的，但是在Java中就很奇怪。

函数式编程的狂热爱好者现在都跳起来惊呼“看！面向对象语言不适合解释器！”我不会这么说。您可能还记得，扫描器本身非常适合面向对象。它包含所有的可变状态来跟踪它在源代码中的位置，有一组定义良好的公共方法，还有一些私有帮助器。

我的感觉是，解释器的每个阶段或部分在面向对象的风格中工作得很好。只不过在它们之间流动的数据结构剥离了行为。

### 节点树元编程

Java可以表达无行为类，但它并不擅长这个。用11行代码在一个对象中填充3个字段是相当乏味的，当我们全部完成后，我们将有21个这样的类。

我不想浪费你的时间或我的墨水把这些都写下来。想想，每个子类的本质是什么？一个名称和一个类型字段列表而已。我们是聪明的语言黑客，对吗？让我们把它 <span name="automate">自动化</span>。


<aside name="automate">

想象一下，当你读到这句话时，我正在跳一种尴尬的机器人舞蹈。"AU-TO-MATE"。

</aside>

我们将编写一个<span name="python">脚本</span>来完成这些工作，而不是单调乏味地手写每个类定义、字段声明、构造函数和初始化器。它有每个树类型的描述——它的名称和字段——并打印出定义具有该名称和状态的类所需的Java代码。

该脚本是一个微型Java命令行应用程序，它生成一个名为“ Expr.java”的文件：

<aside name="python">

我从Jython和IronPython的创始人Jim Hugunin那里得到了编写语法树类脚本的想法。

真正的脚本语言比Java更适合这种情况，但我尽量不让您折腾太多的语言。

</aside>

^code generate-ast

注意，这个文件在另一个包中，是 `.tool` 而不是 `.lox` 。这个脚本并不是解释器本身的一部分，它是一个工具，*我们*这些编写解释器的黑客，通过运行该脚本来生成语法树类。完成后，我们把“Expr.java”与实现中的其它文件进行相同的处理。我们只是自动化了文件的生成方式。

为了生成这些类，还需要对每种类型及其字段进行一些描述。

^code call-define-ast (1 before, 1 after)

为了简洁起见，我将表达式类型的描述塞进了字符串中。每一项都包括类的名称，后跟 `:` 和逗号分隔的字段列表。每个字段都有一个类型和一个名称。


`defineAst()`需要做的第一件事是输出基类Expr。

^code define-ast

我们调用这个函数时，`baseName` 是“Expr”，它既是类的名称，也是它输出的文件的名称。我们将它作为参数传递，而不是对名称进行硬编码，因为稍后我们将为语句添加一个单独的类族。

在基类内部，我们定义每个子类。

^code nested-classes (2 before, 1 after)

<aside name="robust">

这不是世界上最优雅的字符串操作代码，但也不错了。它只在我们给它的类定义集上运行。鲁棒性不是优先考虑的问题。

</aside>

这段代码依次调用：

^code define-type

好了。所有的Java模板都完成了。它在类体中声明了每个字段。它为类定义了一个构造函数，为每个字段提供参数，并在主体中初始化了它们。

现在编译并运行这个Java程序，它会<span name="longer">生成</span>一个新的 &ldquo.java" 文件，其中包含几十行代码。这个文件将变得更长。

<aside name="longer">

[Appendix II][]中包含了我们完成了jlox的实现并定义了其所有语法树节点后，这个脚本所生成的代码

[appendix ii]: appendix-ii.html

</aside>

## 处理树结构

先想象一下吧。尽管我们还没有到那一步，但请考虑一下解释器将如何处理语法树。Lox中的每种表达式在运行时的行为都不一样。这意味着解释器需要选择不同的代码块来处理每种表达式类型。对于词法标记，我们可以简单地根据`TokenType`进行转换。但是我们并没有为语法树设置一个 "type "枚举，只是为每个语法树单独设置一个Java类。

我们可以编写一长串类型测试：

```java
if (expr instanceof Expr.Binary) {
  // ...
} else if (expr instanceof Expr.Grouping) {
  // ...
} else // ...
```

但所有这些顺序类型测试都很慢。类型名称按字母顺序排列在后面的表达式，执行起来会花费更多的时间，因为在找到正确的类型之前，它们会遇到更多的 `if` 情况。我认为这种解决方案不够优雅。

我们有一个类族，我们需要将一组行为与每个类关联起来。在 Java 这样的面向对象语言中，最自然的解决方案是将这些行为放入类本身的方法中。我们可以在Expr上添加一个抽象的<span name="interpreter-pattern">`interpret()`</span>方法，然后每个子类都要实现这个方法来解释自己。

<aside name="interpreter-pattern">

这就是Erich Gamma等人在《设计模式:可重用的面向对象软件的元素》一书中所谓的解释器模式(["Interpreter pattern"][interp])。

[interp]: https://en.wikipedia.org/wiki/Interpreter_pattern

</aside>

这对于小型项目来说还行，但它的扩展性很差。就像我之前提到的，这些树类跨越了几个领域。至少，解析器和解释器都会扰乱它们。[稍后您将看到][resolution]，我们需要对它们进行名称解析。如果我们的语言是静态类型的，我们还需要做类型检查。

[resolution]: resolving-and-binding.html

如果我们为每一个操作的表达式类中添加实例方法，就会将一堆不同的领域混在一起。这违反了[关注点分离原则][separation of concerns]，并会产生难以维护的代码。

[separation of concerns]: https://en.wikipedia.org/wiki/Separation_of_concerns

### 表达式问题

这个问题比起初看起来更基础。我们有一些类型，和一些高级操作，比如“解释”。对于每一对类型和操作，我们都需要一个特定的实现。画一个表:

<img src="image/representing-code/table.png" alt="A table where rows are labeled with expression classes, and columns are function names." />

行是类型，列是操作。每个单元格表示在该类型上实现该操作的唯一代码段。

像Java这样的面向对象的语言，假定一行中的所有代码都自然地挂在一起。它认为你对一个类型所做的所有事情都可能是相互关联的，而使用这类语言可以很容易将它们一起定义为同一个类里面的方法。


<img src="image/representing-code/rows.png" alt="The table split into rows for each class." />

这种情况下，很容易向表中加入新行来扩展列表，简单地定义一个新类即可，不需要修改现有的代码。但是，想象一下，如果你要添加一个新操作（新的一列）。在Java中，这意味着要拆开已有的那些类并向其中添加方法。

<span name="ml">ML</span>家族中的函数式范型正好相反。在这些语言中，没有带方法的类，类型和函数是完全独立的。要为许多不同类型实现一个操作，只需定义一个函数。在该函数体中，您可以使用*模式匹配*（某种基于类型的switch操作）在同一个函数中实现每个类型对应的操作。

<aside name="ml">

ML，是元语言(metalanguage)的简称，它是由Robin Milner和他的朋友们创建的，是伟大的编程语言家族的主要分支之一。它的包括SML、Caml、OCaml、Haskell和F#。甚至Scala、Rust和Swift都有很强的相关性。

就像Lisp一样，它也是一种充满了好想法的语言，即使在40多年后的今天，语言设计者仍然在重新挖掘它们。

</aside>

这使得添加新操作非常简单——只需定义另一个与所有类型模式匹配的的函数即可。

<img src="image/representing-code/columns.png" alt="The table split into columns for each function." />

但是，相对的，很难添加新类型。您必须回头向已有函数中的所有模式匹配添加一个新的case。

每种风格都有特定的 "纹理"。这就是范式名称的字面意思——面向对象的语言希望你按照类型的行来*组织*你的代码。而函数式语言则鼓励你把每一列的代码都归纳为一个*函数*。

一群聪明的语言迷注意到，这两种风格都不容易在<span name="multi">表格</span>中*同时*添加行和列。他们把这个难题称为 "表达式问题"，因为--就像我们现在这样--他们是在试图找出在编译器中建模表达式语法树节点的最佳方法时，第一次遇到了这个问题。

<aside name="multi">

诸如Common Lisp的CLOS，Dylan和Julia这样的支持多范式的语言都能轻松添加新类型和操作。他们通常牺牲的不是静态类型检查，就是单独编译。

</aside>


人们已经抛出了各种各样的语言特性、设计模式和编程技巧，试图解决这个问题，但还没有一种完美的语言能够解决它。与此同时，我们所能做的就是尽量选择一种与我们正在编写的程序的自然架构相匹配的语言。

面向对象在我们的解释器的许多部分都可以正常工作，但是这些树类与Java的本质背道而驰。 幸运的是，我们可以采用一种设计模式来解决这个问题。

### 访问者模式


**访问者模式**是所有*设计模式*中最容易被误解的模式，当您回顾过去几十年软件架构被滥用的状况时，会发现确实如此。

问题出在术语上。这个模式不是关于“visiting（访问）”，它的 “accept”方法也没有让人产生任何有用的联想。许多人认为这种模式与遍历树有关，但情况并非如此。我们*将要*在一组类似树的类上使用它，但这只是一个巧合。如您所见，该模式在单个对象上也可以正常使用。

访问者模式实际上与OOP语言中的函数式风格相似。它让我们可以轻松地向表中添加新列。我们可以在一个地方定义针对一组类型的新操作的所有行为，而不必触及类型本身。这与我们解决计算机科学中几乎所有问题的方式相同：添加中间层。

Before we apply it to our auto-generated Expr classes, let's walk through a
simpler example. Say we have two kinds of pastries: <span
name="beignet">beignets</span> and crullers.

在将其应用到自动生成的 Expr 类之前，让我们先看一个更简单的例子。比方说我们有两种点心:<span name="beignet">beignet(贝奈特饼)</span>和 cruller(油酥卷)。

<aside name="beignet">

贝奈特饼（发音为 "ben-yay"，两个音节并重）是一种油炸糕点，与甜甜圈属于同一家族。当法国人在1700年代殖民北美时，他们带来了贝奈特饼。今天，在美国，它们与新奥尔良的美食联系最紧密。

我最喜欢的吃法是Café du Monde的油锅里新鲜出炉的贝奈特饼，堆上厚厚的糖粉，然后用一杯白咖啡冲下去，我看着游客们摇摇晃晃地走来走去，试图摆脱前一晚狂欢的宿醉。

</aside>

^code pastries (no location)

我们希望能够定义新的糕点操作（烹饪，食用，装饰等），而不必每次都向每个类添加新方法。我们是这样做的。首先，我们定义一个单独的接口。

^code pastry-visitor (no location)

<aside name="overload">

在*设计模式*中，这两种方法的名字都叫 `visit()` ，很容易混淆，需要依赖重载来区分不同方法。这也导致一些读者认为，正确的 visit 方法是在运行时根据其参数类型选择的。事实并非如此。与重载不同，重载是在编译时静态分配的。

为每个方法使用不同的名称使分派更加明显，同时还向您展示了如何在不支持重载的语言中应用此模式。

</aside>

可以对糕点执行的每个操作都是实现该接口的新类。 它对每种类型的糕点都有具体的方法。 这样一来，针对两种类型的操作代码都紧密地嵌套在一个类中。

对于给定的糕点，我们如何根据其类型将其路由定位到访问者的正确方法上？多态性拯救了我们！我们在糕点类中添加这个方法：

^code pastry-accept (1 before, 1 after, no location)

每个子类都需要实现该方法：

^code beignet-accept (1 before, 1 after, no location)

以及：

^code cruller-accept (1 before, 1 after, no location)

要对糕点执行一个操作，我们就调用它的 `accept()` 方法，并将我们要执行的操作vistor 作为参数传入该方法。糕点类——特定子类对 `accept()` 的重写实现——会反过来，在 visitor 上调用合适的 visit 方法，并将*自身*作为参数传入。

这就是魔术的核心。它允许我们在*pastry*类上使用多态派遣来选择*visitor*类上的适当方法。在表中，每个糕点类都是一行，但是如果查看单个访问者的所有方法，它们会形成一个*列*。

<img src="image/representing-code/visitor.png" alt="Now all of the cells for one operation are part of the same class, the visitor." />

我们为每个类添加了一个`accept（）`方法，我们可以根据需要将其用于任意数量的访问者，而无需再次修改*pastry*类。 这是一个聪明的模式。

### 表达式访问者

好了，让我们将它编入表达式类中。我们还要<span name="context">完善</span>一下这个模式。在糕点的例子中，visit和`accept()`方法没有返回任何东西。在实践中，访问者通常希望定义能够产生值的操作。但`accept()`应该具有什么返回类型呢？我们不能假设每个访问者类都想产生相同的类型，所以我们将使用泛型来让每个实现类自行填充一个返回类型。

<aside name="context">

另一个常见的改进是一个附加的“上下文”参数，它传递给访问方法，然后作为参数发送回`accept()` 。这允许操作使用附加参数。我们将在书中定义的访问者不需要这个，所以我省略了它。

</aside>

首先，我们定义访问者接口。同样，我们把它嵌套在基类中，以便将所有的内容都放在一个文件中。

^code call-define-visitor (2 before, 1 after)

这个函数会生成visitor接口。

^code define-visitor

在这里，我们遍历所有的子类，并为每个子类声明一个 visit 方法。当我们以后定义新的表达式类型时，会自动包含这些内容。

在基类中，我们定义抽象 `accept()` 方法。

^code base-accept-method (2 before, 1 after)

最后，每个子类都实现该方法，并调用其类型对应的visit方法。

^code accept-method (1 before, 2 after)

这下好了。现在我们可以在表达式上定义操作，而且无需对类或生成器脚本进行修改。编译并运行这个生成器脚本，输出一个更新后的 "Expr.java "文件。该文件中包含一个生成的Visitor接口和一组使用该接口支持Visitor模式的表达式节点类。

在结束这杂乱的一章之前，我们先实现一下这个Visitor接口，看看这个模式的运行情况。

##  一个（不是很）漂亮的打印器

当我们调试解析器和解释器时，查看解析后的语法树并确保其与期望的结构一致通常是很有用的。我们可以在调试器中进行检查，但这可能是一件苦差事。

相反，我们需要一些代码，在给定语法树的情况下，生成一个明确的字符串表示。将语法树转换为字符串是解析器的逆向操作，当我们的目标是产生一个在源语言中语法有效的文本字符串时，通常被称为 "美化输出"（pretty printing）。

这不是我们的目标。我们希望字符串非常明确地显示树的嵌套结构。如果我们要调试的是操作符的优先级是否处理正确，那么返回`1 + 2 * 3`的打印器并没有什么用，我们想知道`+`或`*`是否在语法树的顶部。

因此，我们生成的字符串表示形式不是Lox语法。相反，它看起来很像Lisp。每个表达式都被显式地括起来，并且它的所有子表达式和词法标记都包含在其中。

给定一个语法树，如：

<img src="image/representing-code/expression.png" alt="An example syntax tree." />

输出结果为：

```text
(* (- 123) (group 45.67))
```

不是很“好看”，但是它确实明确地展示了嵌套和分组。为了实现这一点，我们定义了一个新类。

^code ast-printer

如你所见，它实现了visitor接口。这意味着我们需要为我们目前拥有的每一种表达式类型提供visit方法。

^code visit-methods (2 before, 1 after)

字面量表达式很简单——它们将值转换为一个字符串，并通过一个小检查用Java中的 `null` 代替Lox中的 `nil`。其他表达式有子表达式，所以它们要使用 `parenthesize()` 这个辅助方法：

^code print-utilities

它接收一个名称和一组子表达式作为参数，将它们全部包装在圆括号中，并生成一个如下的字符串：

```text
(+ 1 2)
```

请注意，它在每个子表达式上调用 `accept()` 并将自身传递进去。 这是<span name="tree">递归</span>步骤，可让我们打印整棵树。

<aside name="tree">

这种递归也导致人们认为访问者模式本身与树有关。

</aside>

我们还没有解析器，所以很难看到它的实际应用。现在，我们先使用一个 `main()` 方法来手动实例化一个树并打印它。

^code printer-main

如果一切无误，它就会打印出：

```text
(* (- 123) (group 45.67))
```

您可以继续，删掉这个方法，我们后面不再需要它了。另外，当我们添加新的语法树类型时，我不会费心在AstPrinter中展示它们对应的 visit 方法。如果你想这样做(并且希望 Java 编译器不会报错)，那么你可以自行添加这些方法。在下一章，当我们开始将Lox代码解析为语法树时，它将会派上用场。或者，如果你不想维护 AstPrinter ，可以随时删除它。我们不会再用上它了。


<div class="challenges">

## Challenges

1.  之前我说过，我们在语法元语法中添加的`|`、`*`、`+`等形式只是语法糖。以这个语法为例:

    ```ebnf
    expr → expr ( "(" ( expr ( "," expr )* )? ")" | "." IDENTIFIER )+
         | IDENTIFIER
         | NUMBER
    ```

    生成一个与同一语言相匹配的语法，但不要使用任何语法糖。

    *附加题:* 这段语法编码了什么样的表达式?

1.  访问者模式让你在面向对象语言中模拟函数式语言风格。为函数式语言设计一个互补模式。它应该允许您将一个类型上的所有操作捆绑在一起，并允许您轻松地定义新类型。

    (SML 或 Haskell 是这个练习的理想选择，但是 Scheme 或其它 Lisp 方言也可以)

1.  在[逆波兰表示法][rpn] (RPN) 中，算术运算符的操作数都放在运算符之前，所以`1 + 2`变成了 `1 2 +`。计算时从左到右进行，操作数被压入隐式栈。算术运算符弹出前两个数字，执行运算，并将结果推入栈中。因此,

    ```lox
    (1 + 2) * (4 - 3)
    ```

    在RPN中变为了

    ```lox
    1 2 + 4 3 - *
    ```

    为我们的语法树类定义一个visitor类，该类接受表达式，将其转换为RPN，并返回结果字符串。

[rpn]: https://en.wikipedia.org/wiki/Reverse_Polish_notation

</div>
