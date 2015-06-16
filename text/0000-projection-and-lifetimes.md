- Feature Name: N/A
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Type system changes to address the outlives relation with respect to
projections. The current implementation can be both unsound ([#24662])
and inconvenient ([#23442]).

- Simplify the outlives relation to be syntactically based
- Specify improved rules for the outlives relation and projections

The proposes changes here are backwards incompatible; this is
justified by the need to fix a soundness problem. The impact has been
evaluated and found to be quite minimal. Moreover, by simplifying and
improving the rules, the changes also make the compiler accept
reasonable code that currenty fails to compile (e.g. the example from
[#23442]).

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
being [adapted] to cover associated type projections like
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

This section is intended to summarize the effect of these changes on
existing Rust code. (Detailed coverage of the rules can be found
below, in the detailed design section.) We cover examples of code that
used to compile but now does not, and describe what changes are needed
to make it compile again. We also cover examples of code that
previously failed to compile but now does.

#### Understanding the meaning of `T: 'a` 

This RFC modifies the definition of `<Type>: 'a` to be rather simpler
than it was before. Essentially, the new rules say that `<Type>: 'a`
holds if every type and lifetime parameter that appears in `<Type>`
outlives `'a`. So if you have a type like `Box<&'x Y>`, where `Y` is a
type parameter, then `Box<&'x Y>: 'a` holds if `'x: 'a` and `Y: 'a`.

Examples:

- `Foo<'x>: 'a` where `Foo` is some struct. This relationship holds if `'x: 'a`.
- `fn(&'x i32): 'a` holds if `'x: 'a`.
- `for<'x> fn(&'x &'y i32): 'a` holds if `'y: 'a`. Note that `'x: 'a`
  is not required. This is because `'x` is bound within the type
  itself and is not a specific lifetime yet.
  
#### Fns, trait objects, and lifetimes

The reason that this change is not entirely backwards compatible is
that the older rules for when `<Type>: 'a` holds were somewhat more
subtle. The older rules reasoned about what kinds of data was actually
reachable, and ensured that all of this data outlived `'a`, but they
did not impose restrictions on parts of the type that don't represent
reachable data.

The difference between the old and new rules is easiest to see with a
fn type. Previously, `fn(&'x i32): 'static` would *always* hold, even
though `'x: 'static` is not true. The reason for this was that the fn
pointer itself doesn't *contain* any data at all, and certainly no
data with the lifetime `'x`. Rather, when called, it will be *given*
data with that lifetime.  The newer rules don't draw such fine
distinctions. Similar logic applies to type parameters on trait
objects.

The main place that these changes break normal code is in struct definitions
like the following:

```rust
// no longer legal
struct SomeStruct<'a,T> {
    &'a SomeTrait<T>
}

trait SomeTrait<T> { }
```

This struct now requires a `T:'a` declaration, because `SomeTrait<T>`
appears inside a borrowed reference of lifetime `'a` and thus
`SomeTrait<T>: 'a` must hold:

```rust
// the replacement:
struct SomeStruct<'a,T:'a> {
    &'a SomeTrait<T>
}

trait SomeTrait<T> { }
```

#### Projections

Associated type projections, like `X::Item`, are somewhat different
than other types, because they are defined through the trait system.
The older rules had two ways to decide whether `X::Item: 'a`:

- consult the where-clauses that are in scope (e.g., `where X::Item: 'a`).
- consult the trait definition (e.g., the trait might say `type
  Item: 'static` or something like that).

However, these two rules are not sufficient in most real-world cases.
For example, even a simple function like this one is actually illegal:

```rust
fn next<I:Iterator>(iter: &mut I) -> I::Item {
    iter.next().unwrap()
}
```

For this function to be legal, the compiler must know that `I::Item`
is valid during the function body, and in fact neither of the above
rules tell us that. After all, there are no where clauses, and in this
case the trait definition for `Iterator` doesn't say anything about
the lifetimes involved in the output. Of course, the code above
currently compiles, but that is due to a bug ([#24662]).

This RFC adds a new rule which says that we can conclude that
`I::Item: 'a` if `I: 'a` --- or, more generally, if all the inputs to
the projections outlive `'a` (e.g., a projection like `<A as
Trait<'b,C>>::Item` outlives `'a` if `A: 'a`, `'b: 'a`, and `C: 'a`).

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

### Implementation complications

One complication for the implementation is that there are so many
potential outlives rules for projections. In particular, the rule that
says `<P0 as Trait<P1..Pn>>>: 'a` holds if `Pi: 'a` is not an "if and
only if" rule. So, for example, if we know that `T: 'a` and `'b: 'a`,
then we know that `<T as Trait<'b>>:: Item: 'a` (for any trait and
item), but not vice versa. This complicates inference significantly,
since if variables are involved, we do not know whether to create
edges between the variables or not (put another way, the simple
dataflow model we are currently using doesn't truly suffice for these
rules).

This complication is unfortunate, but to a large extent already exists
with where-clauses and trait matching (see e.g. [#21974]). (Moreover,
it seems to be inherent to the concept of assocated types, since they
take several inputs (the parameters to the trait) which may or may not
be related to the actual type definition in question.)

For the time being, the current implementation takes a pragmatic
approach based on heuristics. It tries to avoid adding edges to the
region graph in various common scenarios, and in the end falls back to
enforcing conditions that may be stricter than necessary, but which
certainly suffice. We have not yet encountered an example in practice
where the current implementation rules do not suffice.

# Drawbacks

N/A

# Alternatives

I'm not aware of any appealing alternatives.

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
[#21974]: https://github.com/rust-lang/rust/issues/21974

# Appendix

The informal explanation glossed over some details. This appendix
tries to be a bit more thorough with how it is that we can conclude
that a projection outlives `'a` if its inputs outlive `'a`. To start,
let's specify the projection `<PROJ>` as:

    <P0 as Trait<P1...Pn>>::Id
    
where `P` can be a lifetime or type parameter as appropriate.
    
Then we know that there exists some impl of the form:

```rust
impl<X0..Xn> Trait<Q1..Qn> for Q0 {
    type Id = T;
}    
```

Here again, `X` can be a lifetime or type parameter name, and `Q` can
be any lifetime or type parameter.

Let `Θ` be a suitable substitution `[Xi => Ri]` such that `∀i. Θ Qi ==
Pi` (in other words, so that the impl applies to the projection). Then
the normalized form of `<PROJ>` is `Θ T`. Note that because trait
matching is invariant, the types must be exactly equal.

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

We aim to show that `∀i. Pi: 'a` implies `(Θ T): 'a`, which implies
that this rule is a sound approximation for normalization.  The
argument follows from two lemmas ("proofs" for these lemmas are
sketched below):

1. First, we show that if `Pi: 'a`, then every "subcomponent" of `Pi:
   'a`.  The idea here is that each variable `Xi` from the impl will
   match against and extract some subcomponent of `Pi`, and we wish to
   show that this subcomponent outlives `'a`.
2. Then we will show that the type `θ T` outlives `'a` if, for each of
   the in-scope parameters `Xi`, `Θ Xi: 'a`.

**Definition 1.** `Constrained(T)` defines the set of type/lifetime
parameters that are *constrained* by a type. This set is found just by
recursing over and extracting all subcomponents *except* for those
found in a projection. This is because a type like `X::Foo` does not
constrain what type `X` can take on, rather it uses `X` as an input to
compute a result:

    Constrained(scalar) = {}
    Constrained(X) = {X}
    Constrained(&'x T) = {'x} | Constrained(T)
    Constrained(R0..Rn+'x) = Union(Constrained(Ri)) | {'x}
    Constrained([T]) = Constrained(T),
    Constrained(for<..> fn(T1..Tn) -> T0) = Union(Constrained(Ti))
    Constrained(<P0 as Trait<P1..Pn>>::Id) = {} // empty set

**Definition 2.** `Constrained('a) = {'a}`. In other words, a lifetime
reference just constraints itself.

**Lemma 1:** Given `P: 'a`, `P = [X => R] Q`, and `X ∈ Constrained(Q)`,
then `R: 'a`. Proceed by induction and by cases over the form of `P`:

1. If `P` is a scalar or parameter, there are no subcomponents, so `R=P`.
2. For nominal types, references, objects, and function types, either
   `R=P` or `R` is some subcomponent of P. The appropriate "outlives"
   rules all require that all subcomponents outlive `'a`, and hence
   the conclusion follows by induction.
3. If `P` is a projection, that implies that `R=P`.
   - Otherwise, `Q` must be a projection, and in that case, `Constrained(Q)` would be
     the empty set.

**Lemma 2:** Given that `FV(T) ∈ X`, `∀i. Ri: 'a`, then `[X => R] T:
'a`. In other words, if all the type/lifetime parameters that appear
in a type outlive `'a`, then the type outlives `'a`. Follows by
inspection of the outlives rules.
