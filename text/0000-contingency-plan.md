- Feature Name: N/A
- Start Date: 2015-05-07
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Detail a "contingency plan" for how we will handle soundness problems
uncovered after the 1.0 release.

# Motivation

With the release of 1.0, the goal is that Rust code which compiles
today will continue to compile into the future (at least until
2.0). However, Rust also has the goal of being a safe language. This
raises the obvious question: what happens if we realize that a
soundness hole that permits safe code to trigger undefined behavior
(e.g., a data race or use after free)? Of course we would like to
close the hole, but that implies that some code which used to compile
will no longer compile (indeed, this is the entire point of closing
the hole!). A similar problem can arise with prominent compiler bugs:
we may find that the compiler is simply doing the wrong thing, but
people have come to rely on that wrong behavior; in that case, fixing
the bug (which is clearly desirable) will also break code.

This RFC proposes various possible strategies for handling these kinds
of situations, and describes the factors that will be used to decide
which of the strategies is most appropriate in any given
situation. The goal in general is to balance two priorities:

- **Rust should be safe.** In particular, it should be possible to
  write Rust code and be assured that you are not accidentally relying
  on known memory safety holes or bugs.
- **Upgrades should be painless.** But, it should also be possible to
  upgrade Rust to a new version with a minimum of fuss.

# Detailed design

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

Frequently, I think, it will make sense to just fix the problem. If
the fix simply causes more code to compile or execute without
crashing, then there is no problem at all. Otherwise, we should
evaluate the impact of the fix on the crates found in the `crates.io`
website (using e.g. the crater tool). If impact is found to be "small"
(which this RFC does not attempt to precisely define), then the fix
can be landed. In some cases, we may also open PRs against open source
projects that workaround the breakage (perhaps in advance of landing
the change itself).

As today, the commit message of any breaking change should include the
term `[breaking-change]` along with a description of how to resolve
the problem, which helps those people who are affected to migrate
their code. A description of the problem should also appear in the
relevant subteam report.

*Optional:* Of course, `crates.io` does not represent the sum total of
Rust code.  There exist (or will exist) plenty of closed-source and
private crates elsewhere. Ideally, a branch containing the breaking
change would also be advertised, so that the developers of those
crates can test the impact and report back. However, this is not an
official requirement of the RFC, because we wish to minimize the "red
tape" required to land a patch. Furthermore, it is not clear how
effective such a thing would be, since many developers are necessarily
paying such close attention to the progress of the Rust project.

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
remainder.

To handle such cases, we propose adding an "opt-in" mechanism that
allows Rust code to declare the version of rustc that it is being
written against.  This same opt-in mechanism can be used for other
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
