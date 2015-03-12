- Feature Name: try
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Add syntax for working with `Result` types in a flexible and
convenient fashion. The keywords `try` and `catch` and the operator
`?` are introduced. The `?` operator is a more flexible variant of
today's `try!` macro: it destructures a `Result`, yielding either the
`Ok` value or propagating an `Err`. Unlike today's macro, `?`
propagates the `Err` result only as far as the innermost enclosing
`try`; if there is no enclosing `try`, then the `Err` result is
returned from the `fn` as today. The `catch` keyword makes it easy to
extend a `try` and handle results that occur within.

This RFC is explicitly *staged*: a minimal impl of the `?` operator
can be done immediately which is equivalent to today's `try!`
work. This would be unfeature-gated faster (ideally, in time for the
1.0 release). The `try`/`catch` semantics can be implemented on a
slower time-scale, and in particular need not be finalized for 1.0.

# Motivation

Why are we doing this? What use cases does it support? What is the expected outcome?

# Detailed design

## Syntax changes

A postfix operator `?` is introduced. This operator has equal
precedence with the other postfix operators (`.`, `()`, and `[]`).

A new `try` expression is introduced. The `try` expression comes in
two varieties (in pseudo-BNF):

    EXPR = ...
         | "try" BLOCK
         | "try" BLOCK "catch" "{" ARMS "}"
         | ...
    BLOCK = "{" <stmts> <expr> "}"
    ARMS = PATTERN => EXPR [, ARM]
         | PATTERN => BLOCK [,] [ARM]

The `try` expression has the same precedence and treatment as `if`
expressions. In particular, if a `try` expression appears in statement
position, it is terminated at the closing brace and does not accept
binary operators. See example below for clarification.

Here are some examples:

**Example 1.**

```rust
let x: Result<_,_> = try { foo()?.bar()?.zed() };
```

**Example 2.**

```rust
fn foo() {
    try {
        do_something()?;
    } catch {
        () => {
            on error
        }
    }
}

fn do_something() -> Result<(),()>
```

## The `?` postfix operator

The `?` postfix operator is used to unpack results and pass errors to
an outer context: it generalizes the `try!` macro that we use
today. To explain `?`, let's first consider the case of a `fn` with no
`try` blocks. In that case, `<expr>?` is exactly equivalent to the
following desugared match statement:

```rust
match IntoResult::into_result(<expr>) {
  Ok(v) => v,
  Err(e) => { return Err(FromError::from_error(e)); }
}
```

This definition is *almost* the same as the `try!` macro; the only
difference is the use of `IntoResult`, which converts the operand of
the `?` operator into a result of some type (covered in detail below).

## The `try` keyword

One of the downsides of the existing `try!` macro is that it does not
offer a convenient way of delimiting the scope of error-handling ---
errors are always returned out of the enclosing function. To aid with
this, this RFC therefore introduces `try` expressions. This section
focuses on simple `try` expressions that do not try to handle the
error; the next section how a `try` expression works when it contains
the optional `catch` clauses.

Previously we saw that the `?` operator, when applied to a result
matching `Err(e)`, caused the `Err(e)` value to be returned out of the
enclosing `fn`. We now amend that definition so that when the `?`
operator is applied to an error inside the scope of a `try`
expression, that `try` expression produces the value `Err(e)`.  `try`
expressions can be nested, in which case the `?` operator affects the
innermost `try` expression.

Here is an example to try and clarify the idea:

```rust
fn something() -> Result<i32,Box<Error>> { .. }

fn foo() -> Result<...,Box<Error>> {
    let x: i32 = something()?; // returns from `foo` on error
    
    let y: Result<i32,Box<Error>> = try {
        let a: i32 = something()?; // breaks out of try expression on error
        let b: i32 = something()?; // same
        a + b
    };

    ...
}
```

As indicated in the comments, the first usage of the `?` operator (in
the definition of `x`) will trigger a return out of the fn `foo` if an
error occurs. The next two uses of the `?` operator, however, are
contained within a `try` expression, and therefore they will only
trigger a break out that expression if an error occurs. If no error
occurs, the `Ok` value is unpacked and return, and hence `a` and `b`
have type `i32`.

