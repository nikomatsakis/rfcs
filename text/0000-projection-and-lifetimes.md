- Feature Name: N/A
- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Type system changes to address the outlives relation with respect to
projections, and to better enforce that all types are well-formed
(meaning that they respect their declared bounds). The current
implementation can be both unsound ([#24662]), inconvenient
([#23442]), and surprising ([#21748], [#25692]). The changes are as follows:

- Simplify the outlives relation to be syntactically based.
- Specify improved rules for the outlives relation and projections.
- Specify more specifically where WF bounds are enforced, covering
  several cases missing from the implementation.

The proposes changes here have been tested and found to cause only a
modest number of regresions (approximately 19 root regressions were
found on crates.io; I'm doing some refactoring on the implementation
and will post an updated set of numbers soon). In order to minimize
the impact on users, the plan is to first introduce the changes in two
stages:

1. Initially, warnings will be issued for cases that violate the rules
   specified in this RFC. These warnings are not lints and cannot be
   silenced except by correcting the code such that it type-checks
   under the new rules.
2. After one release cycle, those warnings will become errors.

Note that although the changes do cause regressions, they also cause
some code (like that in [#23442]) which currently gets errors to
compile successfully.

# Motivation

### Projections and the outlives relation

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

### Well-formed types

A type is considered *well-formed* (WF) if it meets some simple
correctness criteria. For builtin types like `&'a T` or `[T]`, these
criteria are built into the language. For user-defined types like a
struct or an enum, the criteria are declared in the form of where
clauses. In general, all types that appear in the source and elsewhere
should be well-formed.

For example, consider this type, which combines a reference to a
hashmap and a vector of additional key/value pairs:

```rust
struct DeltaMap<'a,K:Hash+'a, V:'a> {
  base_map: &'a mut HashMap<K,V>,
  additional_values: Vec<(K,V)>
}
```

Here, the WF criteria for `DeltaMap<K,V>` are as follows:

- `K:Hash`, because of the bound on `K`
- `K:Sized`, because of the implicit `Sized` bound
- `V:Sized`, because of the implicit `Sized` bound
- `K:'a`, because of the explicit bound
- `V:'a`, because of the explicit bound

Let's look at those `K:'a` bounds a bit more closely. If you leave
them out, you will find that the the structure definition above does
not type-check. This is due to the requirement that the types of all
fields in a structure definition must be well-formed.  In this case,
the field `base_map` has the type `&'a mut HashMap<K,V>`, and a
reference type `&'a mut X` is only well-formed if `X: 'a`. Since here
`X=HashMap<K,V>`, this means that `HashMap<K,V>: 'a` must hold.  The
rules for the outlives relation are given in detail below, but the
simple version is that all type/lifetime parameters in the type must
outlive `'a`, so this means that `K: 'a` and `V: 'a` must hold.  At
this point, the analysis bottoms out -- `K` and `V` are type
parameters, not concrete types, so we are required to add the bound on
the struct to signal to users of the struct that, whatever `K` is, it
must outlive `'a` in order for the struct to be considered
well-formed.

#### An aside: explicit WF requirements on types

You might wonder why you have to write `K:Hash` and `K:'a` explicitly.
After all, they are obvious from the types of the fields. The reason
is that we want to make it possible to check whether a type like
`DeltaMap<'foo,T,U>` is well-formed *without* having to inspect the
types of the fields -- that is, in the current design, the only
information that we need to use to decide if `DeltaMap<'foo,T,U>` is
well-formed is the set of bounds and where-clauses.

This has real consequences on usability. It would be possible for the
compiler to infer bounds like `K:Hash` or `K:'a`, but the origin of
the bound might be quite remote. For example, we might have a series
of types like:

```rust
struct Wrap1<'a,K>(Wrap2<'a,K>);
struct Wrap2<'a,K>(Wrap3<'a,K>);
struct Wrap3<'a,K>(DeltaMap<'a,K,K>);
```

Now, for `Wrap1<'foo,T>` to be well-formed, `T:'foo` and `T:Hash` must
hold, but this is not obvious from the declaration of
`Wrap1`. Instead, you must trace deeply through its fields to find out
that this obligation exists.

#### Implied lifetime bounds

To help avoid undue annotation, Rust relies on implied lifetime bounds
in certain contexts. Currently, this is limited to fn bodies. The idea
is that for functions, we can make callers do some portion of the WF
validation, and let the callees just assume it has been done
already. (This is in contrast to the type definition, where we
required that the struct itself declares all of its requirements up
front in the form of where-clauses.)

To see this in action, consider a function that uses a `DeltaMap`:

```rust
fn foo<'a,K:Hash,V>(d: DeltaMap<'a,K,V>) { ... }
```

You'll notice that there are no `K:'a` or `V:'a` annotations required
here. This is due to *implied lifetime bounds*. Unlike structs, a
function's caller must examine not only the explicit bounds and
where-clauses, but *also* the argument and return types. When there
are generic type/lifetime parameters involved, the caller is in charge
of ensuring that those types are well-formed. (This is in contrast
with type definitions, where the type is in charge of figuring out its
own requirements and listing them in one place.)

As the name "implied lifetime bounds" suggests, we currently limit
implied bounds to region relationships. That is, we will implicitly
derive a bound like `K:'a` or `V:'a`, but not `K:Hash` -- this must
still be written manually. It might be a good idea to change this, but
that would be the topic of a separate RFC.

Currently, implied bound are limited to fn bodies. This RFC expands
the use of implied bounds to cover impl definitions as well, since
otherwise the annotation burden is quite painful. More on this in the
next section.

*NB.* There is an additional problem concerning the interaction of
implied bounds and contravariance ([#25860]). To better separate the
issues, this will be addressed in a follow-up RFC that should appear
shortly.

#### Missing WF checks

Unfortunately, the compiler currently fails to enforce WF in several
important cases. For example, the
[following program](http://is.gd/6JXjyg) is accepted:

```rust
struct MyType<T:Copy> { t: T }

trait ExampleTrait {
    type Output;
}

struct ExampleType;

impl ExampleTrait for ExampleType {
    type Output = MyType<Box<i32>>;
    //            ~~~~~~~~~~~~~~~~
    //                   |
    //   Note that `Box<i32>` is not `Copy`!
}
```

However, if we simply naively add the requirement that associated
types must be well-formed, this results in a large annotation burden
(see e.g. [PR 25701](https://github.com/rust-lang/rust/pull/25701/)).
For example, in practice, many iterator implementation break due to
region relationships:

```rust
impl<'a, T> IntoIterator for &'a LinkedList<T> { 
   type Item = &'a T;
   ...
}
```

The problem here is that for `&'a T` to be well-formed, `T: 'a` must
hold, but that is not specified in the where clauses. This RFC
proposes using implied bounds to address this concern -- specifically,
every impl is permitted to assume that all types which appear in the
trait reference are well-formed, and in turn each "user" of an impl
will validate this requirement whenever they project out of a trait
reference (e.g., to do a method call, or normalize an associated
type).

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
that the older rules for when `T: 'a` holds for some type
`T` were somewhat more subtle. The older rules reasoned about what
kinds of data was actually reachable, and ensured that all of this
data outlived `'a`, but they did not impose restrictions on parts of
the type that don't represent reachable data.

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

#### Fns, trait objects, and WF

Another change in this document is that we enforce WF requirements
more aggressively around fns and trait objects. Previously, we made no
effort to check that argument types are WF, because in some cases they
involve higher-ranked lifetimes and that check cannot be done.  So,
for example, if you have a function like `for<'a> fn(x: &'a T)`, then
the argument is well-formed only if `T: 'a`, but of course we don't
know what `'a` is yet. We could require `for<'a> T: 'a`, but that's
too strict -- `T: 'a` doesn't have to hold for ALL `'a`, just for the
particular lifetimes that occur in practice. So instead we defer
validation until the point of call, where `'a` is known.

However, there are plenty of simpler cases. For example, imagine that
`HashSet<K>` requires `K: Hash`, then it would be nice if
`fn(HashSet<NotHash>)` reported an error, since `NotHash: Hash`
doesn't hold. Currently, the error is deferred until you try to call
the function, but under this RFC, the type itself is not well-formed.

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
      | &r T                          // Reference (mut doesn't matter here)
      | O0..On+r                      // Object type
      | [T]                           // Slice type
      | for<r..> fn(T1..Tn) -> T0     // Function pointer
      | <P0 as Trait<P1..Pn>>::Id     // Projection
    P = r                             // Region name
      | T                             // Type
    O = for<r..> TraitId<P1..Pn>      // Object type fragment
    r = 'x                            // Region name
    
We'll use this to describe the rules in detail.
    
### Syntactic definition of the outlives relation

The outlives relation is defined in purely syntactic terms as follows.
These are inference rules written in a primitive ASCII notation. :) As
part of defining the outlives relation, we need to track the set of
lifetimes that are bound within the type we are looking at.  Let's
call that set `R=<r0..rn>`. We'll start out with a simple
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
      R ⊢ scalar: 'a

    OutlivesNominalType:
      ∀i. P[i]: 'a
      --------------------------------------------------
      R ⊢ Id<P0..Pn>: 'a

    OutlivesReference:
      R ⊢ 'x: 'a
      R ⊢ T: 'a
      --------------------------------------------------
      R ⊢ &'x T: 'a

    OutlivesObject:
      ∀i. R ⊢ O[i]: 'a
      R ⊢ 'x: 'a
      --------------------------------------------------
      R ⊢ O0..On+'x: 'a

    OutlivesFunction:
      ∀i. R,r.. ⊢ Ti: 'a
      --------------------------------------------------
      R ⊢ for<r..> fn(T1..Tn) -> T0

    OutlivesTraitRef:
      ∀i. R,r.. ⊢ Pi: 'a
      --------------------------------------------------
      R ⊢ for<r..> TraitId<P0..Pn>: 'a      
      
#### Outlives for lifetimes

The outlives relation for lifetimes depends on whether the lifetime in
question was bound within a type or not. In the usual case, we decide
the relationship between two lifetimes by consulting the environment.
Lifetimes representing scopes within the current fn have a
relationship derived from the code itself, lifetime parameters have
relationships defined by where-clauses and implied bounds:

      'x not in R
      ('x: 'a) in Env
      --------------------------------------------------
      R ⊢ 'x: 'a
      
For higher-ranked lifetimes, we simply ignore the relation, since the
lifetime is not yet known. This means for example that `fn<'a> fn(&'a
i32): 'x` holds, even though we do not yet know what region `'a` is
(and in fact it may be instantiated many times with different values
on each call to the fn).
      
      'x in R
      --------------------------------------------------
      R ⊢ 'x: 'a

#### Outlives for type parameters

For type parameters, the only way to draw "outlives" conclusions is to
find information in the environment (which is being threaded
implicitly here, since it is never modified). In terms of a Rust
program, this means both explicit where-clauses and implied bounds
derived from the signature (discussed below).

    OutlivesTypeParameterEnv:
      X: 'a in Env
      --------------------------------------------------
      R ⊢ X: 'a


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
      ∀i. R ⊢ Pi: 'a
      --------------------------------------------------
      R ⊢ <P0 as Trait<P1..Pn>>::Id: 'a

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

#### Implementation complications

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
      R ⊢ scalar WF

    WfParameter:
      --------------------------------------------------
      R ⊢ X WF

    WfNominalType:
      ∀i. R ⊢ Pi Wf             // parameters must be WF,
      C = WhereClauses(Id)      // and the conditions declared on Id must hold...
      R ⊢ [P0..Pn] C            // ...after substituting parameters, of course
      --------------------------------------------------
      R ⊢ Id<P0..Pn> WF

    WfReference:
      R ⊢ T WF                  // T must be WF
      R ⊢ T: 'x                 // T must outlive 'x
      --------------------------------------------------
      R ⊢ &'x T WF

    WfSlice:
      R ⊢ T WF
      R ⊢ T: Sized
      --------------------------------------------------
      [T] WF

    WfProjection:
      ∀i. R ⊢ Pi WF             // all components well-formed
      R ⊢ <P0 as Trait<P1..Pn>> // the projection itself is valid
      --------------------------------------------------
      R ⊢ <P0 as Trait<P1..Pn>>::Id WF

#### WF checking and higher-ranked types

There are two places in Rust where types can introduce lifetime names
into scope: fns and trait objects. These have somewhat different rules
than the rest, simply because they modify the set `R` of bound
lifetime names. Let's start with the rule for fn types:

    WfFn:
      ∀i. R, r.. ⊢ Ti
      --------------------------------------------------
      R ⊢ for<r..> fn(T1..Tn) -> T0

Basically, this rule says that a `fn` type is *always* WF, regardless
of what types it references.  This certainly accepts a type like
`for<'a> fn(x: &'a T)`.  However, it also accepts some types that it
probably shouldn't. Consider for example if we had a type like
`NoHash` that is not hashable; in that case, it'd be nice if
`fn(HashMap<NoHash, u32>)` were not considered well-formed. But these
rules would accept it, because `HashMap<NoHash,u32>` appears inside a
fn signature.

The object type rule is similar, though it includes an extra clause:

    WfObject:
      rᵢ = union of implied region bounds from Oi
      ∀i. rᵢ: r
      ∀i. R ⊢ Oi WF
      --------------------------------------------------
      R ⊢ O0..On+r WF

The first two clauses here state that the explicit lifetime bound `r`
must be an approximation for the the implicit bounds `rᵢ` derived from
the trait definitions. That is, if you have a trait definition like

```rust
trait Foo: 'static { ... }
```

and a trait object like `Foo+'x`, when we require that `'static: 'x`
(which is true, clearly, but in some cases the implicit bounds from
traits are not `'static` but rather some named lifetime).

The next clause states that all object fragments must be WF. An object
fragment is WF if its components are WF:

    WfObjectFragment:
      ∀i. R, r.. ⊢ Pi
      --------------------------------------------------
      R ⊢ for<r..> TraitId<P1..Pn>
      
Note that we don't check the where clauses declared on the trait
itself. These are checked when the object is created. The reason not
to check them here is because the `Self` type is not known (this is an
object, after all), and hence we can't check them in general. (But see
*unresolved questions*.)

#### WF checking a trait reference

In some contexts, we want to check a trait reference, such as the ones
that appear in where clauses or type parameter bounds. The rules for
this are given here:

    WfObjectFragment:
      ∀i. R, r.. ⊢ Pi
      C = WhereClauses(Id)      // and the conditions declared on Id must hold...
      R, r0...rn ⊢ [P0..Pn] C   // ...after substituting parameters, of course
      --------------------------------------------------
      R ⊢ for<r..> P0: TraitId<P1..Pn>

The rules are fairly straightforward. The components must be well 
      
#### Implied bounds

Implied bounds can be derived from the WF and outlives relations.  The
implied bounds from a type `T` are given by expanding the requirements
that `T: WF`. Since we currently limit ourselves to implied region
bounds, we we are interesting in extracting requirements of the form:

- `'a:'r`, where two regions must be related;
- `X:'r`, where a type parameter `X` outlives a region; or,
- `<T as Trait<..>>::Id: 'r`, where a projection outlives a region.

Some caution is required around projections when deriving implied
bounds. If we encounter a requirement that e.g. `X::Id: 'r`, we cannot
for example deduce that `X: 'r` must hold. This is because while `X:
'r` is *sufficient* for `X::Id: 'r` to hold, it is not *necessary* for
`X::Id: 'r` to hold. So we can only conclude that `X::Id: 'r` holds,
and not `X: 'r`.

#### When should we check the WF relation and under what conditions?

Currently the compiler performs WF checking in a somewhat haphazard
way: in some cases (such as impls), it omits checking WF, but in
others (such as fn bodies), it checks WF when it should not have
to. Partly that is due to the fact that the compiler currently
connects the WF and outlives relationship into one thing, rather than
separating them as described here.

**Constants/statics.** The type of a constant or static can be checked
for WF in an empty environment.

**Struct/enum declarations.** In a struct/enum declaration, we should
check that all field types are WF, given the bounds and where-clauses
from the struct declaration.

**Function items.** For function items, the environment consists of
all the where-clauses from the fn, as well as implied bounds derived
from the fn's argument types. These are then used to check the WF of
all fn argument types, the fn return type, and any types that appear
in the fns body. These WF requirements are imposed at each fn or
associated fn definition (as well as within trait items).

**Trait impls.** In a trait impl, we assume that all types appearing
in the impl header are well-formed. This means that the initial
environment for an impl consists of the impl where-clauses and implied
bounds derived from its header. Example: Given an impl like
`impl<'a,T> SomeTrait for &'a T`, the environment would be `T: Sized`
(explicit where-clause) and `T: 'a` (implied bound derived from `&'a
T`). This environment is used as the starting point for checking the items:

- Associated types must be WF in the trait environment.
- The types of associated constants must be WF in the trait environment.
- Associated fns are checked just like regular function items, but
  with the additional implied bounds from the impl signature.

**Inherent impls.** In an inherent impl, we can assume that the self
type is well-formed, but otherwise check the methods as if they were
normal functions.

**Trait declarations.** Trait declarations (and defaults) are checked
in the same fashion as impls, except that there are no implied bounds
from the impl header.

**Type aliases.** Type aliases are currently not checked for WF, since
they are considered transparent to type-checking. It's not clear that
this is the best policy, but it seems harmless, since the WF rules
will still be applied to the expanded version. See the *Unresolved
Questions* for some discussion on the alternatives here.

Several points in the list above made use of *implied bounds* based on
assuming that various types were WF. We have to ensure that those
bounds are checked on the reciprocal side, as follows:

**Fns being called.** Before calling a fn, we check that its argument
and return types are WF. This check takes place after all
higher-ranked lifetimes have been instantiated. Checking the argument
types ensures that the implied bounds due to argument types are
correct. Checking the return type ensures that the resulting type of
the call is WF.

**Method calls, "UFCS" notation for fns and constants.** These are the
two ways to project a value out of a trait reference. A method call or
UFCS resolution will require that the trait reference is WF according
to the rules given above.

**Normalizing associated type references.** Whenever a projection type
like `T::Foo` is normalized, we will require that the trait reference
is WF.

# Drawbacks

N/A

# Alternatives

I'm not aware of any appealing alternatives.

# Unresolved questions

For trait object fragments, should we check WF conditions when we can?
For example, if you have:

```rust
trait HashSet<K:Hash>
```

should an object like `Box<HashSet<NotHash>>` be illegal? It seems
like that would be inline with our "best effort" approach to bound
regions, so probably yes.

[RFC 192]: https://github.com/rust-lang/rfcs/blob/master/text/0192-bounds-on-object-and-generic-types.md
[RFC 195]: https://github.com/rust-lang/rfcs/blob/master/text/0195-associated-items.md
[RFC 447]: https://github.com/rust-lang/rfcs/blob/master/text/0447-no-unused-impl-parameters.md
[#21748]: https://github.com/rust-lang/rust/issues/21748
[#23442]: https://github.com/rust-lang/rust/issues/23442
[#24662]: https://github.com/rust-lang/rust/issues/24622
[#22436]: https://github.com/rust-lang/rust/pull/22436
[#22246]: https://github.com/rust-lang/rust/issues/22246
[#25860]: https://github.com/rust-lang/rust/issues/25860
[#25692]: https://github.com/rust-lang/rust/issues/25692
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
    Constrained(O0..On+'x) = Union(Constrained(Oi)) | {'x}
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
