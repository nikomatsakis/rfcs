- Feature Name: N/A
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

- Fix various oversights in the wellformedness rules.
- Remove contravariance from the language so that implied bounds are sound.
  - Describe how we could add it back later.
  
This is a breaking change motivated by soundness concerns. The impact
on existing code has been evaluated and found to be quite small.
  
# Motivation

### Implied lifetime bounds

Rust has long relied on *implied lifetime bounds* in order to lower
the annotation burden on programmers. For example, consider the
following function:

```rust
fn get_item<'a,'b,T>(x: &'a &'b [T]) -> &'a T {
    &x[0]
}
```

The compiler requires that references cannot outlive their referent
(the thing they are referring to). In `get_item`, this means that if
we return a `&'a T`, we must know that `T: 'a` for that to be legal.
Moreover, in the data we are returning is actually found inside a
vector of type `&'b [T]`, so we also have to know that `'b` outlives
`'a`, since we are going to be approximating the lifetime of the data
from `'b` to `'a`.

To avoid requiring explicit where clauses in cases like these, the
compiler currently leans on a notion of *implied lifetime bounds*. The
basic idea is that we can deduce a lot of lifetime relationships just
by looking at the argument types. In `get_item`, for example, the
caller provided a reference of type `&'a &'b [T]`. Since we know that
all references outlive their referents, this tells us both of the
conditions that we needed in order to check this function is correct:

- `'b` outlives `'a`, because the outer reference has lifetime `'a`
  and it refers to an inner reference with lifetime `'b`.
- `T` outlives `'b`, because the inner reference has lifetime `'b` and
  it refers to data of type `T`, so that data must outlive the
  lifetime.
  
### Problems with implied bounds and contravariance

