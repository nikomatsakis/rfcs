- Feature Name: N/A
- Start Date: 2015-05-07
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

This RFC has two main goals:

- define what precisely constitutes a breaking change for the Rust language itself;
- define a language versioning mechanism that extends the sorts of
  changes we can make without causing compilation failures (for
  example, adding new keywords).
  
# Motivation

With the release of 1.0, we need to establish clear policy on what
precisely constitutes a "minor" vs "major" change to the language.
**This RFC proposes limiting breaking changes to changes with
soundness implications**: this includes both bug fixes in the compiler
itself, as well as changes to the type system or RFCs that are
necessary to close flaws uncovered later.

However, simply landing all breaking changes immediately could be very
disruptive to the ecosystem. Therefore, **the RFC also proposes
specific measures to mitigate the impact of breaking changes**, and
some criteria when those measures might be appropriate.

Furthermore, there are other kinds of changes that we may want to make
which feel like they *ought* to be possible, but which are in fact
breaking changes. The simplest example is adding a new keyword to the
language -- despite being a purely additive change, a new keyword can
of course conflict with existing identifiers. Therefore, **the RFC
proposes a simple annotation that allows crates to designate the
version of the language they were written for**. This effectively
permits some amount of breaking changes by making them "opt-in"
through the version attribute.

However, even though the version attribute can be used to make
breaking changes "opt-in" (and hence not really breaking), this is
still a tool to be used with great caution. After all, just as we want
upgrading Rust to be hassle-free, we want it to be hassle-free to
upgrade a project to use the latest version of the
language. Therefore, **the RFC also proposes guidelines on when it is
appropriate to include an "opt-in" breaking change and when it is
not**.

This RFC is focused specifically on the question of what kinds of
changes we can make within a single major version (as well as some
limited mechanisms that lay the groundwork for certain kinds of
anticipated changes). It intentionally does not address the question
of a release schedule for Rust 2.0, nor does it propose any new
features itself. These topics are complex enough to be worth
considering in separate RFCs.

# Detailed design

The detailed design is broken into two major section: how to address
soundness changes, and how to address other, opt-in style changes. We
do not discuss non-breaking changes here, since obviously those are
safe.

### Soundness changes

When compiler bugs or soundness problems are encountered in the
language itself (as opposed to in a library), the preferred strategy
is simply to fix them, despite the fact that this may break code
(meaning: make code that was compiling and doing one reasonable thing
either stop compiling or start doing another thing). However, not all
breakage is considered equal, and it's important to do this fix the
right way so as to ease the transition.

The first step is to evaluate the impact of the fix on the crates
found in the `crates.io` website (using e.g. the crater tool). If
impact is found to be "small" (which this RFC does not attempt to
precisely define), then the fix can simply be landed. As today, the
commit message of any breaking change should include the term
`[breaking-change]` along with a description of how to resolve the
problem, which helps those people who are affected to migrate their
code. A description of the problem should also appear in the relevant
subteam report.

However, if we wish to minimize the impact, the following steps can
optionally be taken:

1. Identify important crates (such as those with many dependencies)
   and work with the crate author to correct the code as quickly as
   possible, ideally before the fix even lands.
2. Work hard to ensure that the error message identifies the problem
   clearly and suggests the appropriate solution.
3. Provide an annotation that allows for a scoped "opt out" of the
   newer rules, as described below. While the change is still
   breaking, this at least makes it easy for crates to update and get
   back to compiling status quickly.
4. Begin with a deprecation or other warning before issuing a hard
   error. In extreme cases, it might be nice to begin by issuing a
   deprecation warning for the unsound behavior, and only make the
   behavior a hard error after the deprecation has had time to
   circulate. This gives people more time to update their crates.
   However, this option may frequently not be available, because the
   source of a compilation error is often hard to pin down with
    precision.
   
Some of the factors that should be taken into consideration when
deciding whether and how to minimize the impact of a fix:

- How many crates on `crates.io` are affected?
  - This is a general proxy for the overall impact (since of course
    there will always be private crates that are not part of
    crates.io).
- Were particularly vital or widely used crates affected?
  - This could indicate that the impact will be wider than the raw
    number would suggest.
