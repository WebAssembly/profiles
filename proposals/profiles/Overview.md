# Language Profiles for Wasm

This page describes a proposal for adding _profiles_ to WebAssembly, which define language subsets with reduced functionality or restricted semantics.


## Motivation

Wasm aims to be a universal execution platform that is applicable to a wide variety of domains, environments, and ecosystems, such as the Web, cloud, edge, mobile, embedded, IoT, blockchain, desktop, even server-side computing.

Wasm is currently defined as one monolithic language with a fixed prescribed semantics.
This is meant to provide maximum portability, even across unrelated ecosystems that Wasm is employed in.

However, different ecosystems tend to have different requirements and constraints.
With Wasm growing new features and more complexity, not all of them are equally well served.
For example:

* Blockchain ecosystem require deterministic execution and hence cannot use threads or floats in their vanilla form.

* Resource-limited environments like embedded will have a hard time supporting expensive features like garbage collection.

* Embedded ecosystems may be tied to hardware that does not support SIMD, and might struggle with the extra implementation cost.

* For ecosystems that use Wasm to achieve high assurance, language simplicity is itself a value, since it affects e.g. the cost of verification.

Each of the major feature additions proposed for Wasm do not only increases the complexity and size of engines, they also make toolchains more complex, which may differ between ecosystems.

It hence is inevitable that some ecosystems will only support certain subsets of Wasm.
We are already seeing such a development with respect to determinism and SIMD.

In order to maximise portability in the face of this, the standard must not ignore *ecosystem diversity*.
Instead, it ought to _embrace_ it by defining a suitable set of language _profiles_ that the creators of an ecosystem can choose from.

