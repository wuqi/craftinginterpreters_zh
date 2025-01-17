> 偶尔，你会发现自己处于一种奇怪的境地。你以最自然的方式逐渐进入其中，但当你置身其中时，你会突然大吃一惊，问自己这一切到底是怎么发生的。
>
> <cite>索尔·海尔达尔, <em>康提基号</em></cite>

哦，糟糕！我们的语言实现正在遭受水淹之灾！很久以前，当我们[添加变量和代码块][statements]时，我们的作用域紧密而完美。然而，当我们后来[引入闭包][functions]时，我们原本防水的解释器出现了一道裂缝。虽然大多数真实的程序不太可能从这个裂缝中溜走，但作为语言实现者，我们立下了神圣的誓言，即在语义的最深处、最潮湿的角落，我们也要追求正确无误。现在，我们需要修补这个漏洞，以保护我们的语言世界免受任何不可预测的灾害。

[statements]: statements-and-state.html
[functions]: functions.html

我们将在本章中全面探讨这个漏洞，然后仔细修补它。在此过程中，我们将更加深入地了解词法作用域，以及它在Lox和其他C语言中的应用。我们还将有机会学习*语义分析*——一种从用户源代码中提取意义而无需运行它的强大技术。

## 静态作用域

简单回顾一下：Lox，就像大多数现代语言一样，使用*词法*作用域。这意味着你只需阅读程序的文本，就能确定变量名引用的是哪个声明。例如：

```lox
var a = "outer";
{
  var a = "inner";
  print a;
}
```

在这个例子中，我们知道被打印的`a`是上一行声明的变量，而不是全局变量。运行程序不会——也*不能*——影响这一点。作用域规则是语言*静态*语义的一部分，这就是为什么它们也被称为*静态作用域*。

我还没有详细解释这些作用域规则，但现在是时候<span name="precise">讲清楚</span>了：

<aside name="precise">

这仍然远远不及真正的语言规范那么精确。那些文档必须非常明确，以至于即使是火星人或彻头彻尾的恶意程序员，如果遵守规范中的规定，也会被迫实现正确的语义。

当一种语言可能由相互竞争的公司实现时，这种精确性很重要，这些公司希望他们的产品与其它产品不兼容，以将客户锁定在他们的平台上。对于这本书来说，我们可以很庆幸地忽略那些阴暗的欺诈行为。

</aside>

**变量指向的是使用变量的表达式外围环境中，前面具有相同名称的最内层作用域中的变量声明。**

这里有很多要解释的东西:

*   I say "variable usage" instead of "variable expression" to cover both
    variable expressions and assignments. Likewise with "expression where the
    variable is used".

*   "Preceding" means appearing before *in the program text*.

    ```lox
    var a = "outer";
    {
      print a;
      var a = "inner";
    }
    ```

    Here, the `a` being printed is the outer one since it appears <span
    name="hoisting">before</span> the `print` statement that uses it. In most
    cases, in straight line code, the declaration preceding in *text* will also
    precede the usage in *time*. But that's not always true. As we'll see,
    functions may defer a chunk of code such that its *dynamic temporal*
    execution no longer mirrors the *static textual* ordering.

    <aside name="hoisting">

    In JavaScript, variables declared using `var` are implicitly "hoisted" to
    the beginning of the block. Any use of that name in the block will refer to
    that variable, even if the use appears before the declaration. When you
    write this in JavaScript:

    ```js
    {
      console.log(a);
      var a = "value";
    }
    ```

    It behaves like:

    ```js
    {
      var a; // Hoist.
      console.log(a);
      a = "value";
    }
    ```

    That means that in some cases you can read a variable before its initializer
    has run -- an annoying source of bugs. The alternate `let` syntax for
    declaring variables was added later to address this problem.

    </aside>

*   "Innermost" is there because of our good friend shadowing. There may be more
    than one variable with the given name in enclosing scopes, as in:

    ```lox
    var a = "outer";
    {
      var a = "inner";
      print a;
    }
    ```

    Our rule disambiguates this case by saying the innermost scope wins.