Implied lifetimes bounds are used throughout Rust, and without them,
the language would be much more verbose. Unfortunately, we recently
realized that, at least in their current incarnation they interact
with variance to create a [soundness hole][#25860]. The problem is
through variance callers can manipulate the type of a function and
alter the expected argument types to be supertypes of the declared
types; this in turn means that those declared types no longer imply
the same set of bounds.

Consider the following example function `victim`:

```rust
fn victim<'a, 'b, T>(_: &'a &'b (), v: &'b T) -> &'a T { v }
```

Here, based on the type of the first argument, the body of `victim`
assumes that `'b:'a`.  However, a malicious caller can use
contravariant subtyping to change the type of the first argument to
something more strict:

```
fn bad<'a, T>(x: &'a T) -> &'static T {
    let f: fn(&'static &'static (), &'a T) -> &'static T =
        foo;
    ...
```

You see that the type of the function pointer `f` doesn't match the
declared types on `victim`, but is nonetheless accepted by the
compiler. This is variance "working as intended", because there is no
argument you could pass to `f` that wouldn't be valid for `victim`.
In other words, based solely on the types of the arguments, `f` can be
called for a *narrower* range of types than `victim`, so everything
seems ok.

Unfortunately, this subtyping relationship is not taking implied
lifetime bounds into account. When you consider those, we find that
`f` can be caller in a wider range of scenarios, because it can be
called for *any* two lifetimes `'a` and `'b`, rather than just when
`'b:'a`.

### Simple solution: remove contravariance

For the time being, this RFC proposes a simple measure to fix this:
**remove contravariance on fn argument types**. This *sounds* like a
drastic step, but it actually has very little impact on the
language. This is because the only place that contavariance occurs
today is with `fn` pointers -- in particular, since closures are trait
objects, they are
[invariant with respect to their argument types][#23938].

Removing contravariance is in some ways a very appealing change.  It
results in a much simpler mental model for users, because parameters
now fall into two simple categories:

- *Invariant:* Invariant types/lifetimes cannot be changed at all. A
  type or lifetime parameter is invariant if it appears in mutable
  state (like a `Cell` or `&mut T`), a trait object, or as a fn
  argument.
- *Variant:* Lifetimes in variant types can be *approximated* with
  shorter lifetimes. Most types (and lifetimes) are variant in
  practice.
  
However, removing contravariance is also somewhat
unsatisfying. Clearly it closes the soundness hole, but it also feels
like a bigger change than is needed, since it closes off one of the
fundamental forms of variance. Therefore, the RFC also describes a
possible future direction, called for/where types, that would permit
both implied bounds *and* contravariance. Implementing this extension
involves some non-trivial unknowns, however, and might also prove to
be a larger breaking change than removing contravariance; therefore,
it is described here but not proposed.

### Well-formed types

The notion of implied bounds rests upon a more fundamental notion of a
*well-formed type*. A type is considered *well-formed* if it meets
some simple correctness criteria. For builtin types like `&'a T` or
`[T]`, these criteria are built into the language. For user-defined
types like a struct or an enum, the criteria are declared in the form
of where-clauses.

In general, all types that appear in the source code ought to be
checked for well-formedness. For example, a struct declaration can
only be considered valid if all of the field types are well-formed.
This is the reason that struct

For example, consider this type, which combines a reference to a
hashmap and a vector of additional key/value pairs:

```rust
struct DeltaMap<'a, K:'a+Hash, V:'a> {
  base_map: &'a HashMap<K,V>,
  additional_values: Vec<(K,V)>
}
```

Here, the WF criteria for `DeltaMap<'a,K,V>` are that `K:'a`, `V:'a`,
and `K:Hash` must all be satisfied.

You may have also noticed that, in type declarations, we do not use
any mechanism of implied bounds. In general, Rust's design tries to
make type definitions (which occur relatively infrequently) more
explicit, and fn declarations (and in particular code in fn bodies)
relatively compact. This avoids "long distance" errors.

### Shortcomings today

Unfortunately, enforcement of WF bounds is somewhat haphazard today.
For example, for a type like `[T]`, we do not currently require that
`T: Sized` holds (e.g., [#21748]), and we do not enforce
well-formedness for associated type definitions in impls. We need to
make enforcement more uniform and consistent, both for soundness and
to help users set expectations.

# Detailed Design

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

    WfSlice:
      T WF
      T: Sized
      --------------------------------------------------
      [T] WF

    WfProjection:
      ∀i. Pi WF
      --------------------------------------------------
      <P0 as Trait<P1..Pn>>::Id WF

#### Deferred WF checking

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
point in type-checking where the fn is called or the trait object is
used, as we are guaranteed that all higher-ranked lifetimes are fully
instantiated at that point.

We employ a similar deferral strategy with where-clauses: because
where-clauses may contain higher-ranked bounds, we do not check that
the types appearing in those where-clauses are well-formed at the
declaration, but rather later, during trait checking, when we know
that all higher-ranked lifetimes are instantiated.

**Note:** If the types in the fn signature, do not involve
higher-ranked lifetimes, we could choose to enforce well-formedness in
that case. But it's important we not rely on that, since some of the
types may not be checked. Another option is to add the implied bounds
into the environment when checking that the argument types are
well-formed. This would be consistent but more challenging to
implement.

#### Implied bounds

In some cases, we use the WF relation to create *implied bounds*.
This is intended to reduce the annotation overhead for end-users. The
implied bounds for a given type `T` are the region relationships that
would be required to make `T` well-formed. The result is a set of
outlives clauses of the form:

- `'a:'r`
- `T:'r`
- `<T as Trait<..>>::Id: 'r`

Note that when computing the implied bounds, if we find a projection
like `<T as Trait<..>>::Id: 'r`, we add an implied bound for the
projection as a whole, but not for its components (i.e., not `T: 'r`,
in this example). This is because there are many ways to prove that a
projection outlives `'r`, and recursing to its components is but one.

#### When should we check the WF relation and under what conditions?

Currently the compiler performs WF checking in a somewhat haphazard
way: in some cases (such as impls), it omits checking WF, but in
others (such as fn bodies), it checks WF when it should not have
to. Partly that is due to the fact that the compiler currently
connects the WF and outlives relationship into one thing, rather than
separating them as proposed here.

**Constants/statics.** The type of a constant or static can be checked
for WF in an empty environment.

**Struct/enum declarations.** In a struct/enum declaration, we should
check that all field types are WF given the where-clauses from the struct.

**Function items.** For function items, the environment consists of
all the where-clauses from the fn, as well as implied bounds derived
from the fn's argument types. These are then used to check the WF of
all fn argument types, the fn return type, and any types that appear
in the fns body.

**Trait impls.** In a trait impl, we assume that all types appearing
in the impl header are well-formed. This means that the initial
environment for an impl consists of the impl where-clauses and implied
bounds derived from its header. Example: Given an impl like
`impl<'a,T> SomeTrait for &'a T`, the environment would be `T: Sized`
(explicit where-clause) and `T: 'a` (implied bound derived from `&'a
T`). This environment is used as the starting point for checking the
associated types, constants, and fns.

**Inherent impls.** In an inherent impl, we simply check the methods
as if they were normal functions.

**Trait declarations.** Trait declarations (and defaults) are checked
in the same fashion as impls, except that there are no implied bounds
from the impl header.

**Type aliases.** Type aliases are currently not checked for WF, which
is harmless since they are transparent to type-checking, effectively.
If we chose, we could check for WF, but it would mean more
annotations, for example: `type Ref<'a,T> = &'a T` would require a `T:
'a` annotation.

Several points in the list above made use of *implied bounds* based on
assuming that various types were WF. We have to ensure that those
bounds are checked on the reciprocal side, as follows:

**Fns being called.** Before calling a fn, we check that its argument
and return types are WF. This check takes place after all
higher-ranked lifetimes have been instantiated. Checking the argument
types ensures that the implied bounds due to argument types are
correct. Checking the return type ensures that the resulting type of
the call is WF.

**Projections of types, fns, and consts.** When we project an item out
of a trait, we must also validate that the types in the trait
reference are well-formed. Examples of projections are calling a trait
function (or obtaining it through UFCS), normalizing a projected like,
or using an associated constant. Checking the types in the trait
reference implies that the implied bounds from the impl are correct at
the time when members of the impl are accessed.

# Drawbacks

N/A

# Alternatives

N/A

# Unresolved questions

None.

# Appendix: for/where, an alternative view

The natural generalization of our current `fn` types is adding the
ability to attach arbitrary where clauses to higher-ranked types. That
is, a type like `for<'a,'b> fn(&'a &'b T)` might be written out more
explicitly by adding the implied bounds as explicit where-clauses
attached to the `for`:

    for<'a,'b> where<'b:'a, T:'b> fn(&'a &'b T)
    
These where-clauses must be discharged before the fn can be called.
They can also be discharged through subtyping, if no higher-ranked
regions are involved: that is, there might be a typing rule that
allows a where clause to be dropped from the type so long as it is
moved to the environment. (Similarly, having fewer where clauses would
be a subtype of having more.)

You can view the current notion of implied bounds as being a more
limited form of this formalism where the `where` clauses are exactly
the implied bounds of the argument types. However, making the `where`
clauses explicit has some advantages, because it means that one can
vary the types of the arguments (via contravariance) while leaving the
where clauses intact.

For example, if you had a function:

    fn foo<'a,'b,T>(&'a &'b T) { ... }

Under this RFC, the type of this function is:

    for<'a,'b> fn(&'a &'b T)
    
Under the for/where scheme, the full type would be:

    for<'a,'b> where<'b:'a, T:'b> fn(&'a &'b T)
    
Now, if we upcast this type to only accept static data as argument,
the where clauses are unaffected:

    for<'a,'b> where<'b:'a, T:'b> fn(&'static &'static T)
    
Viewed this way, we can see why the current fn types (in which one
cannot write where clauses explicitly) are invariant: changing the
argument types is fine, but it also changes the where clauses, and the
new where clauses are not a superset of the old ones, so the subtyping
relation does not hold. That is, if we write out the implicit where clauses
that result implicitly, we can see why variance on fns causes problems:

    for<'a,'b> where<'b:'a, T:'b> fn(&'a &'b T, &'b T) -> &'a T 
    <:
    for<'a,'b> fn(&'static &'static T, &'b T) -> &'a T
    ? (today yes, under this RFC no)
    
Clearly, this subtype relationship should not hold, because the where
clauses in the subtype are not implied by the supertype.

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
[#25860]: https://github.com/rust-lang/rust/issues/25860
[#23938]: https://github.com/rust-lang/rust/pull/23938
[#21748]: https://github.com/rust-lang/rust/issues/21748
