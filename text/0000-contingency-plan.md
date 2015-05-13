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
necessary to close flaws uncovered later. The *detailed description*
describes the steps to take when considering such a change.

However, there are other kinds of changes that we may want to make
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

One thing that is considered out of scope for this RFC is the question
of when to release Rust 2.0 (or, more generally, a new major version
of Rust). This RFC is focused on exploring the kinds of changes we can
make within a single major version; the question of when to change
major versions is complex enough to be worth considering separately.

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
3. Provide some annotation that allows crates to "opt out" of the
   stricter rule. For example, when we were working on the coherence
   orphan rules, we included a `#[old_orphan_check]` annotation that
   could be used to get the older semantics back. Use of this
   annotation should be considered deprecated.
   - While the change is still breaking, this at least makes it easy
     for crates to update and get back to compiling status quickly.
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

### Opt-in changes

For changes that are not soundness changes, an opt-in strategy should
be used. This section describes

The possible strategies are summarized below; for each strategy, there
is also a corresponding subsection going into more detail.

1. **Just fix it.** The simplest strategy: just fix the bug and let
  code break. This is appropriate when breakage would be limited in
  scope or severity, or when the older behavior is an ICE, crash, or
  something else that people would not be relying on.
2. **Deprecate and supply an alternative.** The preferred
  strategy for larger changes: introduce a deprecation warning
  indicating that the API or language feature is unsound, and
  recommend an alternative API. These deprecated APIs or constructs
  can be fully forbidden at some later point when the community has
  had time to migrate.
3. **Deprecate and opt-in to newer semantics.** In some cases, it
  may not be possible for the unsafe and safe alternatives to
  co-exist. In that case, we can introduce an explicit opt-in to the
  newer behavior, and deprecate code that does not use this opt-in.
  
The end of this section defines an overall workflow evaluating these
alternatives.

### "Just fix it"

### "Deprecate and supply an alternative"

This is the preferred alternative for those cases where the impact of
a change is determined to be too large to "just fix it". This approach
is particularly relevant to "safe" APIs that are found to be, in fact,
unsafe. The idea is to introduce a new, correct API and deprecate the
older API. The older API will however remain available for use so that
people have time to transition to the newer API.

We propose using the same deprecation lint as for other situations,
but a possible alternative is to use a distinct lint, so that people
can permit normal deprecation but forbid deprecation due to memory
safety violations.

### "Deprecate and opt-in to newer semantics"

Unfortunately, sometimes it is not possible for the older and newer
semantics to co-exist. For example, fixing bugs in the inference
engine often entails adding a constraint into a complex system of
constraints, and it is not necessarily possible to isolate the effect
of that new constraint.  This implies that we cannot issue a lint for
errors that result from the new constriant but errors for the
remainder. **In general, in these cases, we'd prefer to "just fix it"
if possible, possibly preceded by a more-aggressive-than-normal
outreach program to minimize breakage.** This is because maintaining
more than one version of type-system rules is a burden.

However, there are some changes that are more additive in nature, or
where we'd prefer to permit a more phased rollout. To handle such
cases, we propose adding an "opt-in" mechanism that allows Rust code
to declare the version of rustc that it is being written against.
This same opt-in mechanism can be used for other
almost-but-not-quite-backwards-compatible changes, such as introducing
new keywords.

The specific proposal is an attribute `#![rust_version="X.Y.Z"]` that
can be attached to the crate. Every build of the Rust compiler will
have built into it the current version as well as the oldest version
that included an update to the type-safety rules. When the Rust
compiler encounters such an attribute, it will compare the crate's
declared version (`X.Y.Z`) against these builtin versions.

- If the crate's declared version is newer than the version of the
  compiler, issue a lint warning of some kind.
- If the crate's declared version is older than the most recent
  type-safety update, issue a deprecation warning.
- Otherwise, proceed normally. The declared version will however
  determine the set of keywords that are in scope.

This version of rules implies that it is considered deprecated
behavior to not opt-in to type-safety updates. However, usafe of
crates that failed to opt-in is unaffected. This may be undesirable. A
possible extension then is that the version number becomes part of the
metadata, and linking against an external crate whose declared version
is older than the most recent type-safety update is also considered
deprecated behavior. This would increase the pressure on library
authors to opt-in to newer versions, while permitting use of older
libraries.

Another alternative design is to list features by name instead of
using a version number. The alternatives section below describes the
reasoning that led us to adopt a version-number based design.

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

# Alternatives

In terms of the overall scheme, there are two primary alternatives:

1. *Just fix everything.* This ensures Rust is maximally safe, but at
the cost of compatibility. It doesn't give people a good migration
path and seems likely to divide the ecosystem so that people continue
to use older builds of Rust, which is bad for everyone.
2. *Issue a new major release for such change.* This is basically the
same as the previous option, but in a more semver-compliant fashion.
In particular, it carries the same downsides.

With regard to specifics, the following alternatives or possible
extensions were described in the text above:

- Deprecation due to memory safety might be considered a distinct lint
  of its own.
- Use of crates whose declared version does not include type-system
  updates could be considered deprecated behavior.
  
Instead of using a version number to indicate opt-in to new keywords
or language semantics, one might instead prefer a more keyword-based
approach. For example, one could write `#![feature(foo)]` to opt in to
the feature "foo" and its associated keywords and type rules, rather
than `#![rust_version="1.2.3"]`. While using minimum version numbers
is more opaque than named features, they do offer several advantages:

1. Using named features, the list of features that must be attached to
   Rust code will grow indefinitely, presuming your crate wants to
   stay up to date.
2. Named features presents a combinatoric testing problem, where we
   should (in principle) test for all possible combinations of
   features. In cases where new rust versions also involve tweaks to
   the type system rules, this seems particularly risky.

# Unresolved questions

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