Since this rule makes no mention of any runtime behavior, it implies that a
variable expression always refers to the same declaration through the entire
execution of the program. Our interpreter so far *mostly* implements the rule
correctly. But when we added closures, an error snuck in.

```lox
var a = "global";
{
  fun showA() {
    print a;
  }

  showA();
  var a = "block";
  showA();
}
```

<span name="tricky">Before</span> you type this in and run it, decide what you
think it *should* print.

<aside name="tricky">

I know, it's a totally pathological, contrived program. It's just *weird*. No
reasonable person would ever write code like this. Alas, more of your life than
you'd expect will be spent dealing with bizarro snippets of code like this if
you stay in the programming language game for long.

</aside>

OK... got it? If you're familiar with closures in other languages, you'll expect
it to print "global" twice. The first call to `showA()` should definitely print
"global" since we haven't even reached the declaration of the inner `a` yet. And
by our rule that a variable expression always resolves to the same variable,
that implies the second call to `showA()` should print the same thing.

Alas, it prints:

```text
global
block
```

Let me stress that this program never reassigns any variable and contains only a
single `print` statement. Yet, somehow, that `print` statement for a
never-assigned variable prints two different values at different points in time.
We definitely broke something somewhere.

### Scopes and mutable environments

In our interpreter, environments are the dynamic manifestation of static scopes.
The two mostly stay in sync with each other -- we create a new environment when
we enter a new scope, and discard it when we leave the scope. There is one other
operation we perform on environments: binding a variable in one. This is where
our bug lies.

Let's walk through that problematic example and see what the environments look
like at each step. First, we declare `a` in the global scope.

<img src="image/resolving-and-binding/environment-1.png" alt="The global environment with 'a' defined in it." />

That gives us a single environment with a single variable in it. Then we enter
the block and execute the declaration of `showA()`.

<img src="image/resolving-and-binding/environment-2.png" alt="A block environment linking to the global one." />

We get a new environment for the block. In that, we declare one name, `showA`,
which is bound to the LoxFunction object we create to represent the function.
That object has a `closure` field that captures the environment where the
function was declared, so it has a reference back to the environment for the
block.

Now we call `showA()`.

<img src="image/resolving-and-binding/environment-3.png" alt="An empty environment for showA()'s body linking to the previous two. 'a' is resolved in the global environment." />

The interpreter dynamically creates a new environment for the function body of
`showA()`. It's empty since that function doesn't declare any variables. The
parent of that environment is the function's closure -- the outer block
environment.

Inside the body of `showA()`, we print the value of `a`. The interpreter looks
up this value by walking the chain of environments. It gets all the way
to the global environment before finding it there and printing `"global"`.
Great.

Next, we declare the second `a`, this time inside the block.

<img src="image/resolving-and-binding/environment-4.png" alt="The block environment has both 'a' and 'showA' now." />

It's in the same block -- the same scope -- as `showA()`, so it goes into the
same environment, which is also the same environment `showA()`'s closure refers
to. This is where it gets interesting. We call `showA()` again.

<img src="image/resolving-and-binding/environment-5.png" alt="An empty environment for showA()'s body linking to the previous two. 'a' is resolved in the block environment." />

We create a new empty environment for the body of `showA()` again, wire it up to
that closure, and run the body. When the interpreter walks the chain of
environments to find `a`, it now discovers the *new* `a` in the block
environment. Boo.

I chose to implement environments in a way that I hoped would agree with your
informal intuition around scopes. We tend to consider all of the code within a
block as being within the same scope, so our interpreter uses a single
environment to represent that. Each environment is a mutable hash table. When a
new local variable is declared, it gets added to the existing environment for
that scope.

That intuition, like many in life, isn't quite right. A block is not necessarily
all the same scope. Consider:

```lox
{
  var a;
  // 1.
  var b;
  // 2.
}
```