- Does the change silently change the result of running the program,
  or simply cause additional compilation failures?
  - The latter, while frustrating, are easier to diagnose.
- What changes are needed to get code compiling again? Are those
  changes obvious from the error message?
  - The more cryptic the error, the more frustrating it is when
    compilation fails.
    
#### Opting out

In some cases, it may be useful to permit users to opt out of new type
rules. The intention is that this "opt out" is used as a temporary
crutch to make it easy to get the code up and running. Depending on
the severity of the soundness fix, the "opt out" may be permanently
available, or it could be removed in a later release. In either case,
use of the "opt out" API would trigger the deprecation lint.

One interesting wrinkle is that, when possible, we would like to
ensure that it is possible for a library to opt-out of the newer rules
and remain compilable with an older compiler. In the past when we've
used temporary opt outs, we've done so by adding custom attributes
(e.g., `#[old_orphan_check]`). However, if we were to do that now,
it would cause the older compilers to emit an "unknown attribute" error.

XXX describe a specific attribute

### Opt-in changes

For breaking changes that are not related to soundness, an opt-in
strategy should be used. This section describes an attribute for
opting in to newer language updates, and gives guidelines on what
kinds of changes should or should not be introduced in this fashion.

#### Rust version attribute

The specific proposal is an attribute `#![rust_version="X.Y"]` that
can be attached to the crate; the version `X.Y` in this attribute is
called the crate's "declared version". Every build of the Rust
compiler will also have a version number built into it reflecting the
current release.

When a `#[rust_version="X.Y"]` attribute is encountered, the compiler
will endeavor to produce the semantics of Rust "as it was" during
version `X.Y`. RFCs that propose "opt-in breaking changes" should
discuss how the older behavior can be supported in the compiler, but
this is expected to be straightforward: if supporting older behavior
is hard to do, it may indicate that the opt-in breaking change is too
complex and should not be accepted.

If the crate declares a version `X.Y` that is *newer* than the
compiler itself, the compiler should simply issue a warning and
proceed as if the crate had declared the compiler's version (i.e., the
newer version the compielr knows about).

Note that if the changes introducing by the Rust version `X.Y` affect
parsing, implementing these semantics may require some limited amount
of feedback between the parser and the tokenizer, or else a limited
"pre-parse" to scan the set of crate attributes and extract the
version. For example, if version `X.Y` adds new keywords, the
tokenizer will likely need to be configured appropriately with the
proper set of keywords. For this reason, it may make sense to require
that the `#![rust_version]` attribute appear *first* on the crate.

#### When opt-in changes are appropriate

Opt-in changes allow us to greatly expand the scope of the kinds of
additions we can make without breaking existing code, but they are not
applicable in all situations. A good rule of thumb is that an opt-in
change is only appropriate if the exact effect of the older code can
be easily recreated in the newer system with only surface changes to
the syntax.

Another view is that opt-in changes are appropriate if those changes
do not affect the "abstract AST" of your Rust program. In other words,
existing Rust syntax is just a serialization of a more idealized view
of the syntax, in which there are no conflicts between keywords and
identifiers, syntactic sugar is expanded, and so forth. Opt-in changes
might affect the translation into this abstract AST, but should not
affect the semantics of the AST itself at a deeper level.

So, for example, the conflict between new keywords and existing
identifiers can (generally) be trivially worked around by renaming
identifiers, though the question of public identifiers is an
interesting one (contextual keywords may suffice, or else perhaps some
kind of escaping syntax -- we defer this question here for a later
RFC).

# Drawbacks

**Allowing unsafe code to continue compiling -- even with warnings --
raises the probability that people experiences crashes and other
undesirable effects while using Rust.** However, in practice, most
unsafety hazards are more theoretical than practical: consider the
problem with the `thread::scoped` API. To actually create a data-race,
one had to place the guard into an `Rc` cycle, which would be quite
unusual. Therefore, a compromise path that warns about bad content but
provides an option for gradual migration seems preferable.

**Deprecation implies that a maintenance burden.** For library APIs,
this is relatively simple, but for type-system changes it can be quite
onerous. We may want to consider a policy for dropping older,
deprecated type-system rules after some time, as discussed in the
section on *unresolved questions*.

