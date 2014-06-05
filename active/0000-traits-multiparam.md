- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR #: (leave this empty)
- Rust Issue #: (leave this empty)

# Summary

Introduce a version of Haskell's *multiparameter type classes* into
Rust. Since Rust tends towards object-oriented terminology, I refer to
these as *multi-dispatch traits*. The key idea is that you have a
trait where multiple types are used to select the appropriate impl, in
contrast to the traits we have today, which always select the impl
based on exactly one type.

The proposal actually consists of two semi-independent parts:

1. Introduce *where clauses* as a more explicit, and more expressive,
   form of bounds.
2. Permit the `Self` type of a trait to be named. Also permit traits
   to require that the `Self` type be a tuple, so as to support
   multiple dispatch.

# Motivation

In the summary, I said that the goal of this RFC was to permit
*multi-dispatch*, where multiple types are used to select the impl
that will be used for a given trait. Actually, this description is
somewhat inaccurate, because the key idea of the RFC is model a
multi-dispatch situation using single-dispatch on a tuple of types.
Consider the `Add` trait as an example. In today's world, `Add` is
modeled as follows:

    trait Add<R,S> {
        fn add(self: &Self, right: &R) -> S;
    }
    
Here the type of the left-hand side corresponds to the *implicit* type
parameter `Self`. The type of the right-hand side is `R` and the type
of their sum is `S`. This is suboptimal because it means that only the
type of the left-hand side is used to select the impl; this in turn
implies that, without undue contortion, one cannot support something
like a complex number type that can be added to both integers or other
complex numbers.

## Self declarations in traits

The proposal consists of two parts. The first is simply to permit
traits to give an explicit name to their `Self` parameter. For
example, `Add` might opt to declare its self parameter as being called
`L`:

    trait Add<R,S> for L {
        fn add(self: &L, right: &R) -> S;
    }
    
This form is equivalent to the previous declaration, but with a more
natural name for `Self`. Next, we permit a limited form of pattern
matching, so that a trait can declare that its self-type is always a
set of tuples:

    trait Add<S> for (L, R) {
        fn add(self: &L, right: &R) -> S;
    }

This syntax means that the trait `Add` must always be implemented for
2-tuples, and the first component will be called `L` and the second
component will be called `R`. This is an important shift, because it now
becomes possible to declare implementations such as:

    impl Add<Complex> for (int, Complex) { ... }
    impl Add<Complex> for (Complex, int) { ... }
    impl Add<Complex> for (Complex, Complex) { ... }

In all cases, the coherence rules are satisfied, because at least one
component of the tuple is local to the `Complex` crate.

There are two options regarding traits that do not include a `for`
clause.  One option is to consider traits like `trait Foo { ... }` as
being equivalent to `trait Foo for Self { ... }`. Another is to say
that the `Self` parameter has no name unless it is given one, and
hence `trait Foo { ... }` cannot explicitly refer to the self type.

## Where clauses

The second part of this RFC is a proposal to add `where` clauses as a
secondary way of specifying type parameter bounds. Where clauses are
motivated by two concerns: first, that the current syntax is great for
simple cases but does not scale well to complex scenarios, and second,
that the current syntax simply cannot express the full range of things
one might like to express.

Today, when declared a type parameter, one may introduce trait bounds
following a `:`:

    impl<K:Hash+Eq,V> HashMap<K, V> {
        ..
    }

Sometimes the number of parameters can grow quite large, such as
this example extracted from some rather generic code in `rustc`:

    fn set_var_to_merged_bounds<
        T:Clone+InferStr+LatticeValue,
        V:Clone+Eq+ToStr+Vid+UnifyVid<Bounds<T>>>(
            &self,
            v_id: V,
            a: &Bounds<T>,
            b: &Bounds<T>,
            rank: uint)
            -> ures;

The proposal is to permit a `where` clause that follows the generic
parameterized item but precedes the `{`. Thus the first example could
be rewritten:

    impl<K,V> HashMap<K, V>
        where K : Hash + Eq
    {
        ..
    }

Naturally this applies to anything that can be parameterized: `impl`
declarations, `fn` declarations, and possibly `trait` and `struct`
definitions, though those do not current admit trait bounds.

The grammar for a `where` clause would be as follows (BNF):

    WHERE = 'where' BOUND { ',' BOUND } [,]
    BOUND = TYPE ':' TRAIT { '+' TRAIT } [+]
    TRAIT = Id [ '<' [ TYPE { ',' TYPE } [,] ] '>' ]
    TYPE  = ... (same type grammar as today)

The bounds which appear in the `WHERE` clause are unioned together.
Note that we accept the `+` notation in the `where` clause, which
gives the end user some license to bunch together small bounds or
space them out:

    fn set_var_to_merged_bounds<T,V>(&self,
                                     v_id: V,
                                     a: &Bounds<T>,
                                     b: &Bounds<T>,
                                     rank: uint)
                                     -> ures
        where T:Clone,
              T:InferStr,
              T:LatticeValue,
              V:Clone+Eq+ToStr+Vid+UnifyVid<Bounds<T>>,
    {                                     
        ..
    }
    
The current syntax would still be accepted and would effectively be
syntactic sugar for a `where` clause.

In these examples, using `where` arguably helped to make things more
readable, but was not strictly necessary. However, in a multi-dispatch
scenario, there are cases that simply cannot be expressed by the
current syntax. Let us continue with the `Complex` number example
from the previous section. Suppose we had this function `increment()`
that operates over `Complex` numbers:

    fn increment(c: Complex) -> Complex {
        1 + c
    }
    
Now perhaps we would like to make `increment` generic so that it can
work over any type that can be added to an integer. If we try to
do this with a normal bound, we encounter a problem:

    fn increment<T:...>(c: T) -> T {
        1 + c
    }
    
There is nothing we can write after `T:` that expresses the type we
want. We want to say that `(int, T)` is addable, but that notation
simply does not exist. This is exacerbated by the choice to use tuples
to express multi-dispatch, but is not specific to that choice: in
general, the bound syntax requires that the `Self` type be a type
parameter, but in this case we want it to be (or at least reference)
the type `int`.

Using a where clause, there is no problem:

    fn increment<T>(c: T) -> T
        where (int, T) : Add<T>
    {
        1 + c
    }    

# Detailed design


# Drawbacks



# Alternatives

## 

## Double dispatch

# Unresolved questions

What parts of the design are still TBD?

[comparison]: http://citeseerx.ist.psu.edu/viewdoc/summary?doi=10.1.1.110.122
[pnkfelix]: http://blog.pnkfx.org/blog/2013/04/22/designing-syntax-for-associated-items-in-rust/#background
[part1]: http://www.smallcultfollowing.com/babysteps/blog/2013/04/02/associated-items/
[part2]: http://www.smallcultfollowing.com/babysteps/blog/2013/04/03/associated-items-continued/