At the first marked line, only `a` is in scope. At the second line, both `a` and
`b` are. If you define a "scope" to be a set of declarations, then those are
clearly not the same scope -- they don't contain the same declarations. It's
like each `var` statement <span name="split">splits</span> the block into two
separate scopes, the scope before the variable is declared and the one after,
which includes the new variable.

<aside name="split">

Some languages make this split explicit. In Scheme and ML, when you declare a
local variable using `let`, you also delineate the subsequent code where the new
variable is in scope. There is no implicit "rest of the block".

</aside>

But in our implementation, environments do act like the entire block is one
scope, just a scope that changes over time. Closures do not like that. When a
function is declared, it captures a reference to the current environment. The
function *should* capture a frozen snapshot of the environment *as it existed at
the moment the function was declared*. But instead, in the Java code, it has a
reference to the actual mutable environment object. When a variable is later
declared in the scope that environment corresponds to, the closure sees the new
variable, even though the declaration does *not* precede the function.

### Persistent environments

There is a style of programming that uses what are called **persistent data
structures**. Unlike the squishy data structures you're familiar with in
imperative programming, a persistent data structure can never be directly
modified. Instead, any "modification" to an existing structure produces a <span
name="copy">brand</span> new object that contains all of the original data and
the new modification. The original is left unchanged.

<aside name="copy">

This sounds like it might waste tons of memory and time copying the structure
for each operation. In practice, persistent data structures share most of their
data between the different "copies".

</aside>

If we were to apply that technique to Environment, then every time you declared
a variable it would return a *new* environment that contained all of the
previously declared variables along with the one new name. Declaring a variable
would do the implicit "split" where you have an environment before the variable
is declared and one after:

<img src="image/resolving-and-binding/split.png" alt="Separate environments before and after the variable is declared." />

A closure retains a reference to the Environment instance in play when the
function was declared. Since any later declarations in that block would produce
new Environment objects, the closure wouldn't see the new variables and our bug
would be fixed.

This is a legit way to solve the problem, and it's the classic way to implement
environments in Scheme interpreters. We could do that for Lox, but it would mean
going back and changing a pile of existing code.

I won't drag you through that. We'll keep the way we represent environments the
same. Instead of making the data more statically structured, we'll bake the
static resolution into the access *operation* itself.

## Semantic Analysis

Our interpreter **resolves** a variable -- tracks down which declaration it
refers to -- each and every time the variable expression is evaluated. If that
variable is swaddled inside a loop that runs a thousand times, that variable
gets re-resolved a thousand times.

We know static scope means that a variable usage always resolves to the same
declaration, which can be determined just by looking at the text. Given that,
why are we doing it dynamically every time? Doing so doesn't just open the hole
that leads to our annoying bug, it's also needlessly slow.

A better solution is to resolve each variable use *once*. Write a chunk of code
that inspects the user's program, finds every variable mentioned, and figures
out which declaration each refers to. This process is an example of a **semantic
analysis**. Where a parser tells only if a program is grammatically correct (a
*syntactic* analysis), semantic analysis goes farther and starts to figure out
what pieces of the program actually mean. In this case, our analysis will
resolve variable bindings. We'll know not just that an expression *is* a
variable, but *which* variable it is.

There are a lot of ways we could store the binding between a variable and its
declaration. When we get to the C interpreter for Lox, we'll have a *much* more
efficient way of storing and accessing local variables. But for jlox, I want to
minimize the collateral damage we inflict on our existing codebase. I'd hate to
throw out a bunch of mostly fine code.

Instead, we'll store the resolution in a way that makes the most out of our
existing Environment class. Recall how the accesses of `a` are interpreted in
the problematic example.

<img src="image/resolving-and-binding/environment-3.png" alt="An empty environment for showA()'s body linking to the previous two. 'a' is resolved in the global environment." />

In the first (correct) evaluation, we look at three environments in the chain
before finding the global declaration of `a`. Then, when the inner `a` is later
declared in a block scope, it shadows the global one.

<img src="image/resolving-and-binding/environment-5.png" alt="An empty environment for showA()'s body linking to the previous two. 'a' is resolved in the block environment." />

