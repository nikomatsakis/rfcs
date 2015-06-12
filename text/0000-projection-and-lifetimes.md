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
      | &ℓ T                          // Reference (mut doesn't matter here)
      | R0..Rn+ℓ                      // Object type
      | [T]                           // Slice type
      | for<ℓ0..ℓn> fn(T1..Tn) -> T0  // Function pointer
      | <P0 as Trait<P1..Pn>>::Id     // Projection
    P = ℓ                             // Lifetime reference
      | T                             // Type
    R = for<ℓ0..ℓn> TraitId<P0..Pn>   // Trait reference  
    ℓ = 'x                            // Named lifetime
    

    
### Syntactic definition of the outlives relation

The outlives relation is defined in purely syntactic terms as follows.
These are inference rules written in a primitive ASCII notation. :) As
part of defining the outlives relation, we need to track the set of
lifetimes that are bound within the type we are looking at.  Let's
call that set `Λ=<ℓ0..ℓn>`. We'll start out with a simple
"indirection" rule:

    OutlivesScalar:
      <> ⊢ T: 'a
      --------------------------------------------------
      T: 'a
      
This says that `T: 'a` is true if we can judge that `T: 'a` when no
lifetimes are bound. OK, with that preliminary out of the way, let's
look at The first set of rules.

#### Simple cases of the outlives

These rules cover the simple cases, where no type parameters or
projections are involved:

    OutlivesScalar:
      --------------------------------------------------
      Λ ⊢ scalar: 'a

    OutlivesNominalType:
      ∀i. P[i]: 'a
      --------------------------------------------------
      Λ ⊢ Id<P0..Pn>: 'a

    OutlivesReference:
      Λ ⊢ 'x: 'a
      Λ ⊢ T: 'a
      --------------------------------------------------
      Λ ⊢ &'x T: 'a

    OutlivesObject:
      ∀i. Λ ⊢ R[i]: 'a
      Λ ⊢ 'x: 'a
      --------------------------------------------------
      Λ ⊢ R0..Rn+'x: 'a

    OutlivesFunction:
      ∀i. Λ,ℓ.. ⊢ Ti: 'a
      --------------------------------------------------
      Λ ⊢ for<ℓ..> fn(T1..Tn) -> T0

    OutlivesTraitRef:
      ∀i. Λ,ℓ.. ⊢ Pi: 'a
      --------------------------------------------------
      Λ ⊢ for<ℓ..> TraitId<P0..Pn>: 'a      
      
#### Outlives for lifetimes

The outlives relation for lifetimes depends on whether the lifetime in
question was bound within a type or not. In the usual case, we decide
the relationship between two lifetimes by consulting the environment.
Lifetimes representing scopes within the current fn have a
relationship derived from the code itself, lifetime parameters have
relationships defined by where-clauses and implied bounds:

      'x not in Λ
      ('x: 'a) in Env
      --------------------------------------------------
      Λ ⊢ 'x: 'a
      
For higher-ranked lifetimes, we simply ignore the relation, since the
lifetime is not yet known. This means for example that `fn<'a> fn(&'a
i32): 'x` holds, even though we do not yet know what region `'a` is
(and in fact it may be instantiated many times with different values
on each call to the fn).
      
      'x in Λ
      --------------------------------------------------
      Λ ⊢ 'x: 'a

#### Outlives for type parameters

For type parameters, the only way to draw "outlives" conclusions is to
find information in the environment (which is being threaded
implicitly here, since it is never modified). In terms of a Rust
program, this means both explicit where-clauses and implied bounds
derived from the signature (discussed below).

    OutlivesTypeParameterEnv:
      X: 'a in Env
      --------------------------------------------------
      Λ ⊢ X: 'a


#### Outlives for projections