## Notes on phasing

# Alternatives

**Rather than having a means to opt-in to minor breaking changes, one
might consider simply issuing a new major release for every such
change.** This seems like to have two potential negative effects. It
may simply cause us to not make some of the changes we would make
otherwise, or work harder to fit them within the existing syntactic
constraints. It may also serve to dilute the meaning of issuing a new
major version, since even additive changes that do not affect existing
code in any meaningful way would result in a major release. One would
then be tempted to have some *additional* numbering scheme, PR blitz,
or other means to notify people when a new major version is coming
that indicates deeper changes.

**Rather than simply fixing soundness bugs, we could use the opt-in
mechanism to fix them conditionally.** This was initially considered
as an option, but eventually rejected for the following reasons:

- This would effectively cause a deeper split between minor versions;
  currently, opt-in is limited to "surface changes" only, but allowing
  opt-in to affect the type system feels like it would be creating two
  distinct languages.
- It seems likely that all users of Rust will want to know that their code
  is sound and would not want to be working with unsafe constructs or bugs.
- Users may choose not to opt-in to newer versions because they do not
  need the new features introduced there or because they wish to
  preserve compatibility with older compilers. It would be sad for
  them to lose the benefits of bug fixes as well.
- We already have several mitigation measures, such as opt-out or
  temporary deprecation, that can be used to ease the transition
  around a soundness fix. Moreover, separating out new type rules so
  that they can be "opted into" can be very difficult and would
  complicate the compiler internally; it would also make it harder to
  reason about the type system as a whole.

**Rather than using a version number to opt-in to minor changes, one
might consider using the existing feature mechanism.** For example,
one could write `#![feature(foo)]` to opt in to the feature "foo" and
its associated keywords and type rules, rather than
`#![rust_version="1.2.3"]`. While using minimum version numbers is
more opaque than named features, they do offer several advantages:

1. Using named features, the list of features that must be attached to
   Rust code will grow indefinitely, presuming your crate wants to
   stay up to date.
2. Using a version attribute preserves a mental separation between
   "experimental work" (feature gates) and stable, new features.
3. Named features present a combinatoric testing problem, where we
   should (in principle) test for all possible combinations of
   features.
   
# Unresolved questions

**Should we add a mechanism for "escaping" keywords?"** We may need a
mechanism for escaping keywords in the future. Imagine you have a
public function named `foo`, and we add a keyword `foo`. Now, if you
opt in to the newer version of Rust, your function declarative is
illegal: but if you rename the function `foo`, you are making a
breaking change, which you may not wish to do. If we had an escaping
mechanism, you would probably still want to deprecate `foo` in favor
of a new function `bar` (since typing `foo` would be awkward), but it
could still exist.

**What precisely constitutes "small" impact?** This RFC does not
attempt to define when the impact of a patch is "small" or "not
small". We will have to develop guidelines over time based on
precedent. One of the big unknowns is how indicative the breakage we
observe on `crates.io` will be of the total breakage that will occur:
it is certainly possible that all crates on `crates.io` work fine, but
the change still breaks a large body of code we do not have access to.

**When and how should we remove deprecated behavior?** At some point
we will want to remove deprecate content; this is particular true of
type-system rules, since supporting those can greatly complicate the
compiler implementation (supporting older libraries is usually, but
not always, relatively painless).

One option for removing deprecated content is to declare a new semver
version. If there are deprecated libraries that still have significant
use, we could even move them out of `std` and into external crates
that are still available. This would be a backwards incompatible
change but would make it easier still for people to upgrade. Whether
this is appropriate probably depends on the severity and other
particulars of the problem.

Another option would be to reassess the impact of removing the
deprecated behavior, just as we would do for a new breaking change. If
we find that the impact is sufficiently low that the "just fix it"
option applies, we could simply remove the deprecated behavior at that
time without declaring a new major version.

**Should deprecation due to unsoundness have a special lint?** We may
not want to use the same deprecation lint for unsoundness that we use
for everything else.

**Should this RFC have been named 'How I learned to stop worrying and
love the lint'?** I sort of thought so, but I feared that nobody would
know what the RFC was about.