The next lookup walks the chain, finds `a` in the *second* environment and
stops there. Each environment corresponds to a single lexical scope where
variables are declared. If we could ensure a variable lookup always walked the
*same* number of links in the environment chain, that would ensure that it
found the same variable in the same scope every time.

To "resolve" a variable usage, we only need to calculate how many "hops" away
the declared variable will be in the environment chain. The interesting question
is *when* to do this calculation -- or, put differently, where in our
interpreter's implementation do we stuff the code for it?

Since we're calculating a static property based on the structure of the source
code, the obvious answer is in the parser. That is the traditional home, and is
where we'll put it later in clox. It would work here too, but I want an excuse to
show you another technique. We'll write our resolver as a separate pass.

### A variable resolution pass

After the parser produces the syntax tree, but before the interpreter starts
executing it, we'll do a single walk over the tree to resolve all of the
variables it contains. Additional passes between parsing and execution are
common. If Lox had static types, we could slide a type checker in there.
Optimizations are often implemented in separate passes like this too. Basically,
any work that doesn't rely on state that's only available at runtime can be done
in this way.

Our variable resolution pass works like a sort of mini-interpreter. It walks the
tree, visiting each node, but a static analysis is different from a dynamic
execution:

*   **There are no side effects.** When the static analysis visits a print
    statement, it doesn't actually print anything. Calls to native functions or
    other operations that reach out to the outside world are stubbed out and
    have no effect.

*   **There is no control flow.** Loops are visited only <span
    name="fix">once</span>. Both branches are visited in `if` statements. Logic
    operators are not short-circuited.

<aside name="fix">

Variable resolution touches each node once, so its performance is *O(n)* where
*n* is the number of syntax tree nodes. More sophisticated analyses may have
greater complexity, but most are carefully designed to be linear or not far from
it. It's an embarrassing faux pas if your compiler gets exponentially slower as
the user's program grows.

</aside>

## A Resolver Class

Like everything in Java, our variable resolution pass is embodied in a class.

^code resolver

Since the resolver needs to visit every node in the syntax tree, it implements
the visitor abstraction we already have in place. Only a few kinds of nodes are
interesting when it comes to resolving variables:

*   A block statement introduces a new scope for the statements it contains.

*   A function declaration introduces a new scope for its body and binds its
    parameters in that scope.

*   A variable declaration adds a new variable to the current scope.

*   Variable and assignment expressions need to have their variables resolved.

The rest of the nodes don't do anything special, but we still need to implement
visit methods for them that traverse into their subtrees. Even though a `+`
expression doesn't *itself* have any variables to resolve, either of its
operands might.

### Resolving blocks

We start with blocks since they create the local scopes where all the magic
happens.

^code visit-block-stmt

This begins a new scope, traverses into the statements inside the block, and
then discards the scope. The fun stuff lives in those helper methods. We start
with the simple one.

^code resolve-statements

This walks a list of statements and resolves each one. It in turn calls:

^code resolve-stmt

While we're at it, let's add another overload that we'll need later for
resolving an expression.

^code resolve-expr

These methods are similar to the `evaluate()` and `execute()` methods in
Interpreter -- they turn around and apply the Visitor pattern to the given
syntax tree node.

The real interesting behavior is around scopes. A new block scope is created
like so:

^code begin-scope

Lexical scopes nest in both the interpreter and the resolver. They behave like a
stack. The interpreter implements that stack using a linked list -- the chain of
Environment objects. In the resolver, we use an actual Java Stack.

^code scopes-field (1 before, 2 after)

This field keeps track of the stack of scopes currently, uh, in scope. Each
element in the stack is a Map representing a single block scope. Keys, as in
Environment, are variable names. The values are Booleans, for a reason I'll
explain soon.

The scope stack is only used for local block scopes. Variables declared at the
top level in the global scope are not tracked by the resolver since they are
more dynamic in Lox. When resolving a variable, if we can't find it in the stack
of local scopes, we assume it must be global.

