- Feature Name: N/A
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Simplify and clarify the rules on well-formedness with the aim of
correcting various soundness errors.

# Motivation

### Introduction

[RFC 192] introduced the outlives relation `T: 'a` and described the
rules that are used to decide when one type outlives a lifetime. In
particular, the RFC describes rules that govern how the compiler
determines what kind of borrowed data may be "hidden" by a generic
type. For example, given this function signature:

```rust
fn foo<'a,I>(x: &'a I)
    where I: Iterator
{ ... }
```

the compiler is able to automatically determine that:

- all borrowed content in the type `I` outlives the function body;
- all borrowed content in the type `I` outlives the lifetime `'a`.

When associated types were introduced in [RFC 195], some new rules
were required to decide when an "outlives relation" involving a
projection (e.g., `I::Item: 'a`) should hold. The initial rules were
[very conservative][#22246]. This led to the rules from [RFC 192]
being adapted [adapted] to cover associated type projections like
`I::Item`. Unfortunately, these adapted rules are not ideal, and can
still lead to [annoying errors in some situations][#23442]. Finding a
better solution has been on the agenda for some time.

Simultaneously, we realized in [#24662] that the compiler had a bug
that caused it erroneously assume that every projection like `I::Item`
outlived the current function body, just as it assumes that type
parameters like `I` outlive the current function body. **This bug can
lead to unsound behavior.** Unfortunately, simply implementing the
naive fix for #24662 exacerbates the shortcomings of the current rules
for projections, causing widespread compilation failures in all sorts
of reasonable and obviously correct code.

This RFC describes modifications to the type system that aim to both
restore soundness and to make working with associated types more
convenient in some situations. The changes are **largely but not
completely backwards compatible**. We have run tests against crates.io
and the RFC includes examples of the kinds of compilation failures
that can occur.

### Effect of the changes

The detailed design section of this RFC dives into the details of the
type-system rules.  This section is intended to summarize the effect
of these changes on existing Rust code.  We cover examples of code
that used to compile but now does not, and describe what changes are
needed to make it compile again. We also cover examples of code that
previously failed to compile but now does.

#### Understanding the meaning of `T: 'a` 

This RFC modifies the definition of `<Type>: 'a` to be rather simpler
than it was before. Essentially, the rules say that `<Type>: 'a` holds
if every lifetime that appears in `<Type>` outlives `'a`. So if you
have a type like `Box<&'x i32>`, then `Box<&'x i32>: 'a` holds if `'x:
'a`.

Examples:

- `Foo<'x>: 'a` where `Foo` is some struct. This relationship holds if `'x: 'a`.
- `fn(&'x i32): 'a` holds if `'x: 'a`.
- `for<'x> fn(&'x &'y i32): 'a` holds if `'y: 'a`. Note that `'x: 'a`
  is not required. This is because `'x` is bound within the type
  itself and is not a specific lifetime yet.

#### Fns, trait objects, and lifetimes

Previously, the rule for when `<Type>: 'a` holds was more subtle. It reasoned
about what kinds of data was actually reachable, and ensured that all of this
data outlived `'a`, but it did not impose restrictions on parts of the type
that don't represent reachable data.

This is easiest to see with a fn type. Previously, `fn(&'x i32):
'static` would *always* hold, even though `'x: 'static` is not
true. The reason for this was that the fn pointer itself doesn't
*contain* any data at all, and certainly no data with the lifetime
`'x`. Rather, when called, it will be *given* data with that lifetime.
The newer rules don't draw such fine distinctions. Similar logic
applies to type parameters on trait objects.

The main place that these changes break normal code is in struct definitions
like the following:

```rust
// no longer legal
struct SomeStruct<'a,T> {
    &'a SomeTrait<T>
}
```

This struct now requires a `T:'a` declaration, because `SomeTrait<T>`
appears inside a borrowed reference of lifetime `'a` and thus
`SomeTrait<T>: 'a` must hold:

```rust
// the replacement:
struct SomeStruct<'a,T:'a> {
    &'a SomeTrait<T>
}
```

# Detailed design

This section dives into detail on the proposed type rules.

### A little type grammar

We extend the type grammar from [RFC 192] with projections and slice
types:

    T = scalar (i32, u32, ...)        // Boring stuff
      | X                             // Type variable
      | Id<P0..Pn>                    // Nominal type (struct, enum)
      | &'x T                         // Reference (mut doesn't matter here)
      | R0..Rn+'x                     // Object type
      | [T]                           // Slice type
      | fn(T1..Tn) -> T0              // Function pointer
      | <P0 as Trait<P1..Pn>>::Id     // Projection
    P = 'x                            // Lifetime parameter
      | T                             // Type parameter
    R = TraitId<P0..Pn>               // Trait reference  
    
### Syntactic definition of the outlives relation

The outlives relation is defined in purely syntactic terms as follows.
These are inference rules written in a primitive ASCII notation. :)

The first set of rules cover the simple cases, where no type
parameters or projections are involved:

    Scalar:
      --------------------------------------------------
      Env |- scalar: 'a

    NominalType:
      P[i]: 'a for i in 0..n
      --------------------------------------------------
      Id<P0..Pn>: 'a

    Reference:
      'x: 'a
      T: 'a
      --------------------------------------------------
      &'x T: 'a

    Object:
      R[i]: 'a for i in 0..n
      'x: 'a
      --------------------------------------------------
      R0..Rn+'x: 'a

    Function:
      Ti: 'a for i in 0..n
      --------------------------------------------------
      fn(T1..Tn) -> T0

    TraitRef:
      Ti: 'a for i in 0..n
      --------------------------------------------------
      TraitId<P0..Pn>: 'a      

    TraitRef:
      Ti: 'a for i in 0..n
      --------------------------------------------------
      TraitId<P0..Pn>: 'a      

The more interesting rules 

    Scalar:
      --------------------------------------------------
      Env |- scalar: 'a


In [RFC 192], the outlives relation was defined in terms

- Simplify (and strengthen) the `T: 'a` relation to be purely syntactic
- Specify various inference rules for the outlives relation applied to projections
  - `<P0 as Trait<P1..Pn>>::Item: 'a`
- Enforce   



# Drawbacks

Why should we *not* do this?

# Alternatives

What other designs have been considered? What is the impact of not doing this?

# Unresolved questions

What parts of the design are still TBD?

[RFC 192]: https://github.com/rust-lang/rfcs/blob/master/text/0192-bounds-on-object-and-generic-types.md
[RFC 195]: https://github.com/rust-lang/rfcs/blob/master/text/0195-associated-items.md
[#23442]: https://github.com/rust-lang/rust/issues/23442
[#24662]: https://github.com/rust-lang/rust/issues/24622
[#22436]: https://github.com/rust-lang/rust/pull/22436
[#22246]: https://github.com/rust-lang/rust/issues/22246
[adapted]: https://github.com/rust-lang/rust/issues/22246#issuecomment-74186523
