+++
title = "Rust has no \"implicit return\""
date = 2024-10-15
updated = 2024-10-15
description = "How returns work in Rust, and how implicit returns are a myth."

[taxonomies]
tags = ["rust", "implicit return"]
+++

# Rust has no "implicit return"

One of the things that makes Rust unique compared to other languages is its approach to returns. If you've come from languages like Python or JavaScript, you might be used to always using the `return` keyword whenever you want to return something. In Rust, however, you don't always need to use the `return` keyword. That behavior ends up being very confusing for some people, and it's usually known as "implicit returns". With that in mind, let's explain what it means and how returns actually work in Rust.


## Expressions Everywhere

In Rust, the building blocks of everything are expressions. There are also statements, but they are just expressions that evaluate to the unit type `()`. In simple terms, a function consists of a sequence of expressions, which are separated by semicolons.

Here's a basic example of a Rust function:

```rust
fn sum(a: i32, b: i32) -> i32 {
    a + b
}
```

In this example, there's no return keyword, but the function still returns the value of `a + b`. This works because `a + b` is an expression, and it's the last thing evaluated in the function. In Rust, **functions always return the value of the last evaluated expression**.

Notice the absence of a semicolon (;) at the end of the `a + b` line. This is what makes the Rust function return the expression. But how does that really work?


## The Role of the Semicolon (;)

The semicolon (;) in Rust doesn't mean "end of statement" like it does in some other languages. Instead, it behaves like an expression separator. Let's look at a more detailed example to see how it works:

```rust
fn example() -> i32 {
    let x = 5;
    let y = 10;
    x + y;
}
```

At first glance, you might think that this function would return `15` (since `x + y` is `15`), but it actually returns `()`, which creates a compiler error. Why? Because `x + y;` has a semicolon at the end.

For us to understand how that happens,  let's rewrite the expression block in that function in a different way and label each expression in it:


```rust
let x = 5 ; let y = 10 ; x + y ; ε
+-------+   +--------+   +---+   +
    A            B         C     D
```

In Rust, a `let` statement is exactly what it says on the tin: a statement. Statements can be seen as expressions that evaluate to `()`. Therefore, in this case, `A` and `B` are statements.

When it comes to expression `C`, we can see that it is not a statement. It evaluates to `15`, therefore making it an expression. However, the value of that expression isn't being saved anywhere, so it just gets discarded.

Now, for expression `D`, I'm borrowing a concept from compiler books. If you've read any books about writing compilers, you may have come across the symbol `ε`. That symbol represents an empty string. Note that it's different from the empty string `""`. The `ε` is necessary because the semicolon is an expression separator. That is, a semicolon always needs to be between two expressions so that it can separate them. A semicolon-terminated expression list, just like in the example, doesn't have a final expression that the last semicolon can separate from the rest. Therefore, the Rust compiler adds `ε` at the end of the expression list to allow the last semicolon to separate two expressions as intended.

And with all of that in mind, we reach the final piece of information: **`ε` gets evaluated to `()`**.

To illustrate that, let's rewrite the expression block again, but with the values of each expression:

```rust
    ()    ;     ()     ;  15   ; ()
+-------+   +--------+   +---+   +
    A            B         C     D
```

As we can see, the statements got evaluated to `()`, the expression got evaluated to `15`, and the `ε` got evaluated to `()`. And since `()` is the value of the last evaluated expression, that's what the Rust function ends up returning.

Let's remove the semicolon at the end of the function.

```rust
fn example() -> i32 {
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

With that last semicolon gone, the Rust compiler doesn't add `ε` at the end of the expression list, and `15` is now the last value of the expression list, which gets returned from the function because functions always return the value of the last evaluated expression.


## Generalizing beyond functions

That behavior applies to everything, not just functions. Here's an example that doesn't use functions.

```rust
fn main() {
    let a = 3.0;
    let b = 4.0;


    let x = {
        let a_squared: f32 = a * a;
        let b_squared: f32 = b * b;

        (a_squared + b_squared).sqrt()
    };
}
```

The `let x` statement is followed by an expression block. An expression block is a list of semicolon-separated expressions surrounded by `{}` brackets, and they behave the same way as explained above. In fact, you can syntactically see functions as:

```rust
fn function_name() -> ReturnType <expression_block>
```


## Final Thoughts

So, what's the takeaway? Rust doesn't have implicit returns because it always returns the result of the last evaluated expression in an expression block. And if there isn't a final expression in an expression list inside an expression block, Rust fills it in with `ε`, which gets evaluated to `()`.

The above explanation isn't the easiest or the most intuitive to learn, however, and so most people internalize it as "If you want to return a value, you can remove the semicolon from the last expression and it'll get implicitly returned". However, that way of phrasing it doesn't fully explain what's going on under the hood, which creates a lot of confusion and misconceptions about Rust's "implicit returns". Hopefully this article helps you better understand how returns actually work in Rust.