Since scopes are stored in an explicit stack, exiting one is straightforward.

^code end-scope

Now we can push and pop a stack of empty scopes. Let's put some things in them.

### Resolving variable declarations

Resolving a variable declaration adds a new entry to the current innermost
scope's map. That seems simple, but there's a little dance we need to do.

^code visit-var-stmt

We split binding into two steps, declaring then defining, in order to handle
funny edge cases like this:

```lox
var a = "outer";
{
  var a = a;
}
```

What happens when the initializer for a local variable refers to a variable with
the same name as the variable being declared? We have a few options:

1.  **Run the initializer, then put the new variable in scope.** Here, the new
    local `a` would be initialized with "outer", the value of the *global* one.
    In other words, the previous declaration would desugar to:

    ```lox
    var temp = a; // Run the initializer.
    var a;        // Declare the variable.
    a = temp;     // Initialize it.
    ```

2.  **Put the new variable in scope, then run the initializer.** This means you
    could observe a variable before it's initialized, so we would need to figure
    out what value it would have then. Probably `nil`. That means the new local
    `a` would be re-initialized to its own implicitly initialized value, `nil`.
    Now the desugaring would look like:

    ```lox
    var a; // Define the variable.
    a = a; // Run the initializer.
    ```

3.  **Make it an error to reference a variable in its initializer.** Have the
    interpreter fail either at compile time or runtime if an initializer
    mentions the variable being initialized.

Do either of those first two options look like something a user actually
*wants*? Shadowing is rare and often an error, so initializing a shadowing
variable based on the value of the shadowed one seems unlikely to be deliberate.

The second option is even less useful. The new variable will *always* have the
value `nil`. There is never any point in mentioning it by name. You could use an
explicit `nil` instead.

Since the first two options are likely to mask user errors, we'll take the
third. Further, we'll make it a compile error instead of a runtime one. That
way, the user is alerted to the problem before any code is run.

In order to do that, as we visit expressions, we need to know if we're inside
the initializer for some variable. We do that by splitting binding into two
steps. The first is **declaring** it.

^code declare

Declaration adds the variable to the innermost scope so that it shadows any
outer one and so that we know the variable exists. We mark it as "not ready yet"
by binding its name to `false` in the scope map. The value associated with a key
in the scope map represents whether or not we have finished resolving that
variable's initializer.

After declaring the variable, we resolve its initializer expression in that same
scope where the new variable now exists but is unavailable. Once the initializer
expression is done, the variable is ready for prime time. We do that by
**defining** it.

^code define

We set the variable's value in the scope map to `true` to mark it as fully
initialized and available for use. It's alive! 

### Resolving variable expressions

Variable declarations -- and function declarations, which we'll get to -- write
to the scope maps. Those maps are read when we resolve variable expressions.

^code visit-variable-expr

First, we check to see if the variable is being accessed inside its own
initializer. This is where the values in the scope map come into play. If the
variable exists in the current scope but its value is `false`, that means we
have declared it but not yet defined it. We report that error.

After that check, we actually resolve the variable itself using this helper:

^code resolve-local

This looks, for good reason, a lot like the code in Environment for evaluating a
variable. We start at the innermost scope and work outwards, looking in each map
for a matching name. If we find the variable, we resolve it, passing in the
number of scopes between the current innermost scope and the scope where the
variable was found. So, if the variable was found in the current scope, we
pass in 0. If it's in the immediately enclosing scope, 1. You get the idea.

If we walk through all of the block scopes and never find the variable, we leave
it unresolved and assume it's global. We'll get to the implementation of that
`resolve()` method a little later. For now, let's keep on cranking through the
other syntax nodes.

### Resolving assignment expressions

The other expression that references a variable is assignment. Resolving one
looks like this:

^code visit-assign-expr

First, we resolve the expression for the assigned value in case it also contains
references to other variables. Then we use our existing `resolveLocal()` method
to resolve the variable that's being assigned to.

### Resolving function declarations