Projections have the most possibilities. First, we may find
information in the environment, as with type parameters, but we can
also consult the trait definition to find bounds (consider an
associated type declared like `type Foo: 'static`). These rule only
apply if there are no higher-ranked lifetimes in the projection; for
simplicity's sake, we encode that by requiring an empty list of
higher-ranked lifetimes. (This is somewhat stricter than necessary,
but reflects the behavior of my prototype implementation.)

    OutlivesProjectionEnv:
      <P0 as Trait<P1..Pn>>::Id: 'a in Env
      --------------------------------------------------
      <> ⊢ <P0 as Trait<P1..Pn>>::Id: 'a

    OutlivesProjectionTraitDef:
      WC = [Xi => Pi] WhereClauses(Trait) 
      <P0 as Trait<P1..Pn>>::Id: 'a in WC
      --------------------------------------------------
      <> ⊢ <P0 as Trait<P1..Pn>>::Id: 'a

All the rules covered so far already exist today. This last rule,
however, is not only new, it is the crucial insight of this RFC.  It
states that if all the components in a projection's trait reference
outlive `'a`, then the projection must outlive `'a`:

    OutlivesProjectionComponents:
      ∀i. Λ ⊢ Pi: 'a
      --------------------------------------------------
      Λ ⊢ <P0 as Trait<P1..Pn>>::Id: 'a

Given the importance of this rule, it's worth spending a bit of time
discussing it in more detail. The following explanation is fairly
informal. A more detailed look can be found in the appendix.

Let's begin with a concrete example of an iterator type, like
`std::vec::Iter<'a,T>`. We are interested in the projection of
`Iterator::Item`:

    <Iter<'a,T> as Iterator>::Item

or, in the more succint (but potentially ambiguous) form:

    Iter<'a,T>::Item

Since I'm going to be talking a lot about this type, let's just call
it `<PROJ>` for now. We would like to determine whether `<PROJ>: 'x` holds.

Now, the easy way to solve `<PROJ>: 'x` would be to normalize `<PROJ>`
by looking at the relevant impl:

```rust
impl<'b,U> Iterator for Iter<'b,U> {
    type Item = &'b U;
    ...
}
```

From this impl, we can conclude that `<PROJ> == &'a T`, and thus
reduce `<PROJ>: 'x` to `&'a T: 'x`, which in turn holds if `'a: 'x`
and `T: 'x` (from the rule `OutlivesReference`).

But often we are in a situation where we can't normalize the
projection. What can we do then? The rule
`OutlivesProjectionComponents` says that if we can conclude that every
lifetime/type parameter `Pi` to the trait reference outlives `'x`,
then we know that a projection from those parameters outlives `'x`. In
our example, the trait reference is `<Iter<'a,T> as Iterator>`, so
that means that if the type `Iter<'a,T>` outlives `'x`, then the
projection `<PROJ>` outlives `'x`. Now, you can see that this
trivially reduces to the same result as the normalization, since
`Iter<'a,T>: 'x` holds if `'a: 'x` and `T: 'x` (from the rule
`OutlivesNominalType`).

OK, so we've seen that applying the rule
`OutlivesProjectionComponents` comes up with the same result as
normalizing (at least in this case), and that's a good sign. But what
is the basis of the rule?

The basis of the rule comes from reasoning about the impl that we used
to do normalization. Let's consider that impl again, but this time
hide the actual type that was specified:

```rust
impl<'b,U> Iterator for Iter<'b,U> {
    type Item = /* <TYPE> */;
    ...
}
```

