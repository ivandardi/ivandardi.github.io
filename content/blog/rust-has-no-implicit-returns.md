+++
title = "Rust has no implicit return (DRAFT)"
date = 2024-10-15
updated = 2024-10-15
description = "How returns work in Rust, and how implicit returns are a myth."

[taxonomies]
tags = ["rust", "implicit return"]
+++


One of the things that makes Rust unique compared to other languages is its approach to returns. If you've come from languages like Python or JavaScript, you might be used to always using the `return` keyword whenever you want to return something. However, when it comes to Rust, you can return values from a function without using the `return` keyword. This behavior is usually known as "implicit return" and can be very confusing for some people.

In this blog post I'll explain my own mental framework for what "implicit return" means, and the derivations that come from it. But first, a disclaimer: what I say here can sometimes not be 100% accurate. Sometimes what I write will be missing some information that I deem unnecessary for the explanation, sometimes it'll have a simplified interpretation of what is actually happening, and sometimes I'll be wrong due to pure ignorance. Please excuse me when that happens, and I appreciate feedback whenever something needs fixing.

With that out of the way, let's get to the explanation!


## Expressions Everywhere

In Rust, I like to think that everything is an expression. Sure, that may not be completely true, since macros, language items and other things exist. But generally speaking, the simplification that everything is an expression can be useful.

A special mention here are `let` statements. As the name suggests, they are statements instead of expressions. However, I like to see them as assignment expressions that evaluate to the unit type `()`. In C, for example, you can write `a = b = c = 10`. That's because `c = 10` evaluates to `10`, and then `b = c = 10` turns into `b = 10`, which also gets evaluated to `10`, and so on. That doesn't work in Rust because the the `let` statement evaluates to `()`, and so you'd assign one variable the value `10` and the next one the value `()`. Again, this is simplifying a lot, but it helps build the mental framework for what comes next.

If everything is an expression, then what are functions? In simple terms, a function consists of a sequence of expressions which are separated by semicolons.

Expressions usually come in expression blocks. For example:

```rust
fn expression_blocks() -> i32 {
    let result = {
        let a = 2;
        let b = 10;
        a * b
    };

    result
}
```

The code that is being assigned to the `result` variable is what I call an expression block. It contains a sequence of expressions, separated by semicolons, wrapped around in curly braces `{}`. Expression blocks, or just blocks in common Rust terms, are always delimited by a pair of curly braces. They can contain 0, 1 or more expressions, and they can also contain other expression blocks.

If you think about it, functions always have a pair of curly braces after the function signature. That is also an expression block! In fact, you can syntactically see functions as:

```rust
fn function_name() -> ReturnType <expression_block>
```

I am aware that they're officially "block expressions" instead of "expression blocks". However, I like visualizing them as blocks—delimited by curly braces `{}`—that contain expressions, which is why I call them that way. And expression blocks are important for this mental framework that I'm building because they have a very important property:

**Expression blocks always evaluate to the last expression in the expression sequence it contains.**

For us to be able to fully understand what that means, we first must look at what semicolons do.


## The Role of the Semicolon (;)

Here's a basic example of a Rust function:

```rust
fn sum(a: i32, b: i32) -> i32 {
    a + b
}
```

In this example, there's no `return` keyword, but the function still returns the value of `a + b`. This works because `a + b` is an expression, and it's the last thing in the expression block of the function. More generally speaking:

**Rust functions always return the evaluation of its expression block**.

Notice the absence of a semicolon (;) at the end of the `a + b` line—this is what "makes" the Rust function return the expression and is known as the "implicit return" that everyone talks about. But how does that really work? That's where semicolons come in.

In Rust, the semicolon (;) doesn't simply mark an "end of statement" like in some other languages. Instead, it's used to separate expressions. Here's a more detailed example to show how this works:

```rust
fn example() -> i32 {
    let x = 5;
    let y = 10;
    x + y;
}
```

At first glance, you might think that this function would "implicitly" return `15` (since `x + y` is `15`)—but it actually returns `()`, which creates a compiler error. Why? Because `x + y;` has a semicolon at the end.

Let's rewrite the expression block in the function in a different way and label each expression in it:

```rust
let x = 5 ; let y = 10 ; x + y ; ε
+-------+   +--------+   +---+   +
    A            B         C     D
```

As explained above, `let` statements—and statements in general—evaluate to the unit type `()`.

Expression `C` evaluates to `15`. However, the value of that expression isn't being saved anywhere, so it just gets discarded.

Now, for expression `D`, I'm borrowing a concept from the literature. If you've read anything about writing compilers, you may have come across the symbol `ε`. That symbol represents an empty string. Note that it's different from the empty string `""`. The `ε` is necessary because the semicolon is an expression _separator_, meaning it must appear with an expression on either side. In the example, we don't have a final expression that the last semicolon can separate from the rest. Therefore, the Rust compiler adds `ε` at the end of the expression list to allow the last semicolon to separate two expressions as intended.

And with all of that in mind, we reach the crucial piece of information: **`ε` gets evaluated to `()`**.

To illustrate that, let's rewrite the expression block, this time replacing the code with the values of their respective evaluations:

```rust
    ()    ;     ()     ;  15   ; ()
+-------+   +--------+   +---+   +
    A            B         C     D
```

We can observe that the statements got evaluated to `()`, the sum expression got evaluated to `15`, and the `ε` got evaluated to `()`. And because it's the last value in the sequence, that's what the Rust function returns. With that in mind, we now know why this compiles:

```rust
fn main() {
    let result: () = example_unit_type();
}

fn example_unit_type() {
    let x = 5;
    let y = 10;
    x + y;
}
```

And for completeness, let's remove the semicolon at the end of the `example` function and do the same steps again:

```rust
fn example_fixed() -> i32 {
    let x = 5;
    let y = 10;
    x + y
}
```

```rust
let x = 5 ; let y = 10 ; x + y
+-------+   +--------+   +---+
    A            B         C
```

```rust
    ()    ;     ()     ;  15
+-------+   +--------+   +---+
    A            B         C
```

Because the last semicolon is gone, all semicolons are already separating two expressions as intended, so there's no need add `ε` at the end of the expression sequence. After evaluation, `15` is the final value of the expression sequence, which then gets returned by the function.


## Final Thoughts

So, what's the takeaway? Rust doesn't have implicit returns because expression blocks evaluate to the last expression it contains, which then get returned by functions. And whenever a semicolon isn't separating two expressions, the Rust compiler appends an `ε`, which gets evaluated to `()`.

This isn't the easiest or the most intuitive rule to learn and remember. It's commonly interpreted as "If you want to return a value, you can remove the semicolon from the last expression and it'll get implicitly returned". However, that way of phrasing it doesn't fully explain what's going on under the hood, which can create a lot of confusion and misconceptions about Rust's "implicit returns". Hopefully the mental framework that I explained in article helps you better understand how returns actually work in Rust.