Finally, functions. Functions both bind names and introduce a scope. The name of
the function itself is bound in the surrounding scope where the function is
declared. When we step into the function's body, we also bind its parameters
into that inner function scope.

^code visit-function-stmt

Similar to `visitVariableStmt()`, we declare and define the name of the function
in the current scope. Unlike variables, though, we define the name eagerly,
before resolving the function's body. This lets a function recursively refer to
itself inside its own body.

Then we resolve the function's body using this:

^code resolve-function

It's a separate method since we will also use it for resolving Lox methods when
we add classes later. It creates a new scope for the body and then binds
variables for each of the function's parameters.

Once that's ready, it resolves the function body in that scope. This is
different from how the interpreter handles function declarations. At *runtime*,
declaring a function doesn't do anything with the function's body. The body
doesn't get touched until later when the function is called. In a *static*
analysis, we immediately traverse into the body right then and there.

### Resolving the other syntax tree nodes

That covers the interesting corners of the grammars. We handle every place where
a variable is declared, read, or written, and every place where a scope is
created or destroyed. Even though they aren't affected by variable resolution,
we also need visit methods for all of the other syntax tree nodes in order to
recurse into their subtrees. <span name="boring">Sorry</span> this bit is
boring, but bear with me. We'll go kind of "top down" and start with statements.

<aside name="boring">

I did say the book would have every single line of code for these interpreters.
I didn't say they'd all be exciting.

</aside>

An expression statement contains a single expression to traverse.

^code visit-expression-stmt

An if statement has an expression for its condition and one or two statements
for the branches.

^code visit-if-stmt

Here, we see how resolution is different from interpretation. When we resolve an
`if` statement, there is no control flow. We resolve the condition and *both*
branches. Where a dynamic execution steps only into the branch that *is* run, a
static analysis is conservative -- it analyzes any branch that *could* be run.
Since either one could be reached at runtime, we resolve both.

Like expression statements, a `print` statement contains a single subexpression.

^code visit-print-stmt

Same deal for return.

^code visit-return-stmt

As in `if` statements, with a `while` statement, we resolve its condition and
resolve the body exactly once.

^code visit-while-stmt

That covers all the statements. On to expressions...

Our old friend the binary expression. We traverse into and resolve both
operands.

^code visit-binary-expr

Calls are similar -- we walk the argument list and resolve them all. The thing
being called is also an expression (usually a variable expression), so that gets
resolved too.

^code visit-call-expr

Parentheses are easy.

^code visit-grouping-expr

Literals are easiest of all.

^code visit-literal-expr

A literal expression doesn't mention any variables and doesn't contain any
subexpressions so there is no work to do.

Since a static analysis does no control flow or short-circuiting, logical
expressions are exactly the same as other binary operators.

^code visit-logical-expr

And, finally, the last node. We resolve its one operand.

^code visit-unary-expr

With all of these visit methods, the Java compiler should be satisfied that
Resolver fully implements Stmt.Visitor and Expr.Visitor. Now is a good time to
take a break, have a snack, maybe a little nap.

## Interpreting Resolved Variables

Let's see what our resolver is good for. Each time it visits a variable, it
tells the interpreter how many scopes there are between the current scope and
the scope where the variable is defined. At runtime, this corresponds exactly to
the number of *environments* between the current one and the enclosing one where
the interpreter can find the variable's value. The resolver hands that number to
the interpreter by calling this:

^code resolve

We want to store the resolution information somewhere so we can use it when the
variable or assignment expression is later executed, but where? One obvious
place is right in the syntax tree node itself. That's a fine approach, and
that's where many compilers store the results of analyses like this.

We could do that, but it would require mucking around with our syntax tree
generator. Instead, we'll take another common approach and store it off to the
<span name="side">side</span> in a map that associates each syntax tree node
with its resolved data.

<aside name="side">

I *think* I've heard this map called a "side table" since it's a tabular data
structure that stores data separately from the objects it relates to. But
whenever I try to Google for that term, I get pages about furniture.

</aside>

