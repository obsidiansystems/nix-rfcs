---
feature: (fill me in with a unique ident, my_awesome_feature)
start-date: (fill me in with today's date, YYYY-MM-DD)
author: John Ericson
co-authors: (find a buddy later to help out with the RFC)
shepherd-team: (names, to be nominated and accepted by RFC steering committee)
shepherd-leader: (name to be appointed by RFC steering committee)
related-issues: https://github.com/NixOS/nix/issues?q=milestone%3A%22ca-derivations%20stabilisation%22%20state%3Aopen
---

# Summary
[summary]: #summary

Finishing, and then stabilizing, [content-addressed derivations] and [dynamic derivations].

# Motivation
[motivation]: #motivation

[Content-addressing][content-address] and [dynamic derivations][dyn-drv] have both sat around in a partial state of completion in Nix for quite some time.
Unlike other longstanding experimental features like Flakes, however, the store layer and thus these features are and will remain an important synchronization point for many implementations, on an ongoing basis.
This is because we expect --- and want --- a flourishing of different approaches to implementing builds.
And because "implementing builds" can be broken down into different approaches for underlying networking, scheduling, storage strategies, as there is some independence of the implementation strategy in each of these, one ought to be able to combine different approaches together, yielding even more combinations.
This only heightens the cross-cutting nature of these interfaces, and the need to get them right.

Because of this, it is useful to get together a few interested parties to decide on what the future should look like.
These features give a chance to rethink the core interfaces of the "store layer" of Nix (What do derivations look like? What does the binary cache look like?) in a way that will shape the ecosystem for years to come.

At the same time though, we don't want the perfect to be the enemy of the good.
We need to "ship" things to get familiarity with them, so that our final decisions can be informed from experience and not just theory alone.
Otherwise, this stuff will just sit in limbo longer.

As such, this RFC aims to propose a *roadmap* more than final destination.
It aims to clearly lay out what already makes sense, and should be agreeable to all parties, versus what still lacks consensus, and should be the focus of further exploration.
By framing the window of debate, the hope is that we will know about areas we wish to gain more practical experience for, focus on those areas, and quickly be able to approach a final design.

Any highly technical one-off RFC is going to collect dust.
What we need instead is a *living* document that will authoritatively standardize the interface for any implementation.
An official, implementation-agnostic standard would indeed be the best, but in the absence of that, we have the Nix Reference manual.
The [Store chapter] in particular has been greatly expanded with details about how content addressing works.

The goal of this RFC's roadmap is to lay out what we should do next in order to answer the remaining questions, ship working implementations, and, eventually, agree on a final specification for stabilization.
With an agreed upon process, we should be able to do all of the above successfully, and fairly quickly.

# Detailed design and roadmap
[design]: #detailed-design

## Already-made design decisions

These decisions are informed by the experience we've had in the past few years.

### Shallow traces only in the build trace

As described in the [manual][build-trace] (see also [#11896]), the main *build trace* should only contain resolved derivations for keys.
(In the [Build Systems à la Carte] paper terminology, these are called "shallow traces".)
This ensures we have a complete small step trace which is possible to audit, and makes avoiding various soundness issues much easier.

The use of shallow traces should better integrate with tools and projects like https://reproducibility.nixos.social/ where the aim is to track full bit-for-bit reproducibility.

### Build trace should use derivation paths

For most of CA derivation's history so far, the build trace has used derivation hashes and not regular store paths for referring to derivations in its keys (see [#11897]).
It is very unwieldy to introduce a second way of addressing derivations not used by the rest of Nix, and, as it turns out, it is also wholly unnecessary.

Along with the switch to shallow traces, the switch back to regular derivation paths will soon be implemented.

### RPC to avoid hash rewriting

For the past few years, Nix has readily rewritten outputs for content-addressing derivations.
This has worked surprisingly well in many cases, but is not sound in general, without Nix understanding every on-disk format it encounters, which is not feasible if hard-coded, and too much work for now if soft-coded.

For output-to-output references this can be avoided by imperatively submitting outputs to Nix, and getting back store paths on each submission.
Derivation builders can then use those received store paths to prepare the next output however they like.

We have a draft of what the RPC layer should look like in [PR #13768].
This currently uses [Varlink], which is an easy-to-use style of JSON RPC.
Ease of use is an important concern since arbitrary user-written derivations would be using this format.

Self references however are not addressed with this approach.
We can support them as an extension to the RPC protocol, but this would fundamentally work in the same way as before, with Nix rewriting self references (though it would still not need to go back to rewriting output-to-output references).
See below for more discussion.

### Same RPC for dynamic derivations

Dynamic derivations also need RPC for the builder to use.
And there is much agreement that using the full daemon protocol is overkill and inconvenient.
Since we need an RPC protocol for content-addressing derivations, per the above, it is a natural choice to also use the same one for this.
Only one additional operation is needed, which is inserting a derivation.

The old approach had the derivation submit a store object serialization for the derivation.
The new approach is for the user to just submit a derivation in JSON format.
Firstly, this is easier.
Secondly, this is more natural for decoupling the format derivations are canonically serialized as from the RPC format.
For example, the submitted JSON (like any RPC request) doesn't need to be in some exact normal form.
Nix, when it ingests it, will compute the store path, and return that to the caller.

## Not yet made decisions

These decisions still need to be informed by remaining implementation work.

### Canonical derivation format / addressing of derivations

For input-addressing, it is hard/impossible to change how derivations are hashed/addressed without changing output paths.
That would be a big breaking change, and would have to be carried out as a new, opt-in sort of derivation.
For content-addressing however, the derivation addresses are just used in the build trace, which is easier to migrate.

(For example: Rewriting the build trace, with a signature scheme delegating to the original entries, doubles the size of the build trace, but the build trace is tiny. The store objects themselves (and their content address paths) are not affected by this.)

As such, switching to content-addressing derivations is the perfect time to rethink the derivations format.

Decisions we might consider:

- Get rid of [ATerm].

  Eelco Visser had nice ambitions for ATerm to become a widely-used serialization format, but JSON has largely won the niche it was aiming for.
  A new derivation format should use something widely used, even if JSON is not appropriate for various reasons.

- Derivation options should be represented explicitly

  The current pattern of "stealing environment variables" (or structured attrs) is hard to use, and bad for compatibility forwards and backwards.

- Separation of concerns

  It would be nice to move some of the complexity out of the derivation format altogether.
  For example, perhaps structured attrs could be "desugared" away, so we are back to just caring about arbitrary files and environment variables.
  Or it would be nice for fixed-output derivations to just be floating content-addressing derivations with a separate assertion step.

Since (as described above) it is much easier to migrate the build trace, we don't need to figure out all these things at once.
We can instead see content-addressing as beginning a brief period of experimentation with the derivation format, rather than working really hard to fix everything with a single "v2" and delaying everything until then.

### Build trace signature format

Even after the switch to shallow traces and derivation path keys is implemented, the existing build trace format will still have some other questionable decisions.
For example, separate outputs are still signed separately, even though they are all built together.
Also the signature schema is rudimentary, and not forwards compatible with more flexible attestations/provenance (e.g. chain of trust "I am signing this because I trust this other public key which signed it").

We should incorporate the lessons of [laut] in making a much more robust signature scheme.

Note: This will probably result in Nix's "build result" data structure starting to look more like the build trace, and vice-versa.

### Whither self-references

This is the biggest uncertain question, in my mind.
As discussed above, there is no way to support [self-references][content-address] for content-addressed derivation outputs without some sort of Nix-side rewriting.
And currently, Nixpkgs is full of self-references.

There are a few ways this can play out:

- We get rid of all self-references in Nixpkgs.
  The new RPC format and content-addressing works for everything.

  This would be fantastic, but it would involve significant effort on the part of Nixpkgs maintainers patching software.
  It is something that, at best, would happen slowly over a long period of time.

- We get rid of all problematic self-references in Nixpkgs.
  Content addressing is used everywhere.
  Self-reference support via rewriting is used where possible (most cases).
  The remaining cases where it doesn't work, software is patched to not require self-references instead.

  This is more feasible, but it is still unknown how common the rewriting-defeating self references are.

- We have many rewriting-defeating self-references, more than we can patch, and we still use input-addressing in this case.

The last case puts the least amount of work on Nixpkgs.
But it does incur a large conceptual cost in Nix.
This is because, in order to properly support input-addressing in a world where we care about trust and attestation (i.e. many of the things we want to get out of content-addressing, in addition to faster rebuilds), we have to switch store models.
Instead of stores having a *set* of store objects, they need to have a *map*, where the store paths of store objects are not determined by the store objects themselves, but are separate freely-varying data.

I have [some design work][closure-integrity] on this front.
It is not necessarily hard to implement, but it will conceptually burden many tools in the Nix ecosystem.

(The easiest way to explain the set-map distinction is, asking "do I have this store path?" is not good enough, because that store path could be many such things.)

It would be nice to avoid this outcome, but to actually do so, we will need more hard evidence about the prevalence of rewriting-defeating self-references in Nixpkgs.

## Next steps

For content-addressing derivations, the next main goal is to gather the evidence needed to decide on the best way forward for the self-references question above.

But note also that the design uncertainty above *only* affects content-addressing *arbitrary* derivation outputs.
Dynamic derivations do not suffer from these issues, even though they build on content-addressing:

- derivations never have self-references (this has always been true), so the problem doesn't affect them

- while derivation producing derivations must be content-addressing (since derivations are always content-addressed), the dynamic derivations (outputs of those derivation-producing derivations) themselves can just be input addressed.

We want to continue implementing what we know we will need now outside of Nix, namely in Hydra.
These actually dovetail perfectly, as Hydra will also be useful for larger-scale experimental builds of Nixpkgs to gather the evidence we need.

### Hydra support

Hydra is currently undergoing a major overhaul with a new "queue runner" implementation in Rust.
This should soon (late December / early January) be put into production (hydra.nixos.org).
After they land, it is the perfect time to implement new features, now atop a much easier to maintain foundation.

Hydra has had some content-addressing support for a while, but with the build trace change described above that we've already committed to, this will need to be reworked.
We'll do that.

Hydra has never had support for dynamic derivations, but a chief aim of the new queue runner is for much more efficient handling of many concurrent build jobs.
This is fantastic timing, as the biggest uncertainty around dynamic derivations is the scalability of many more, smaller derivations.
Dynamic derivations should be implemented in Hydra too, after its content-addressing support is reworked.

### Evaluate self-references situation

Interested parties should support standing up hydra builders (perhaps we can use the staging hydra instance too) to try doing at-scale CA building of Nixpkgs.
We can evaluate the self-references situation as described above to figure out which solution we should pursue.

### Evaluate dynamic derivations situation

We should nixify some infamously large projects like Chromium to see how dynamic derivations scale for the sheer number of derivations.

At the same time, Nix itself should dogfood dynamic derivations (and Hydra) for its own PR CI, to study not the sheer number of derivations but the latency for practical purposes (local dev and CI).
(The Nix team should already be dogfooding hydra because GitHub Actions are slow and miss things.)

# Examples and Interactions
[examples-and-interactions]: #examples-and-interactions

The interactions are numerous since so many tools implement or use the Nix store layer interfaces in whole or part.
The roadmap tries to cover some of these tools and the work that needs to be done to bring up their support for the latest designs.

# Drawbacks
[drawbacks]: #drawbacks

There isn't much of a disadvantage, other than that doing the work requires time and effort.
The benefits of these architectural changes regarding security and incrementality are very strong.

# Alternatives
[alternatives]: #alternatives

The roadmap is fairly open ended already, so I find it a bit hard to think of yet more alternatives.
Of course, if anything comes up during the RFC process we can fill this section in accordingly.

# Prior art
[prior-art]: #prior-art

The prior art is our experience with the experiment so far.

# Unresolved questions
[unresolved]: #unresolved-questions

The unresolved questions are explicitly part of the roadmap, since this proposal is about the process towards an only-partially-determined outcome.

# Future work
[future]: #future-work

### Build trace performance

It is possible that for performance reasons we will need to add back in some deep build traces, but this should only be done as an optional caching layer (see [#11928]).
It is deferred for now.

### Garbage collection

Very similarly, there are many possible policies one might wish to have to clean up a shallow build trace.
Many of these benefit from things like a deep derivation caching layering, to figure out which small steps are relevant to the big steps one conceptually has as GC roots.
Since there is a wide policy space --- actually it is sound to delete any individual shallow build trace at any time, and since this effectively would depend on (customizable) versions of the caching logic above, this is also deferred for now.

<!-- Link references -->
[content-addressed derivations]: https://releases.nixos.org/nix/nix-2.33.0/manual/development/experimental-features.html#xp-feature-ca-derivations
[dynamic derivations]: https://releases.nixos.org/nix/nix-2.33.0/manual/development/experimental-features.html#xp-feature-dynamic-derivations
[content-address]: https://releases.nixos.org/nix/nix-2.33.0/manual/store/store-object/content-address.html
[dyn-drv]: https://releases.nixos.org/nix/nix-2.33.0/manual/store/derivation/index.html#extending-the-model-to-be-higher-order
[Store chapter]: https://nix.dev/manual/nix/development/store/index.html
[build-trace]: https://releases.nixos.org/nix/nix-2.33.0/manual/store/build-trace.html
[#11896]: https://github.com/NixOS/nix/issues/11896
[#11897]: https://github.com/NixOS/nix/issues/11897
[#11928]: https://github.com/NixOS/nix/issues/11928
[PR #13768]: https://github.com/NixOS/nix/pull/13768
[Varlink]: https://varlink.org/
[laut]: https://github.com/mschwaig/laut
[closure-integrity]: https://github.com/obsidiansystems/nix/tree/closure-integrity
[Build Systems à la Carte]: https://dl.acm.org/doi/10.1145/3236774
[ATerm]: https://releases.nixos.org/nix/nix-2.33.0/manual/protocols/derivation-aterm.html
