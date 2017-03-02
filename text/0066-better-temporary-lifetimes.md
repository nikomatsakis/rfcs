- Start Date: 2014-05-04
- RFC PR: [rust-lang/rfcs#66](https://github.com/rust-lang/rfcs/pull/66)
- Rust Issue: [rust-lang/rust#15023](https://github.com/rust-lang/rust/issues/15023)

# Summary

Temporaries live for the enclosing block when found in a let-binding. This only
holds when the reference to the temporary is taken directly. This logic should
be extended to extend the cleanup scope of any temporary whose lifetime ends up
in the let-binding.

For example, the following doesn't work now, but should:

```rust
use std::os;

fn main() {
	let x = os::args().slice_from(1);
	println!("{}", x);
}
```

# Motivation

It is common in Rust code for the compiler to create "temporary values". For example,
an expression like `foo(&bar())` might be more explicitly written as follows:

```rust
let x = bar();
foo(&x)
```

Because the compiler is introducing these temporaries, it raises the
question of when these temporary values ought to be dropped. Ordinary
named values are always dropped (in reverse order of their
introduction) at the end of the block in which they are defined; but
it is often convenient for temporary values to be dropped much sooner.

The current rules regarding temporaries originated before RFCs were
introduced and have not been clearly written out, except for
[some comments in the source code][comment]. The rules (which we will
describe shortly) are based on **purely syntactic criteria**, and they
were derived based on the reasoning described in
[Rust issue #3511][3511] as well as two separate blog posts ([1],
[2]).

[comment]: https://github.com/rust-lang/rust/blob/8430042a49bbd49bbb4fbc54f457feb5fb614f56/src/librustc/middle/region.rs#L898-L957
[1]: http://smallcultfollowing.com/babysteps/blog/2012/09/15/rvalue-lifetimes/
[2]: http://smallcultfollowing.com/babysteps/blog/2014/01/09/rvalue-lifetimes-in-rust/
[3511]: https://github.com/rust-lang/rust/issues/3511

At the time that the blog post was written, the major controversy was
between using local, syntactic criteria -- as was eventually decided
-- or settling on some rules that are driven by the region inference
in the compiler. In short, the argument for inference is that the
compiler can figure out how long the temporary **needs** to live, and
hence we can automatically drop it sometime after that. Ultimately,
though, it was decided that basing execution order on an inference
pass was unwise, because it would make the execution unpredictable and
would hinder efforts to improve the inference (this was a wise
decision: for example, the current attempts to adopt "non-lexical
lifetimes" would be greatly complicated if temporary lifetimes were
related to region inference).

However, in the time since, it has also become clear that the current
rules can also be somewhat awkward. Users have to introduce far more
explicit temporaries than seems necessary, which is an ergonomic
problem, but can also complicate learning the language (one more thing
to get wrong).

This RFC proposes a hybrid rule. Instead of using purely local,
syntactic criteria to decide temporary lifetimes, the rules are made
somewhat broader so that they also take into account the signatures of
functions that are being called. The results still do not require
running region inference, however, or using any kind of advanced
analysis: in essense, they are still largely syntactic in nature,
although they do require us to know the types of functions that are
being called.

### The current rules

Intuitively, the current rules say something like this: temporaries
will be dropped at the end of the innermost enclosing statement,
unless the compiler can "obviously see" that they are going to get
stored into a variable.

So, consider this example of code:

```rust
{
    let v = &map.borrow_mut().get(22).clone();
    ...
}    
```

There are in fact two temporaries involved here (actually, there are
more, but they don't add anything to the example so I'll leave the
others out). If we spelled those temporaries out, it might look like
this. I'll insert explicit calls to `mem::drop` to show where the
compiler will drop the various temporaries:

```rust
{
    // the statement `let v = &map.borrow_mut().get(22).clone()`:
    let tmp0 = map.borrow_mut();
    let tmp1 = tmp0.get(22).clone();
    let v = &tmp1;
    mem::drop(tmp0);
    
    // other statements:
    ...
    
    // as we exit the block, we drop variables and temporaries
    // in reverse order of how they were defined:
    mem::drop(v);
    mem::drop(tmp1);
}    
```

As you can see, the first temporary (`tmp0`) winds up getting dropped
as the first statement completed. This is basically the default
behavior: we will drop all temporaries as soon as the innermost
statement completes (as well as a few other cases, like conditionally
executed code).

The second temporary `tmp1`, however, lives until the end of the
block. `tpm1` is introduced because of the
`&map.borrow_mut().get(22).clone()` expression: in particular,
`map.borrow_mut().get(22).clone()` is not an "lvalue" (that is, it
does not evaluate to a memory location, but rather a
value). Therefore, we cannot create a reference to it, so we must
create a temporary to store that value. In this case, the compiler can
see **syntactically** that the reference that results is going to be
stored into `v`; if we dropped `tmp0` at the same time as `tmp1`, it
is obvious that this program would not compile. Therefore, the
compiler chooses to **extend the lifetime of this temporary to the end
of the enclosing block**.

The next two sections will define these two concepts more
precisely. We specify the "normal" behavior for temporaries using
**destruction scopes** (these are the rules that apply to `tmp0`). We
then specify the rules to decide which temporaries will have an
**extended lifetime** (these are the rules that apply to `tmp1`).

#### Destruction scopes 

The default behavior for a temporary created by some expression E is
that it will be dropped as we exit the innermost enclosing
**destruction scope** that contains E. These destruction scopes are
attached to the following AST nodes:

- The scope for a block `{ statement[0]; ...; statement[n]; tail_expression }`.
- The scope for each statement `statement[i]` in a block.
- Conditionally executed code:
  - `if` and `while` conditions
  - match arms and guards
  - `for` and `loop` bodies
  - the right-hand side of the short-circuiting operators `&&` or `||`

##### Scope tree

In general, the execution of destructors in Rust is driven by *scopes*
(these scopes are also used to define lifetimes at present). This RFC
will not attempt to define the scope tree in full, but we will define
it in part (most of it, really). The basic idea that there is a scope
tree derived from the AST, but in some cases individual nodes wind up
with multiple associated scopes, or we define scopes that do not have
clear nodes in the AST tree.

Let's return to our example, elaborated slightly:

```rust
{
    let v = &map.borrow_mut().get(22).clone();
    let foo = use(v);
    ...
    use(foo)
}    
```

This example will define the following scopes, nested as shown (the
names are meant to mirror those used in the source, and probably could
use to be better chosen):

- Destruction scope for the block as a whole
  - Miscellaneous scope for the block as a whole
    - Remainder scope covering `let v = ...;` and what comes after
      - Destruction scope for the `let v = ...;` statement
        - Miscellaneous scope for the `let v = ...;` statement
          - Miscellaneous scope for the `&map.borrow_mut().get(22).clone()` expression
            - Miscellaneous scope for the `map.borrow_mut().get(22).clone()` expression
              - Miscellaneous scope for the `map.borrow_mut().get(22)` expression
                - Miscellaneous scope for the `map.borrow_mut()` expression
                  - *and so forth*
                - Miscellaneous scope for the `22` expression
      - Remainder scope coverting `let foo = use(v);` and what comes after
        - Destruction scope for the `let foo = use(v);` statement
          - Miscellaneous scope for the `let foo = use(v);` statement
              - Miscellaneous scope for the `use(v)` expression
                - Miscellaneous scope for the `use` expression
                - Miscellaneous scope for the `v` expression
        - *remainder scopes for any elided statements in the `...` section*
          - Miscellaneous scope for the `use(foo)` expression in the block tail
            - Miscellaneous scope for the `use` expression
            - Miscellaneous scope for the `foo` expression

Some interesting things to note:

- Each statement in a block has a "remainder" scope that covers that
  statement as well as subsequent statements (those familiar with ML
  may recognize this as corresponding to something like `let x =
  initializer in remainder`, but not that the remainder also covers
  the initializer).
- Some nodes also have **destruction scopes**. In this example, the
  only such nodes are the block and the statements, since there is no
  conditional execution.
- Otherwise, the tree consists of "miscellaneous" scopes that follow
  the structure of the AST.
  
Based on this, then, one can define the **default destruction scope**
for an expression E by starting with the miscellaneous scope for E and
walking up the scope tree until a destruction scope is encountered.
We then say that the default destruction scope for a temporary T is
the default destruction scope of the expression that defines its value.

##### Running example

So, for the temporary `tmp0`, its value is defined by
`map.borrow_mut().get(22).clone()`, hence we start from the
corresponding miscellaneous scope and walk upwards until we find a
destruction scope. The first such scope is the destruction scope for
the `let v = ...;` statement, and therefore `tmp0` will be dropped as
we exit that scope.

#### Extended temporaries

The next step is to define those temporaries whose lifetime we wish to
**extend**. These rules are defined syntactically by matching over the
Rust AST. For any temporary whose lifetime is "extended", it will
always be extended so that the temporary is dropped in the destruction
scope for the innermost block.

The intuition is that we wish to extend the lifetime of temporaries
that we can syntactically see will be assigned into a local variable.
In our running example, that was because the temporary resulted from
an `&`-expression being assigned into a variable (i.e., `let v =
&<expr>`). However, it could also arise from a ref pattern (i.e., `let
ref v = <expr>`).

More specifically, we say that temporaries are extended based on two
"parameterized" grammars.

#### The "extended expression" grammar

The first grammar, `EE`, matches a set of expressions. The grammar
includes a "hole", which we represent using the unicode character `○`,
where some arbitrary rvalue can go.  The idea is that the grammar
matches some expression if you can fill the hole `○` with an rvalue to
create a match (the `...` sections are assumed to match arbitrary
content). Every rvalue that can correspond to the `○` is therefore a
match.

```
EE = & ○
   | & EE
   | StructName { ..., f: EE, ... }
   | [ ..., EE, ... ]
   | ( ..., EE, ... )
   | EE as T
   | ( EE )
```

Let's work through some example expressions. For each expression,
we'll show the rvalue(s) that might match the the hole `○`:

- `&foo()` -- the `○` could be `foo()`
- `(&foo(), &bar())` -- both `foo()` and `bar()` are covered by the hole
- `Struct { f: foo(), g: bar() }` -- both `foo()` and `bar()` are covered by the hole
- `&foo().bar` -- `foo().bar` is covered by the hole, but `foo()` is not

#### 

Temporary lifetimes are a bit confusing right now. Sometimes you can
keep references to them, and sometimes you get the dreaded "borrowed
value does not live long enough" error. Sometimes one operation works
but an equivalent operation errors, e.g. autoref of `~[T]` to `&[T]`
works but calling `.as_slice()` doesn't. In general it feels as though
the compiler is simply being overly restrictive when it decides the
temporary doesn't live long enough.

# Detailed design

When a reference to a temporary is passed to a function (either as a regular
argument or as the `self` argument of a method), and the function returns a
value with the same lifetime as the temporary reference, the lifetime of the
temporary should be extended the same way it would if the function was not
invoked.

For example, `~[T].as_slice()` takes `&'a self` and returns `&'a [T]`. Calling
`as_slice()` on a temporary of type `~[T]` will implicitly take a reference
`&'a ~[T]` and return a value `&'a [T]` This return value should be considered
to extend the lifetime of the `~[T]` temporary just as taking an explicit
reference (and skipping the method call) would.

# Drawbacks

I can't think of any drawbacks.

# Alternatives

Don't do this. We live with the surprising borrowck errors and the ugly workarounds that look like

```rust
let x = os::args();
let x = x.slice_from(1);
```

# Unresolved questions

None that I know of.