Interactive tools like IDEs often incrementally reparse and re-resolve parts of
the user's program. It may be hard to find all of the bits of state that need
recalculating when they're hiding in the foliage of the syntax tree. A benefit
of storing this data outside of the nodes is that it makes it easy to *discard*
it -- simply clear the map.

^code locals-field (1 before, 2 after)

You might think we'd need some sort of nested tree structure to avoid getting
confused when there are multiple expressions that reference the same variable,
but each expression node is its own Java object with its own unique identity. A
single monolithic map doesn't have any trouble keeping them separated.

As usual, using a collection requires us to import a couple of names.

^code import-hash-map (1 before, 1 after)

And:

^code import-map (1 before, 2 after)

### Accessing a resolved variable

Our interpreter now has access to each variable's resolved location. Finally, we
get to make use of that. We replace the visit method for variable expressions
with this:

^code call-look-up-variable (1 before, 1 after)

That delegates to:

^code look-up-variable

There are a couple of things going on here. First, we look up the resolved
distance in the map. Remember that we resolved only *local* variables. Globals
are treated specially and don't end up in the map (hence the name `locals`). So,
if we don't find a distance in the map, it must be global. In that case, we
look it up, dynamically, directly in the global environment. That throws a
runtime error if the variable isn't defined.

If we *do* get a distance, we have a local variable, and we get to take
advantage of the results of our static analysis. Instead of calling `get()`, we
call this new method on Environment:

^code get-at

The old `get()` method dynamically walks the chain of enclosing environments,
scouring each one to see if the variable might be hiding in there somewhere. But
now we know exactly which environment in the chain will have the variable. We
reach it using this helper method:

^code ancestor

This walks a fixed number of hops up the parent chain and returns the
environment there. Once we have that, `getAt()` simply returns the value of the
variable in that environment's map. It doesn't even have to check to see if the
variable is there -- we know it will be because the resolver already found it
before.

<aside name="coupled">

The way the interpreter assumes the variable is in that map feels like flying
blind. The interpreter code trusts that the resolver did its job and resolved
the variable correctly. This implies a deep coupling between these two classes.
In the resolver, each line of code that touches a scope must have its exact
match in the interpreter for modifying an environment.

I felt that coupling firsthand because as I wrote the code for the book, I
ran into a couple of subtle bugs where the resolver and interpreter code were
slightly out of sync. Tracking those down was difficult. One tool to make that
easier is to have the interpreter explicitly assert -- using Java's assert
statements or some other validation tool -- the contract it expects the resolver
to have already upheld.

</aside>

### Assigning to a resolved variable

We can also use a variable by assigning to it. The changes to visiting an
assignment expression are similar.

^code resolved-assign (2 before, 1 after)

Again, we look up the variable's scope distance. If not found, we assume it's
global and handle it the same way as before. Otherwise, we call this new method:

^code assign-at

As `getAt()` is to `get()`, `assignAt()` is to `assign()`. It walks a fixed
number of environments, and then stuffs the new value in that map.

Those are the only changes to Interpreter. This is why I chose a representation
for our resolved data that was minimally invasive. All of the rest of the nodes
continue working as they did before. Even the code for modifying environments is
unchanged.

### Running the resolver

We do need to actually *run* the resolver, though. We insert the new pass after
the parser does its magic.

^code create-resolver (3 before, 1 after)

We don't run the resolver if there are any parse errors. If the code has a
syntax error, it's never going to run, so there's little value in resolving it.
If the syntax is clean, we tell the resolver to do its thing. The resolver has a
reference to the interpreter and pokes the resolution data directly into it as
it walks over variables. When the interpreter runs next, it has everything it
needs.

At least, that's true if the resolver *succeeds*. But what about errors during
resolution?

## Resolution Errors

Since we are doing a semantic analysis pass, we have an opportunity to make
Lox's semantics more precise, and to help users catch bugs early before running
their code. Take a look at this bad boy:

```lox
fun bad() {
  var a = "first";
  var a = "second";
}
```