So when we normalize `<PROJ>`, we obtain the result by applying some
substitution `Θ` to `<TYPE>`. This substitution is a mapping from the
lifetime/type parameters on the impl to some specific values, such
that `<PROJ> == Θ <Iter<'b,U> as Iterator>::Item`. In this case, that
means `Θ` would be `['b => 'a, U => T]` (and of course `<TYPE>` would
be `&'b U`, but we're not supposed to rely on that).

The key idea for the `OutlivesProjectionComponents` is that the only
way that `<TYPE>` can *fail* to outlive `'x` is if either:

- it names some lifetime parameter `'p` where `'p: 'x` does not hold; or,
- it names some type parameter `X` where `X: 'x` does not hold.

Now, the only way that `<TYPE>` can refer to a parameter `P` is if it
is brought in by the substitution `Θ`. So, if we can just show that
all the types/lifetimes that in the range of `Θ` outlive `'x`, then we
know that `Θ <TYPE>` outlives `'x`.

Put yet another way: imagine that you have an impl with *no
parameters*, like:

```rust
impl Iterator for Foo {
    type Item = /* <TYPE> */;
}
```

Clearly, whatever `<TYPE>` is, it can only refer to the lifetime
`'static`.  So clearly `<Foo as Iterator>::Item: 'static` holds. We
know this is true without ever knowing what `<TYPE>` is -- we just
need to see that the trait reference `<Foo as Iterator>` doesn't have
any lifetimes or type parameters in it, and hence the impl cannot
refer to any lifetime or type parameters.

### The WF relation

This section describes the "well-formed" relation. In
[previous RFCs][RFC 192], this was combined with the outlives
relation. We separate it here for reasons that shall become clear when
we discuss WF conditions on impls.

The WF relation is really pretty simple: it just says that a type is
"self-consistent". Typically, this would include validating scoping
(i.e., that you don't refer to a type parameter `X` if you didn't
declare one), but we'll take those basic conditions for granted.

    WfScalar:
      --------------------------------------------------
      scalar WF

    WfParameter:
      --------------------------------------------------
      X WF

    WfNominalType:
      ∀i. Pi Wf            // parameters must be WF,
      C = WhereClauses(Id) // and the conditions declared on Id must hold...
      [P0..Pn] C           // ...after substituting parameters, of course
      --------------------------------------------------
      Id<P0..Pn> WF

    WfReference:
      T WF                 // T must be WF
      T: 'x                // T must outlive 'x
      --------------------------------------------------
      &'x T WF

    WfTrait:
      ∀i. Ri
      --------------------------------------------------
      R0..Rn+'x WF

    WfSlice:
      T WF
      T: Sized
      --------------------------------------------------
      [T] WF

    WfProjection:
      ∀i. Pi WF
      --------------------------------------------------
      <P0 as Trait<P1..Pn>>::Id WF

The following two rules are interesting. For fn arguments and trait
objects, there may be higher-ranked lifetimes involved. This means
that we cannot (in general) check that the types involved are
well-formed. Therefore, we stop at these boundaries.

    WfFn:
      --------------------------------------------------
      for<ℓ..> fn(T1..Tn) -> T0

    WfObject:
      --------------------------------------------------
      R0..Rn+ℓ WF

The WF of types in a fn signature or trait object is deferred to the
point where the fn is calle or the trait object is used.

It is also reasonable to consider when a where-clause is WF. This
will be needed below:

    WfTraitRef:
      
      --------------------------------------------------
      for<ℓ..> P0: Trait<P1..Pn>

OPTIONS:

- Do not check WF of where-clauses, but defer the check until we are
  *using* a where-clause.
- Is there some kind of weird annoying cycle? Wherein we use the where
  clause, check that it's WF, and find that it is due to itself or
  something?

#### When should we check the WF relation and under what conditions?

Currently the compiler performs WF checking in a somewhat haphazard
way: in some cases (such as impls), it omits checking WF, but in
others (such as fn bodies), it checks WF when it should not have
to. Partly that is due to the fact that the compiler currently
connects the WF and outlives relationship into one thing, rather than
separating them as proposed here.

**Struct/enum declarations.** In a struct/enum declaration, we should
create an initial environment consisting of the where clauses declared
on the struct/enum. No implied bounds are considered. We then check
that (1) the where-clauses are WF and (2) that all field types are WF.

**Trait declarations.** 

# Drawbacks

N/A

# Alternatives

N/A

# Unresolved questions

None.

[RFC 192]: https://github.com/rust-lang/rfcs/blob/master/text/0192-bounds-on-object-and-generic-types.md
[RFC 195]: https://github.com/rust-lang/rfcs/blob/master/text/0195-associated-items.md
[RFC 447]: https://github.com/rust-lang/rfcs/blob/master/text/0447-no-unused-impl-parameters.md
[#23442]: https://github.com/rust-lang/rust/issues/23442
[#24662]: https://github.com/rust-lang/rust/issues/24622
[#22436]: https://github.com/rust-lang/rust/pull/22436
[#22246]: https://github.com/rust-lang/rust/issues/22246
[adapted]: https://github.com/rust-lang/rust/issues/22246#issuecomment-74186523
[#22077]: https://github.com/rust-lang/rust/issues/22077
[#24461]: https://github.com/rust-lang/rust/pull/24461

# Appendix

The informal explanation glossed over some details. Let's go from
specific examples to a more general one, where the projection `<PROJ>`
is (here, `P` can be a lifetime or type parameter as appropriate):

    <P0 as Trait<P1...Pn>>::Id
    
The relevant impl is:

```rust
impl<X0..Xn> Trait<Q1..Qn> for Q0 {
    type Id = T;
}    
```

Here again, `X` can be a lifetime or type parameter name, and `Q` can
be any lifetime or type parameter.

Let `Θ` be some substitution `[Xi => Ri]` such that `∀i. Θ Qi ==
Pi`. Then the normalized form of `<PROJ>` is `Θ T`. This substitution
must exist because otherwise this impl is not suitable for the
projection `<PROJ>`. Note that because trait matching is invariant,
the types must be exactly equal.

[RFC 447] and [#24461] require that a parameter `Xi` can only appear
in `T` if it is *constrained* by the trait reference `<Q0 as
Trait<Q1..Qn>>`. The full definition of *constrained* appears below,
but informally it means roughly that `Xi` appears in `Q0..Qn`
somewhere. Let's call the constrained set of parameters
`Constrained(Q0..Qn)`.

Recall the rule `OutlivesProjectionComponents`:

    OutlivesProjectionComponents:
      ∀i. Pi: 'a
      --------------------------------------------------
      <P0 as Trait<P1..Pn>>::Id: 'a

We aim to show that `∀i. Pi: 'a` implies `(Θ T): 'a`, which
implies that this rule is a sound approximation for normalization.
The high-level outline of the argument is as follows:

1. First, we show that 

Our first lemma says that if you have a type `T: 'a`

1. We want to show something like this:
   - We know that the types in the projection outlive `'a`
   - From this we conclude that each *part* of those types outlive `'a` individually
   - From this we conclude that the parts `Ri` each outlive `'a`
   - Then we show that if you have a type and you substitute in parts that outlive `'a`,
     it outlives `'a`
     
     


Our first lemma relates the range of the substitution `Θ` 

    Given:
      ∀i. P: 'a
      [Xi => Ri] Q == P
    Then:
      
      
  Given: T0 = [X => T1] T2 where X in constrained(T2)
  Then: (T0: 'a) => (T1: 'a)




If you look at the "outlives" rules, you can see that the only way for
`<TYPE>: 'x` to fail is if `<TYPE>` contains either a lifetime (other
than `'static`) or a type parameter. Based on the reasoning above, we
know that the only lifetime/type parameters in `<TYPE>` were
constrained by the trait reference. Put another way, the normalized
projection `<PROJ>` is defined by some substitution like:

    






The rule `ProjectionComponents` says that that if we know 

The key
idea is that if we have a projection like `<SomeType as
Iterator>::Item` (or, more succintly, `SomeType::Item`), knowing that
`SomeType: 'a` also tells us that `SomeType::Item: 'a`. Put another way,
if the iterator outlives 


concludes that `<P0 as Trait<P1..Pn>>::Id: 'a` by considering all of
its components `Pi`. To see why this makes sense, consider the impl
that defines the type `Id`:

```rust
impl<A..Z> Trait<Q1..Qn> for Q0 {
    type Id = ...;
}
<```

In [RFC 192], the outlives relation was defined in terms

- Simplify (and strengthen) the `T: 'a` relation to be purely syntactic
- Specify various inference rules for the outlives relation applied to projections
  - `<P0 as Trait<P1..Pn>>::Item: 'a`
- Enforce 

Lemma:

  Given: T0 = [X => T1] T2 where X in constrained(T2)
  Then: (T0: 'a) => (T1: 'a)

