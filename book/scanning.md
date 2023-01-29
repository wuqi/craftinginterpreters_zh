> 大口去吃。任何值得做的事情都值得玩命去做。
>
> <cite>Robert A. Heinlein, <em>Time Enough for Love</em></cite>

任何编译器或解释器的第一步都是<span name="lexing">scanning</span>(扫描)。扫描器以一系列字符的形式接收原始源代码，并将其分组为一系列我们称之为令牌( **tokens** )的块。这些是组成语言语法的有意义的“单词”和“标点”。

<aside name="lexing">

多年来，这项任务被不同地称为 "scanning" 和 "lexing" ( "lexical
analysis" 的简称)。很久以前，当计算机像Winnebagos房车一样大，但内存比你的手表还少时，一些人使用“scanner”仅指处理从磁盘读取原始源代码字符并将其缓冲在内存中的那段代码。然后“lexing”是对角色做有用的事情的后续阶段。

如今，将源文件读入内存是很平常的事情，所以它很少是编译器中的一个独特阶段。因此，这两个术语基本上可以互换。

</aside>
扫描对我们来说也是一个很好的起点，因为这段代码并不难--几乎就是一个夸大的 `switch` 语句。它可以帮助我们在以后处理一些更有趣的材料之前进行热身。在本章结束时，我们将拥有一个全功能的、快速的扫描器，它可以处理任何字符串的Lox源代码并产生令牌，我们将在下一章中把这些令牌送入分析器。

## 解释器框架

因为这是我们真正的第一章，在我们开始实际扫描一些代码之前，我们需要勾画出我们的解释器jlox的基本形状。一切都是从Java中的一个类开始的。

^code lox-class

<aside name="64">

对于退出代码，我使用UNIX中["sysexits.h"][sysexits]标头中定义的约定。这是我能找到的最接近标准的东西。

[sysexits]: https://www.freebsd.org/cgi/man.cgi?query=sysexits&amp;apropos=0&amp;sektion=0&amp;manpath=FreeBSD+4.3-RELEASE&amp;format=html

</aside>

把它放在一个文本文件中，然后去获取你的IDE或者Makefile或者其他设置。我在这里等你准备完毕,准备好了?出发!

Lox是一种脚本语言，这意味着它直接从源代码执行。我们的解释器支持两种运行代码的方式。如果从命令行启动jlox，并给它一个文件路径，它就会读取并执行该文件。

^code run-file

如果您想与解释器进行更亲密的对话，也可以交互运行它。在没有任何参数的情况下启动jlox，它会将您置于一个提示中，您可以在那里一次输入和执行一行代码。

<aside name="repl">

交互式提示也被称为“REPL”（发音类似于“rebel”，但带有“p”）。这个名字来自Lisp，在Lisp中，实现一个函数就像围绕几个内置函数包装一个循环一样简单：

```lisp
(print (eval (read)))
```

从最嵌套的调用向外工作，您读取( **R**ead )一行输入，对其求值(**E**valuate)，打印(**P**rint)结果，然后循环(**L**oop)并再次执行。

</aside>

^code prompt

 `readLine()` 函数，顾名思义，从命令行读取用户输入的一行内容，并返回结果。要终止一个交互式命令行应用程序，你通常需要键入 Control-D。这样做会向程序发出“文件结束”(EOF)的信号。当发生这种情况时， `readLine()` 返回 `null` ，所以我们检查此情况以退出循环。

提示符和文件运行器都是这个核心功能的简单包装:

^code run

由于我们还没有写出解释器，所以它还不是很有用，但不积跬步无以至千里。现在，它打印出我们即将实现的扫描器将发出的令牌，这样我们就可以看到我们是否取得了进展。

### 错误处理

在我们进行设置时，基础设施的另一个关键部分是*错误处理*。教科书有时会忽略这一点，因为它更像是一个实际问题而不是一个正式的计算机科学问题。但是，如果你想做一个真正*可用*的语言，那么优雅地处理错误是至关重要的。

我们的语言为处理错误所提供的工具构成了其用户界面的很大一部分。当用户的代码在工作时，他们根本没有想到我们的语言--他们的头脑中全是*他们的程序*。通常只有当事情出错时，他们才会注意到我们的实现。

<span name="errors">当</span>这种情况发生时，我们有责任向用户提供他们需要的所有信息，让他们了解哪里出错了，并引导他们慢慢回到他们想去的地方。做好这一点意味着从现在开始，在我们解释器的整个实现过程中考虑错误处理。

<aside name="errors">

说了这么多，对于*这个*解释器来说，我们要建立的是非常原始的东西。我很想谈谈交互式调试器、静态分析器和其他有趣的东西，但篇幅有限。

</aside>

^code lox-error

这个 `error()` 函数和它的 `report()` 助手告诉用户在给定的行中出现了一些语法错误。这是能够声称你*有*错误报告的底线。想象一下，如果您在某个函数调用中不小心留下了一个悬空逗号，解释器会打印出来:

```text
Error: Unexpected "," somewhere in your code. Good luck finding it!
```

这不是很有帮助。我们至少需要把他们指向正确的行。更好的做法是列出开头和结尾栏，这样他们就能知道这一行的位置。比这更好的是向用户*展示*错误的那行，比如：

```text
Error: Unexpected "," in argument list.

    15 | function(first, second,);
                               ^-- Here.
```

我很想在这本书中实现类似的东西，但老实说，这是一个非常繁琐的字符串操作代码。对用户来说非常有用，但在书中阅读并不是非常有趣，而且在技术上也不是很有趣。所以我们只保留一个行号。在你们自己的翻译中，请照我说的去做，不要照我做的去做。

我们把这个错误报告函数放在主Lox类中的主要原因是因为 `hadError` 字段。它的定义如下:

^code had-error (1 before)

我们将使用它来确保我们不会试图执行有已知错误的代码。此外，它让我们像一个好的命令行公民应该做的那样，用非零退出代码退出。

^code exit-code (1 before, 1 after)

我们需要在交互循环中重置此标志。如果用户犯了错误，它不应该终止他们的整个会话。

^code reset-had-error (1 before, 1 after)

我把错误报告放在这里而不是塞进扫描器和其他可能发生错误的阶段的另一个原因是为了提醒您，将*产生*错误的代码与*报告*错误的代码分开是一个很好的工程实践。

前端的各个阶段都会检测到错误，但是知道如何将错误呈现给用户并不是他们的工作。在全功能语言实现中，可能会有多种显示错误的方式:在 `stderr` 上，在IDE 的错误窗口中，记录到文件中，等等。您不希望这些代码遍布您的扫描器和解析器。

理想情况下，我们应该有一个实际的抽象，某种<span name="reporter">"ErrorReporter"</span>接口传递给扫描器和解析器，以便我们可以交换不同的报告策略。对于我们这里的简单解释器，我没有这样做，但我至少把错误报告的代码移到了不同的类中。

<aside name="reporter">

在我第一次实现jlox的时候，就是这么干的。我最后把它移除了，因为对于这本书中的最小解释器来说，它感觉设计得过于复杂了。

</aside>

有了一些基本的错误处理，我们的应用程序框架就准备好了。一旦我们有了一个带有 `scanTokens()` 方法的 Scanner 类，我们就可以开始运行它了。在这之前，让我们更精确地了解一下什么是令牌。


## Lexemes 和 Tokens

下面是一行Lox代码:

```lox
var language = "lox";
```

这里，`var` 是声明变量( `variable` )的关键字。那个三个字符的序列“v-a-r”意味着什么。但是如果我们从 `language` 中间抽出三个字母，比如“g-u-a”，它们本身没有任何意义。 

这就是词法分析的目的。我们的工作是扫描字符列表，并把它们组合成仍然代表某些东西的最小序列。这些字符块中的每一个都被称为一个 **lexeme** 。在示例代码行中，lexemes 是:

<img src="image/scanning/lexemes.png" alt="'var', 'language', '=', 'lox', ';'" />

Lexemes 只是源代码的原始子串。然而，在将字符序列组合成词位的过程中，我们也会偶然发现一些其他有用的信息。当我们把 lexemes 和其他数据捆绑在一起时，结果就是一个令牌(token)。它包括其他有用的东西，如:

### 令牌类型

Keywords are part of the shape of the language's grammar, so the parser often
has code like, "If the next token is `while` then do..." That means the parser
wants to know not just that it has a lexeme for some identifier, but that it has
a *reserved* word, and *which* keyword it is.

The <span name="ugly">parser</span> could categorize tokens from the raw lexeme
by comparing the strings, but that's slow and kind of ugly. Instead, at the
point that we recognize a lexeme, we also remember which *kind* of lexeme it
represents. We have a different type for each keyword, operator, bit of
punctuation, and literal type.

<aside name="ugly">

After all, string comparison ends up looking at individual characters, and isn't
that the scanner's job?

</aside>

^code token-type

### Literal value

There are lexemes for literal values -- numbers and strings and the like. Since
the scanner has to walk each character in the literal to correctly identify it,
it can also convert that textual representation of a value to the living runtime
object that will be used by the interpreter later.

### Location information

Back when I was preaching the gospel about error handling, we saw that we need
to tell users *where* errors occurred. Tracking that starts here. In our simple
interpreter, we note only which line the token appears on, but more
sophisticated implementations include the column and length too.

<aside name="location">

Some token implementations store the location as two numbers: the offset from
the beginning of the source file to the beginning of the lexeme, and the length
of the lexeme. The scanner needs to know these anyway, so there's no overhead to
calculate them.

An offset can be converted to line and column positions later by looking back at
the source file and counting the preceding newlines. That sounds slow, and it
is. However, you need to do it *only when you need to actually display a line
and column to the user*. Most tokens never appear in an error message. For
those, the less time you spend calculating position information ahead of time,
the better.

</aside>

We take all of this data and wrap it in a class.

^code token-class

Now we have an object with enough structure to be useful for all of the later
phases of the interpreter.

## Regular Languages and Expressions

Now that we know what we're trying to produce, let's, well, produce it. The core
of the scanner is a loop. Starting at the first character of the source code,
the scanner figures out what lexeme the character belongs to, and consumes it
and any following characters that are part of that lexeme. When it reaches the
end of that lexeme, it emits a token.

Then it loops back and does it again, starting from the very next character in
the source code. It keeps doing that, eating characters and occasionally, uh,
excreting tokens, until it reaches the end of the input.

<span name="alligator"></span>

<img src="image/scanning/lexigator.png" alt="An alligator eating characters and, well, you don't want to know." />

<aside name="alligator">

Lexical analygator.

</aside>

The part of the loop where we look at a handful of characters to figure out
which kind of lexeme it "matches" may sound familiar. If you know regular
expressions, you might consider defining a regex for each kind of lexeme and
using those to match characters. For example, Lox has the same rules as C for
identifiers (variable names and the like). This regex matches one:

```text
[a-zA-Z_][a-zA-Z_0-9]*
```

If you did think of regular expressions, your intuition is a deep one. The rules
that determine how a particular language groups characters into lexemes are
called its <span name="theory">**lexical grammar**</span>. In Lox, as in most
programming languages, the rules of that grammar are simple enough for the
language to be classified a **[regular language][]**. That's the same "regular"
as in regular expressions.

[regular language]: https://en.wikipedia.org/wiki/Regular_language

<aside name="theory">

It pains me to gloss over the theory so much, especially when it's as
interesting as I think the [Chomsky hierarchy][] and [finite-state machines][]
are. But the honest truth is other books cover this better than I could.
[*Compilers: Principles, Techniques, and Tools*][dragon] (universally known as
"the dragon book") is the canonical reference.

[chomsky hierarchy]: https://en.wikipedia.org/wiki/Chomsky_hierarchy
[dragon]: https://en.wikipedia.org/wiki/Compilers:_Principles,_Techniques,_and_Tools
[finite-state machines]: https://en.wikipedia.org/wiki/Finite-state_machine

</aside>

You very precisely *can* recognize all of the different lexemes for Lox using
regexes if you want to, and there's a pile of interesting theory underlying why
that is and what it means. Tools like [Lex][] or
[Flex][] are designed expressly to let you do this -- throw a handful of regexes
at them, and they give you a complete scanner <span name="lex">back</span>.

<aside name="lex">

Lex was created by Mike Lesk and Eric Schmidt. Yes, the same Eric Schmidt who
was executive chairman of Google. I'm not saying programming languages are a
surefire path to wealth and fame, but we *can* count at least one
mega billionaire among us.

</aside>

[lex]: http://dinosaur.compilertools.net/lex/
[flex]: https://github.com/westes/flex

Since our goal is to understand how a scanner does what it does, we won't be
delegating that task. We're about handcrafted goods.

## The Scanner Class

Without further ado, let's make ourselves a scanner.

^code scanner-class

<aside name="static-import">

I know static imports are considered bad style by some, but they save me from
having to sprinkle `TokenType.` all over the scanner and parser. Forgive me, but
every character counts in a book.

</aside>

We store the raw source code as a simple string, and we have a list ready to
fill with tokens we're going to generate. The aforementioned loop that does that
looks like this:

^code scan-tokens

The scanner works its way through the source code, adding tokens until it runs
out of characters. Then it appends one final "end of file" token. That isn't
strictly needed, but it makes our parser a little cleaner.

This loop depends on a couple of fields to keep track of where the scanner is in
the source code.

^code scan-state (1 before, 2 after)

The `start` and `current` fields are offsets that index into the string. The
`start` field points to the first character in the lexeme being scanned, and
`current` points at the character currently being considered. The `line` field
tracks what source line `current` is on so we can produce tokens that know their
location.

Then we have one little helper function that tells us if we've consumed all the
characters.

^code is-at-end

## Recognizing Lexemes

In each turn of the loop, we scan a single token. This is the real heart of the
scanner. We'll start simple. Imagine if every lexeme were only a single character
long. All you would need to do is consume the next character and pick a token type for
it. Several lexemes *are* only a single character in Lox, so let's start with
those.

^code scan-token

<aside name="slash">

Wondering why `/` isn't in here? Don't worry, we'll get to it.

</aside>

Again, we need a couple of helper methods.

^code advance-and-add-token

The `advance()` method consumes the next character in the source file and
returns it. Where `advance()` is for input, `addToken()` is for output. It grabs
the text of the current lexeme and creates a new token for it. We'll use the
other overload to handle tokens with literal values soon.

### Lexical errors

Before we get too far in, let's take a moment to think about errors at the
lexical level. What happens if a user throws a source file containing some
characters Lox doesn't use, like `@#^`, at our interpreter? Right now, those
characters get silently discarded. They aren't used by the Lox language, but
that doesn't mean the interpreter can pretend they aren't there. Instead, we
report an error.

^code char-error (1 before, 1 after)

Note that the erroneous character is still *consumed* by the earlier call to
`advance()`. That's important so that we don't get stuck in an infinite loop.

Note also that we <span name="shotgun">*keep scanning*</span>. There may be
other errors later in the program. It gives our users a better experience if we
detect as many of those as possible in one go. Otherwise, they see one tiny
error and fix it, only to have the next error appear, and so on. Syntax error
Whac-A-Mole is no fun.

(Don't worry. Since `hadError` gets set, we'll never try to *execute* any of the
code, even though we keep going and scan the rest of it.)

<aside name="shotgun">

The code reports each invalid character separately, so this shotguns the user
with a blast of errors if they accidentally paste a big blob of weird text.
Coalescing a run of invalid characters into a single error would give a nicer
user experience.

</aside>

### Operators

We have single-character lexemes working, but that doesn't cover all of Lox's
operators. What about `!`? It's a single character, right? Sometimes, yes, but
if the very next character is an equals sign, then we should instead create a
`!=` lexeme. Note that the `!` and `=` are *not* two independent operators. You
can't write `!   =` in Lox and have it behave like an inequality operator.
That's why we need to scan `!=` as a single lexeme. Likewise, `<`, `>`, and `=`
can all be followed by `=` to create the other equality and comparison
operators.

For all of these, we need to look at the second character.

^code two-char-tokens (1 before, 2 after)

Those cases use this new method:

^code match

It's like a conditional `advance()`. We only consume the current character if
it's what we're looking for.

Using `match()`, we recognize these lexemes in two stages. When we reach, for
example, `!`, we jump to its switch case. That means we know the lexeme *starts*
with `!`. Then we look at the next character to determine if we're on a `!=` or
merely a `!`.

## Longer Lexemes

We're still missing one operator: `/` for division. That character needs a
little special handling because comments begin with a slash too.

^code slash (1 before, 2 after)

This is similar to the other two-character operators, except that when we find a
second `/`, we don't end the token yet. Instead, we keep consuming characters
until we reach the end of the line.

This is our general strategy for handling longer lexemes. After we detect the
beginning of one, we shunt over to some lexeme-specific code that keeps eating
characters until it sees the end.

We've got another helper:

^code peek

It's sort of like `advance()`, but doesn't consume the character. This is called
<span name="match">**lookahead**</span>. Since it only looks at the current
unconsumed character, we have *one character of lookahead*. The smaller this
number is, generally, the faster the scanner runs. The rules of the lexical
grammar dictate how much lookahead we need. Fortunately, most languages in wide
use peek only one or two characters ahead.

<aside name="match">

Technically, `match()` is doing lookahead too. `advance()` and `peek()` are the
fundamental operators and `match()` combines them.

</aside>

Comments are lexemes, but they aren't meaningful, and the parser doesn't want
to deal with them. So when we reach the end of the comment, we *don't* call
`addToken()`. When we loop back around to start the next lexeme, `start` gets
reset and the comment's lexeme disappears in a puff of smoke.

While we're at it, now's a good time to skip over those other meaningless
characters: newlines and whitespace.

^code whitespace (1 before, 3 after)

When encountering whitespace, we simply go back to the beginning of the scan
loop. That starts a new lexeme *after* the whitespace character. For newlines,
we do the same thing, but we also increment the line counter. (This is why we
used `peek()` to find the newline ending a comment instead of `match()`. We want
that newline to get us here so we can update `line`.)

Our scanner is getting smarter. It can handle fairly free-form code like:

```lox
// this is a comment
(( )){} // grouping stuff
!*+-/=<> <= == // operators
```

### String literals

Now that we're comfortable with longer lexemes, we're ready to tackle literals.
We'll do strings first, since they always begin with a specific character, `"`.

^code string-start (1 before, 2 after)

That calls:

^code string

Like with comments, we consume characters until we hit the `"` that ends the
string. We also gracefully handle running out of input before the string is
closed and report an error for that.

For no particular reason, Lox supports multi-line strings. There are pros and
cons to that, but prohibiting them was a little more complex than allowing them,
so I left them in. That does mean we also need to update `line` when we hit a
newline inside a string.

Finally, the last interesting bit is that when we create the token, we also
produce the actual string *value* that will be used later by the interpreter.
Here, that conversion only requires a `substring()` to strip off the surrounding
quotes. If Lox supported escape sequences like `\n`, we'd unescape those here.

### Number literals

All numbers in Lox are floating point at runtime, but both integer and decimal
literals are supported. A number literal is a series of <span
name="minus">digits</span> optionally followed by a `.` and one or more trailing
digits.

<aside name="minus">

Since we look only for a digit to start a number, that means `-123` is not a
number *literal*. Instead, `-123`, is an *expression* that applies `-` to the
number literal `123`. In practice, the result is the same, though it has one
interesting edge case if we were to add method calls on numbers. Consider:

```lox
print -123.abs();
```

This prints `-123` because negation has lower precedence than method calls. We
could fix that by making `-` part of the number literal. But then consider:

```lox
var n = 123;
print -n.abs();
```

This still produces `-123`, so now the language seems inconsistent. No matter
what you do, some case ends up weird.

</aside>

```lox
1234
12.34
```

We don't allow a leading or trailing decimal point, so these are both invalid:

```lox
.1234
1234.
```

We could easily support the former, but I left it out to keep things simple. The
latter gets weird if we ever want to allow methods on numbers like `123.sqrt()`.

To recognize the beginning of a number lexeme, we look for any digit. It's kind
of tedious to add cases for every decimal digit, so we'll stuff it in the
default case instead.

^code digit-start (1 before, 1 after)

This relies on this little utility:

^code is-digit

<aside name="is-digit">

The Java standard library provides [`Character.isDigit()`][is-digit], which seems
like a good fit. Alas, that method allows things like Devanagari digits,
full-width numbers, and other funny stuff we don't want.

[is-digit]: http://docs.oracle.com/javase/7/docs/api/java/lang/Character.html#isDigit(char)

</aside>

Once we know we are in a number, we branch to a separate method to consume the
rest of the literal, like we do with strings.

^code number

We consume as many digits as we find for the integer part of the literal. Then
we look for a fractional part, which is a decimal point (`.`) followed by at
least one digit. If we do have a fractional part, again, we consume as many
digits as we can find.

Looking past the decimal point requires a second character of lookahead since we
don't want to consume the `.` until we're sure there is a digit *after* it. So
we add:

^code peek-next

<aside name="peek-next">

I could have made `peek()` take a parameter for the number of characters ahead
to look instead of defining two functions, but that would allow *arbitrarily*
far lookahead. Providing these two functions makes it clearer to a reader of the
code that our scanner looks ahead at most two characters.

</aside>


Finally, we convert the lexeme to its numeric value. Our interpreter uses Java's
`Double` type to represent numbers, so we produce a value of that type. We're
using Java's own parsing method to convert the lexeme to a real Java double. We
could implement that ourselves, but, honestly, unless you're trying to cram for
an upcoming programming interview, it's not worth your time.

The remaining literals are Booleans and `nil`, but we handle those as keywords,
which gets us to...

## Reserved Words and Identifiers

Our scanner is almost done. The only remaining pieces of the lexical grammar to
implement are identifiers and their close cousins, the reserved words. You might
think we could match keywords like `or` in the same way we handle
multiple-character operators like `<=`.

```java
case 'o':
  if (match('r')) {
    addToken(OR);
  }
  break;
```

Consider what would happen if a user named a variable `orchid`. The scanner
would see the first two letters, `or`, and immediately emit an `or` keyword
token. This gets us to an important principle called <span
name="maximal">**maximal munch**</span>. When two lexical grammar rules can both
match a chunk of code that the scanner is looking at, *whichever one matches the
most characters wins*.

That rule states that if we can match `orchid` as an identifier and `or` as a
keyword, then the former wins. This is also why we tacitly assumed, previously,
that `<=` should be scanned as a single `<=` token and not `<` followed by `=`.

<aside name="maximal">

Consider this nasty bit of C code:

```c
---a;
```

Is it valid? That depends on how the scanner splits the lexemes. What if the scanner
sees it like this:

```c
- --a;
```

Then it could be parsed. But that would require the scanner to know about the
grammatical structure of the surrounding code, which entangles things more than
we want. Instead, the maximal munch rule says that it is *always* scanned like:

```c
-- -a;
```

It scans it that way even though doing so leads to a syntax error later in the
parser.

</aside>

Maximal munch means we can't easily detect a reserved word until we've reached
the end of what might instead be an identifier. After all, a reserved word *is*
an identifier, it's just one that has been claimed by the language for its own
use. That's where the term **reserved word** comes from.

So we begin by assuming any lexeme starting with a letter or underscore is an
identifier.

^code identifier-start (3 before, 3 after)

The rest of the code lives over here:

^code identifier

We define that in terms of these helpers:

^code is-alpha

That gets identifiers working. To handle keywords, we see if the identifier's
lexeme is one of the reserved words. If so, we use a token type specific to that
keyword. We define the set of reserved words in a map.

^code keyword-map

Then, after we scan an identifier, we check to see if it matches anything in the
map.

^code keyword-type (2 before, 1 after)

If so, we use that keyword's token type. Otherwise, it's a regular user-defined
identifier.

And with that, we now have a complete scanner for the entire Lox lexical
grammar. Fire up the REPL and type in some valid and invalid code. Does it
produce the tokens you expect? Try to come up with some interesting edge cases
and see if it handles them as it should.

<div class="challenges">

## Challenges

1.  The lexical grammars of Python and Haskell are not *regular*. What does that
    mean, and why aren't they?

1.  Aside from separating tokens -- distinguishing `print foo` from `printfoo`
    -- spaces aren't used for much in most languages. However, in a couple of
    dark corners, a space *does* affect how code is parsed in CoffeeScript,
    Ruby, and the C preprocessor. Where and what effect does it have in each of
    those languages?

1.  Our scanner here, like most, discards comments and whitespace since those
    aren't needed by the parser. Why might you want to write a scanner that does
    *not* discard those? What would it be useful for?

1.  Add support to Lox's scanner for C-style `/* ... */` block comments. Make
    sure to handle newlines in them. Consider allowing them to nest. Is adding
    support for nesting more work than you expected? Why?

</div>

<div class="design-note">

## Design Note: Implicit Semicolons

Programmers today are spoiled for choice in languages and have gotten picky
about syntax. They want their language to look clean and modern. One bit of
syntactic lichen that almost every new language scrapes off (and some ancient
ones like BASIC never had) is `;` as an explicit statement terminator.

Instead, they treat a newline as a statement terminator where it makes sense to
do so. The "where it makes sense" part is the challenging bit. While *most*
statements are on their own line, sometimes you need to spread a single
statement across a couple of lines. Those intermingled newlines should not be
treated as terminators.

Most of the obvious cases where the newline should be ignored are easy to
detect, but there are a handful of nasty ones:

* A return value on the next line:

    ```js
    if (condition) return
    "value"
    ```

    Is "value" the value being returned, or do we have a `return` statement with
    no value followed by an expression statement containing a string literal?

* A parenthesized expression on the next line:

    ```js
    func
    (parenthesized)
    ```

    Is this a call to `func(parenthesized)`, or two expression statements, one
    for `func` and one for a parenthesized expression?

* A `-` on the next line:

    ```js
    first
    -second
    ```

    Is this `first - second` -- an infix subtraction -- or two expression
    statements, one for `first` and one to negate `second`?

In all of these, either treating the newline as a separator or not would both
produce valid code, but possibly not the code the user wants. Across languages,
there is an unsettling variety of rules used to decide which newlines are
separators. Here are a couple:

*   [Lua][] completely ignores newlines, but carefully controls its grammar such
    that no separator between statements is needed at all in most cases. This is
    perfectly legit:

    ```lua
    a = 1 b = 2
    ```

    Lua avoids the `return` problem by requiring a `return` statement to be the
    very last statement in a block. If there is a value after `return` before
    the keyword `end`, it *must* be for the `return`. For the other two cases,
    they allow an explicit `;` and expect users to use that. In practice, that
    almost never happens because there's no point in a parenthesized or unary
    negation expression statement.

*   [Go][] handles newlines in the scanner. If a newline appears following one
    of a handful of token types that are known to potentially end a statement,
    the newline is treated like a semicolon. Otherwise it is ignored. The Go
    team provides a canonical code formatter, [gofmt][], and the ecosystem is
    fervent about its use, which ensures that idiomatic styled code works well
    with this simple rule.

*   [Python][] treats all newlines as significant unless an explicit backslash
    is used at the end of a line to continue it to the next line. However,
    newlines anywhere inside a pair of brackets (`()`, `[]`, or `{}`) are
    ignored. Idiomatic style strongly prefers the latter.

    This rule works well for Python because it is a highly statement-oriented
    language. In particular, Python's grammar ensures a statement never appears
    inside an expression. C does the same, but many other languages which have a
    "lambda" or function literal syntax do not.

    An example in JavaScript:

    ```js
    console.log(function() {
      statement();
    });
    ```

    Here, the `console.log()` *expression* contains a function literal which
    in turn contains the *statement* `statement();`.

    Python would need a different set of rules for implicitly joining lines if
    you could get back *into* a <span name="lambda">statement</span> where
    newlines should become meaningful while still nested inside brackets.

<aside name="lambda">

And now you know why Python's `lambda` allows only a single expression body.

</aside>

*   JavaScript's "[automatic semicolon insertion][asi]" rule is the real odd
    one. Where other languages assume most newlines *are* meaningful and only a
    few should be ignored in multi-line statements, JS assumes the opposite. It
    treats all of your newlines as meaningless whitespace *unless* it encounters
    a parse error. If it does, it goes back and tries turning the previous
    newline into a semicolon to get something grammatically valid.

    This design note would turn into a design diatribe if I went into complete
    detail about how that even *works*, much less all the various ways that
    JavaScript's "solution" is a bad idea. It's a mess. JavaScript is the only
    language I know where many style guides demand explicit semicolons after
    every statement even though the language theoretically lets you elide them.

If you're designing a new language, you almost surely *should* avoid an explicit
statement terminator. Programmers are creatures of fashion like other humans, and
semicolons are as passé as ALL CAPS KEYWORDS. Just make sure you pick a set of
rules that make sense for your language's particular grammar and idioms. And
don't do what JavaScript did.

</div>

[lua]: https://www.lua.org/pil/1.1.html
[go]: https://golang.org/ref/spec#Semicolons
[gofmt]: https://golang.org/cmd/gofmt/
[python]: https://docs.python.org/3.5/reference/lexical_analysis.html#implicit-line-joining
[asi]: https://www.ecma-international.org/ecma-262/5.1/#sec-7.9