We do allow declaring multiple variables with the same name in the *global*
scope, but doing so in a local scope is probably a mistake. If they knew the
variable already existed, they would have assigned to it instead of using `var`.
And if they *didn't* know it existed, they probably didn't intend to overwrite
the previous one.

We can detect this mistake statically while resolving.

^code duplicate-variable (1 before, 1 after)

When we declare a variable in a local scope, we already know the names of every
variable previously declared in that same scope. If we see a collision, we
report an error.

### Invalid return errors

Here's another nasty little script:

```lox
return "at top level";
```

This executes a `return` statement, but it's not even inside a function at all.
It's top-level code. I don't know what the user *thinks* is going to happen, but
I don't think we want Lox to allow this.

We can extend the resolver to detect this statically. Much like we track scopes
as we walk the tree, we can track whether or not the code we are currently
visiting is inside a function declaration.

^code function-type-field (1 before, 2 after)

Instead of a bare Boolean, we use this funny enum:

^code function-type

It seems kind of dumb now, but we'll add a couple more cases to it later and
then it will make more sense. When we resolve a function declaration, we pass
that in.

^code pass-function-type (2 before, 1 after)

Over in `resolveFunction()`, we take that parameter and store it in the field
before resolving the body.

^code set-current-function (1 after)

We stash the previous value of the field in a local variable first. Remember,
Lox has local functions, so you can nest function declarations arbitrarily
deeply. We need to track not just that we're in a function, but *how many* we're
in.

We could use an explicit stack of FunctionType values for that, but instead
we'll piggyback on the JVM. We store the previous value in a local on the Java
stack. When we're done resolving the function body, we restore the field to that
value.

^code restore-current-function (1 before, 1 after)

Now that we can always tell whether or not we're inside a function declaration,
we check that when resolving a `return` statement.

^code return-from-top (1 before, 1 after)

Neat, right?

There's one more piece. Back in the main Lox class that stitches everything
together, we are careful to not run the interpreter if any parse errors are
encountered. That check runs *before* the resolver so that we don't try to
resolve syntactically invalid code.

But we also need to skip the interpreter if there are resolution errors, so we
add *another* check.

^code resolution-error (1 before, 2 after)

You could imagine doing lots of other analysis in here. For example, if we added
`break` statements to Lox, we would probably want to ensure they are only used
inside loops.

We could go farther and report warnings for code that isn't necessarily *wrong*
but probably isn't useful. For example, many IDEs will warn if you have
unreachable code after a `return` statement, or a local variable whose value is
never read. All of that would be pretty easy to add to our static visiting pass,
or as <span name="separate">separate</span> passes.

<aside name="separate">

The choice of how many different analyses to lump into a single pass is
difficult. Many small isolated passes, each with their own responsibility, are
simpler to implement and maintain. However, there is a real runtime cost to
traversing the syntax tree itself, so bundling multiple analyses into a single
pass is usually faster.

</aside>

But, for now, we'll stick with that limited amount of analysis. The important
part is that we fixed that one weird annoying edge case bug, though it might be
surprising that it took this much work to do it.

<div class="challenges">

## Challenges

1.  Why is it safe to eagerly define the variable bound to a function's name
    when other variables must wait until after they are initialized before they
    can be used?

1.  How do other languages you know handle local variables that refer to the
    same name in their initializer, like:

    ```lox
    var a = "outer";
    {
      var a = a;
    }
    ```

    Is it a runtime error? Compile error? Allowed? Do they treat global
    variables differently? Do you agree with their choices? Justify your answer.

1.  Extend the resolver to report an error if a local variable is never used.

1.  Our resolver calculates *which* environment the variable is found in, but
    it's still looked up by name in that map. A more efficient environment
    representation would store local variables in an array and look them up by
    index.

    Extend the resolver to associate a unique index for each local variable
    declared in a scope. When resolving a variable access, look up both the
    scope the variable is in and its index and store that. In the interpreter,
    use that to quickly access a variable by its index instead of using a map.

</div>
