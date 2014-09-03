- Start Date: 2014-09-02
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Remove the notion of bivariance from the language. Make it an error if
a type or lifetime parameter is unsed (what would today be inferred to
bivariance).

# Motivation

Today, variance inference for lifetimes includes the notion of
*bivariance* -- which essentially amounts to unconstrained. In
principle, this can have some use, but in practice it tends to be a
vector for bugs.  In fact, there is no known Rust code that
*intentionally* uses bivariance (though there seems to be plenty that
does so accidentally and incorrectly). This RFC proposes that we
simply make an inference result of bivariance an error.

As an example of where this comes up, imagine a `struct` with a "phantom"
lifetime parameter, meaning one that is not actually *used* in the fields
of the `struct` itself. One example of such a type is `Items`, the
vector iterator:

    struct Items<'vec, T> {
        x: *mut T
    }
    
Here the lifetime `'vec` is intended to represent the lifetime of the
vector being iterated over and hence to prevent the iterator from
outliving the container. However, because it does not appear in the
body of `Items` at all, the compiler would currently consider it
irrelevant to subtyping. This means that you could convert from a
`Items<'a, T>` to a `Items<'static, T>`, causing the iterator to
outlive the container it is iterating over.

To prevent this scenario, the *actual* definition of the iterator
in the standard library uses a *marker* type. The marker type informs
the compiler that, although `'vec` does not appear to be used, it should
act *as if* it were. For example, `Items` might be modified as follows:

    struct Items<'vec, T> {
        x: *mut T,
        marker: marker::CovariantType<&'vec T>,
    }

the `CovariantType` marker basically informs the compiler that it
should act "as though" a reference of type `&'vec T` were a member of
`Items`, even thought it is not. Another equivalent option here would
be `ContravariantLifetime`.

Currently, the user must know to insert these markers or else silently
get the wrong behavior. This RFC makes it an error to have a type or
lifetime parameter that is not (transitively) used somewhere in the
type. Nothing else is changed.

# Detailed design

Modify variance to report an error whenever a type or lifetime
parameter is determined to be bivariant. This change has been
implemented. You can view the results, and the impact on the standard
library, in [this branch on nikomatsakis's repository][b].

In the case of an unused type (resp. lifetime) parameter, the error
message explicitly suggests the use of a `CovariantType`
(resp. `ContravariantLifetime`) marker:

    type parameter `T` is never used; either remove it, or use a
    marker such as `std::kinds::marker::CovariantType`"
    
The goal is to help users as concretely as possible. The documentation
on the `CovariantType` marker type should also be helpful in guiding
users to make the right choice (the ability to easily attach
documentation to the marker type was in fact the major factor that led
us to adopt marker types in the first place).

# Alternatives

**Allow unused type parameters but not lifetime parameters.**
Currently all type parameters are invariant and hence it is not an
issue if they are unused. However, we plan to infer variance for type
parameters as well in the future (in fact, the code already infers a
suitable variance, it just doesn't make use of it), so forwards
compatibility requires that we reject those type parameters which
would be inferred to bivariant

**Default to a particular variance when a type or lifetime parameter
is unused.** A prior RFC advocated for this approach, mostly because
markers were seen as annoying to use, and because Rust so frequently
has a single right choice (`CovariantType`,
`ContravariantLifetime`). However, after some discussion, it seems
that it is more prudent to make a smaller change and retain explicit
declarations. We can always modify this behavior in the future in a
backwards compatible way.

Some factors that influenced this decision:

- Many unused lifetime parameters (and some unused type parameters) are in
  fact completely unnecessary. Defaulting to a particular variance would
  not help in identifying these cases (though a better dead code lint might).
- There are cases where phantom type parameters ought to be
  *invariant* and not *covariant* (e.g., `AtomicPtr`). Admittedly, these
  are few and far between. But defaulting to covariant would make these
  types silently wrong.
- Phantom type parameters occur rarely so it is not particularly painful
  to use explicit notation.

**Retain bivariance but remove the default.** We could make bivariance
"opt-in" via a marker and only report an error if the user has not
explicitly adopted any marker at all. I didn't do this because it
complicates the implementation and there are no known uses for
bivariance. It remains an option in the future.

**Remove variance inference and use fully explicit declarations.**
Variance inference is a rare case where we do non-local inference
across type declarations. It might seem more consistent to use
explicit declarations. However, variance declarations are notoriously
hard for people to understand. We were unable to come up with a
suitable set of keywords or other system that felt sufficiently
lightweight. Nonetheless this might be a good idea for a future RFC if
someone feels clever.

**Rename the marker types.** The current marker types have
particularly uninspiring names. In particular, it might be advisable
to remove everything but `CovariantType`, since the other markers can
all be modeled using variations on `CovariantType`. Renaming or
restructing the markers, however, seems out of scope for this RFC.

# Unresolved questions

None.

[b]: https://github.com/nikomatsakis/rust/tree/variance-defaults