(The choice of name is inspired by [Java Profiles](https://www.oracle.com/technical-resources/articles/java/architect-profiles.html).)

Where possible, an ecosystem should support the full language (e.g., the Web).
However, where it can't, standardised profiles provide two major advantages:

1. The extent and precise semantics of a language subset is well-defined.

2. Ecosystems making the same choices maintain 100% compatibility between them.

To that end, the goal is that profiles should be _coarse-grained_, organised around _very few_, coherent feature sets and desired properties, rather than many individual instructions.
For example, with respect to current Wasm as well as ongoing proposal work, the following features are conceivable targets for profile restrictions:

* Non-determinism
* Threads
* Vector types (SIMD)
* GC
* Stack switching

In each case, a related profile is expected to affect all instructions and constructs associated with a respective feature.


#### Goals

1. *Well-specified subsets.* Provide new Wasm ecosystems that cannot support the full language with suitable subsets, which would otherwise be forced to invent them on their own.

2. *Minimised fragmentation.* Maximise compatibility across diverse ecosystems by aligning them and keeping the number of profiles to a minimum. (Also, minimise complexity and test matrix for engines or tools that want to be configurable for multiple profiles.)

3. *Durability.* The set of available profiles should remain fairly stable over time, such that ecosystems rarely if ever need to reconsider their choice.


#### Non-Goals

1. *Versioning.* Profiles are _not_ intended to distinguish new vs old versions of the language – that is an independent dimension for which we already have have version numbers. Rather, we fully expect that new versions of the language will extend existing profiles. Profiles are meant to codify _permanent_ use case distinctions, not temporary implementation deviations.

2. *Feature detection.* Profiles are directed at ecosystem creators, not users of the language. While it may make sense to provide programmatic means for certain applications to become portable even across incompatible ecosystems, such means are outside the scope of this proposal.

3. *Alternate semantics.* Profiles can only _restrict_ the language, they are never intended to _change_ it, e.g., by introducing alternative features or alternative behaviours. Language changes would introduce significantly larger complications for ecosystems, implementations, and users. In particular, they would preclude the existence of a "full profile" that includes everything, and targetting which allows implementations to remain profile-agnostic.


#### Risks

1. *Profile inflation*. As stated above, it is a goal of profiles to _minimise_ fragmentation. For that purpose, the number of specified profiles should be as small as possible – a handful rather than a dozen. While it could be tempting to introduce micro-profiles for all sorts of special purposes, that temptation must be avoided. The goal is coarse not fine granularity.

2. *Short-cutting.* Blessing the notion of language subsets may reduce the perceived need for scrutiny or consensus-building when adding new features, because even questionable ones could be "whitewashed" by "barring" them into a profile. Such an approach would still increase fragmentation, adversely affect the complexity budget of the language, and could harm its overall integrity as well as reputation.

3. *Abuse.* It may also be tempting to apply the concept of profiles to scenarios it is not intended for, such as versioning. That would harm the purpose and create conflict with stated goals such as durability and minimised fragmentation.

4. *Over-optimistic assumptions.* Producers may assume that certain invariants that happen to be true for a profile they target hold in general. This may lead to incorrect behaviour of their code when run in more general environments. In general, producers should always assume the presence of the full language, even if they merely target a subset profile.


#### Intended Properties

* There exists a *full* profile, and other profiles may only present syntactic or semantic subsets of this profile, not alter behaviour. (A semantic subset is one that has fewer possible behaviours.)

* All profiles should be mutually compatible and composable. No two profiles should subset semantic behaviour in inconsistent ways (e.g., such that the intersection of their behaviours ends up empty in some places).

* Profiles should only be included in the standard for sufficiently broad use cases. The number of profiles should remain single digit.

* Unless a tool is specific to a platform with a restricted profile, producing code for the full profile should be the default for typical tools, and more specific choices should be explicitly requested.

* Runtime conditioning on the "current" profile should be avoided, both at the language level, and at the toolchain and platform level; users should pick a target profile at produce-time, and know at deploy-time which profile their code will run on.

* Proposals adding non-trivial functionality to the language should consider profile opt-outs for existing Wasm ecosystems that cannot implement it.


## Proposal

This proposal primarily is about adding suitable "specification infrastructure" to enable formally specifying profiles, rather than creating a multitude of profiles already.
We expect most profiles to be introduced as separate proposals, or as part of other feature proposals (see [below](#initial-profiles) for exceptions).

The proposal supports two main flavours of language subsetting:

* _Syntactic_, by _omitting_ a feature, in which case certain constructs are removed from the syntax altogether.

* _Semantic_, by _restricting_ a feature, in which case certain constructs are still present but some behaviours are ruled out.

In either case, the modification is specified by annotating respective rules in the specification – either grammar rules in the former case, or execution/validation rules in the latter – with a marker that defines them as "conditional" on a profile.

More concretely, defining a profile consists of three ingredients:

1. Introducing a new rule marker for the profile.

2. Annotating the rules _excluded_ in the profile with this marker.

3. Specificying the profile as a set of markers (a profile may be the intersection of multiple markers, enabling more modularity or the ability to define profiles derived from others).

Part (1) and (3) will be part of a new Appendix section. Part (2) is applied throughout the main body of the spec.

**Notes:**

(1) This approach is intentionally designed to only support _restricting_ the language. A language subset is not allowed to _modify_ the language in the sense of introducing dialects with new syntax or alternative behaviours!

(2) Profiles are intended to be chosen _globally_ by an ecosystem. They are not meant to be selectable by individual producers.

(3) Producers should view them solely as _restrictions_ to cope with, not as _guarantees_ to programmatically depend on (e.g., the absence of certain behaviours). This is so that (a) their code stays correct under any less restricted profile, and (b) their ecosystem remains free to extend its profile at a later point in time.


### Profile Markers

In the language specification, a _profile marker_ is an alphanumeric token identifying a feature set.
To avoid cluttering rules too much, and since we expect only few of these markers to exist, they are preferred to be one or two letters only.

*Example:* `N` could be used as the marker for the non-deterministic constructs or rules, `T` for threading, `V` for vector instructions (a.k.a. SIMD), etc.


### Grammar Annotations

To omit a construct from a profile, respective productions in the grammar of the _abstract syntax_ are annotated with an associated marker.

This is defined to have the following implications:

1. Any production in the _binary_ or _textual_ syntax that produces abstract syntax with a marked construct is omitted by extension.

2. Any _execution_ or _validation_ rule that handles a marked construct is omitted by extension.

The overall effect is that the respective construct is no longer part of the language under a respective profile.

*Example:* Take the threading proposal, which introduces shared memory primitives. These could be marked as "threading features" by annotating the abstract syntax accordingly:
```
    instr ::= ...
[T]        |  memory.atomic.notify memarg
[T]        |  memory.atomic.wait memarg
[T]        |  inn.atomic.load memarg
[T]        |  inn.atomic.store memarg
      ... (and so on)
```

A rule may be annotated by multiple markers, which e.g. might be the case if a construct is in the intersection of multiple features. The rule is then omitted if either of the features is.

*Example:* Imagine there was an instruction `v128.atomic.load` – it would have to be marked `[T V]`.


### Rule Annotations

To restrict certain behaviours in a profile, rules from the execution semantics are annotated with an associated marker.

This has the simple consequence that the rule is no longer applicable under a respective profile.

*Example:* Consider non-determinism. To define a "deterministic" profile, it is necessary to restrict the choice of NaNs for floating point instructions. This can be achieved by introducing a rule that normalises NaNs by picking a defined representative from the result set:
```
[N] (t.const c1) (t.const c2) t.binop  ~~>  (t.const c)  (iff c in binop_t(c1,c2))
    (t.const c1) (t.const c2) t.binop  ~~>  (t.const c)  (iff c = norm(binop_t(c1,c2)))
```
Here, we assume that the auxiliary meta function `norm` deterministally chooses a canonical candidate from the set of possible results.
In a deterministic profile, the first rule is excluded, hence normalisation is enforced.
Otherwise, both rules overlap, which means that normalisation does not need to happen (technically, the second rule becomes redundant in that case, since it's subsumed by the first).


### Profile Definitions

Finally, a profile is defined by a set of markers. The effect is that in the respective profile, all rules annotated with _any_ of the markers are excluded, i.e., the profile is the result of _intersecting_ the language subsets defined by each marker.

Profile definitions are meant to occur in the Appendix of the spec document.
For each definition, it ought to include a brief explanation and short informal summary of the omissions and restrictions.

*Example:* The "determinitic" profile could be defined as `{N,T}`, ruling out both threading and primitive non-deterministic behaviour. Similarly, a "scalar" profile would be `{V}`, omitting any vector instructions.

An implementation could declare itself as supporting only a "deterministic and scalar" profile, meaning that it only provides the intersection of the two profiles. Technically, this would be equivalent to a profile defined as `{N,T,V}`.

Finally, the "full" profile encompassing all features and behaviours would simply be defined by the empty marker set `{}`.


## Initial Profiles

While this proposal is mainly meant to define the necessary spec infrastructure, it is practically motivated by including a few relevant profiles already.
This is both to provide some precedence for use of the infrastructure, and because the respective features are already part of the spec with some pressure to support profiles handling them:

1. Full profile
2. Deterministic profile (excluding non-deterministic behaviour)
3. Scalar profile (excluding vector types and instructions)

Details to be fleshed out, but we expect them to be uncontroversial.


## Questions and Answers

* What is the relation to feature testing?

  - Feature testing typically is a relatively fine-grained and _intra-ecosystem_ problem, where each implementation may differ in support for new features at a given point in time, usually related to versioning. In contrast, profiles are coarse-grained and only ought to differ _inter-ecosystem_, i.e., in a given ecosystem it is fixed and prescribed globally for all implementations, so that testing for it is not useful. Both mechanisms would hence be mostly complementary.

* What are suitable profiles and criteria for introducing new profiles?

  - This question is mainly deferred to future proposals and the discretion of future CG discussions; evaluating the need for new profiles associated with a feature proposal could become part of the process document.

* Profile markers are the negation of what's in an actual profile, which may be slighlty confusing.

  - This seems difficult to avoid. Maybe rename to "feature markers"? But we want to prevent confusion with feature testing and versioning, which is a different feature.

* How should profiles be handled in the reference interpreter?

  - The interpreter should introduce flags for selecting profiles; the same might be true for similar tools like wabt.

* How should profiles be handled in the test suite?

  - It may be desirable to organise the test suite according to profile markers, i.e., have a sub directory corresponding to each profile marker (as is already the case with SIMD) and allow the runner to select which ones to execute.

* How are APIs affected, e.g., the JS API?

  - They might potentially annotate profile markers as well, e.g., certain API functions might be absent under a given profile.

* How are other tools affected?

  - In general, each toolchain has to define and document which (intersection of) profile(s) it supports. For most tools, it is expected that they continue to support the full language, in which case they do not need to care about profiles (because profiles can only restrict, not modify behaviour). Some tools (e.g., optimisers) may choose to make their target profile configurable.