As can be seen in the example, the type of an expression `try {
<block> }` is `Result<O,E>`.  Here, the type `O` is the type of the
tail expression from `<block>` (in the example, that is `i32`). The
type `E` is the error type: it will be determined by inference, but is
typically the type of errors produced by uses of the `?` operator
(note that it does not *have* to be equal to those values, because of
the possibility for `FromError` to adapt the error type.

In terms of the implementation, fresh type variables will be
introduced for the `O` and `E` types upon encountering a `try`
expression.  Those variables are unified as normal. In some cases type
annotations may be needed to guide the `FromError` instantiation, but
frequently the type can also be determined from context.

The combination of `try` and `?` serves as a general replacement for
the `and_then` method of `Result`, as shown here:

```rust
<x>.and_then(|x| <y>).and_then(|y| <z>) ==>

    try {
        let x = <x>?;
        let y = <y>?;
        <z>
    }
```

(This is very similar to how `try!` replaces uses of `and_then` where
the final result is returned out of the fn.) Similarly, the
`<x>.map(|x| <y>)` method can be written as follows `try { let x =
<x>?; Ok(<y>) }`.

## Catching errors with the `catch` keyword

The `try` keyword can optionally be extended with a `catch` clause.
This is used for those cases where you wish to handle errors and not
propagate them further out. The type of a `try/catch` expression is
therefore not a `Result` (because the error has been handled) but
rather just the type of the `<block>` itself. The `catch` clause
exactly desugars into a `match` statement wrapping the `try`.

Here is an example:

```rust
try {
    let x: i32 = <x>?;
    let y: i32 = <y>?;
    x + y
} catch {
    error => {
        report_error(error);
        -1
    }
}
```

In this case, `<x>` and `<y>` are evaluated. If any errors occur,
control-flow will branch immediately to the catch block, where the
error is matched and reported. In such an error case, the final result
of the `try/catch` expression will be `-1`. If no errors occur, the
result is `x+y`. The types of all arms in the `catch` clause and the
type of the body must agree.

For reference, the code above can be desugared into a `match` and a
`try` expression:

```rust
match (
    try {
        let x: i32 = <x>?;
        let y: i32 = <y>?;
        x + y
    }
) {
    Ok(v) => v,
    Err(error) => {
        report_error(error);
        -1
    }
}
```

These two snippets are exactly equivalent.

## IntoResult

The `IntoResult` trait is used to convert the operand of the `?`
operator into a `Result`. It is fairly straightforward. What follows
is the trait definition and some sample impls. Note that the impls for
`Option` and `bool` are somewhat arbitrary and intended to capture the
most common usage. For example, we've chosen `true` to represent `Ok`,
but it could as well be the opposite. Similarly, we've chosen `Some`
to represent `Ok` for `Option`, but it's certainly plausible to return
an `Option<ErrorCode>` instead. That's not really a problem: these
impls capture the most common case, but users can manually convert an
option or bool into a `Result` if they prefer a different convention.

```rust
trait IntoResult<O,E> {
    fn into_result(self) -> Result<O,E>;
}

impl IntoResult<O,E> for Result<O,E> {
    fn into_result(self) -> Result<O,E> {
        self
    }
}

impl IntoResult<O,()> for Option<O> {
    fn into_result(self) -> Result<O,()> {
        match self { Some(v) => Ok(v), None => Err(()) }
    }
}

impl IntoResult<(),()> for bool {
    fn into_result(self) -> Result<O,E> {
        if self {Ok(())} else {Err(())}
    }
}
```

## Staging with respect to 1.0

This RFC is intended to be staged with respect to the 1.0 release. In
particular, the `?` sugar can be implemented now (and in fact,
[has been implemented][23040], though the implementation does not
include the `IntoResult` trait). The `try` and `try/catch` expressions
can then be implemented afterwards. (Note that `try` cannot be fully
desugared into existing syntax; so implementing `try` will be the most
involved part of the process.)

Note though that we could also choose to keep the entire syntax
feature-gated, though this would require finessing the `try` keyword,
as presently keywords cannot be used in macro names. (We intend to
introduce some mechanism for opting into new keywords, though, so this
is not considered a blocker.)

[23040]: https://github.com/rust-lang/rust/pull/23040

# Drawbacks

Why should we *not* do this?

# Alternatives

Some form of monadic notation.

Accept this RFC, but do not replace the `try!` macro.

# Unresolved questions

- What are the precise set of `IntoResult` implementations we should support?

