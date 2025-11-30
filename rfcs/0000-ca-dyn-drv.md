---
feature: (fill me in with a unique ident, my_awesome_feature)
start-date: (fill me in with today's date, YYYY-MM-DD)
author: John Ericson
co-authors: (find a buddy later to help out with the RFC)
shepherd-team: (names, to be nominated and accepted by RFC steering committee)
shepherd-leader: (name to be appointed by RFC steering committee)
related-issues: (will contain links to implementation PRs)
---

# Summary
[summary]: #summary

Finishing, and then stabilizing, content-addressing and dynamic derivations.

# Motivation
[motivation]: #motivation

Content-Addressing and dynamic derivations have both sat around in a partial state of completion in Nix for quite some time.
Unlike other longstanding experimental features like Flakes, however, they are an important synchronization point for multiple and new implementations on an ongoing basis.
This is because we expect --- and want --- a flourishing of different approaches to implementing builds.
This is true both as a whole, and also in terms of underlying networking, scheduling, storage strategies, as there is some independence between these areas, meaning one ought to be able to combine different approaches together, yielding even more combinations.

Because of this, it is useful to get together a few interested parties to decide on what the future should look like.
These features give a chance to rethink the core interfaces of the "store layer" of Nix (What to derivations look like? What does the binary cache look like?) in a way that will shape the ecosystem for years to come.

At the same time though, we don't want the perfect to be the enemy of the good.
We need to "ship" things to get familiarity with them, so that our final decisions can be informed from experience and not just theory alone.
Otherwise, this stuff will just sit in limbo longer.

As such, this RFC aims to propose a *roadmap* more than final destination.
It aims to clearly lay out what already makes sense, and should be agreeable to all parties, versus what still lacks consensus, and should be the focus of further exploration.
By framing the window of debate, the hopes is that we will know about areas we wish to gain more practical experience for, focus on those areas, and quickly be able to approach a final design.

Any highly technical one-off RFC is going to collect dust.
What we need instead is a *living* document that will authoritatively standardize the interface for any implementation.
An official, implementation-agnostic standard would indeed be the best, but in the absence of that, we have the Nix Reference manual.
The [Store chapter](https://nix.dev/manual/nix/development/store/index.html) in particular has been greatly expanded with details about how content addressing works.
The goal of the roadmap lays out 

# Detailed design
[design]: #detailed-design


This is the core, normative part of the RFC.
Explain the design in enough detail for somebody familiar with the ecosystem to understand, and implement.
This should get into specifics and corner-cases.
Yet, this section should also be terse, avoiding redundancy even at the cost of clarity.

# Examples and Interactions
[examples-and-interactions]: #examples-and-interactions

This section illustrates the detailed design.
This section should clarify all confusion the reader has from the previous sections.
It is especially important to counterbalance the desired terseness of the detailed design;
if you feel your detailed design is rudely short, consider making this section longer instead.

# Drawbacks
[drawbacks]: #drawbacks

What are the disadvantages of doing this?

# Alternatives
[alternatives]: #alternatives

What other designs have been considered? What is the impact of not doing this?
For each design decision made, discuss possible alternatives and compare them to the chosen solution.
The reader should be convinced that this is indeed the best possible solution for the problem at hand.

# Prior art
[prior-art]: #prior-art

You are unlikely to be the first one to tackle this problem.
Try to dig up earlier discussions around the topic or prior attempts at improving things.
Summarize, discuss what was good or bad, and compare to the current proposal.
If applicable, have a look at what other projects and communities are doing.
You may also discuss related work here, although some of that might be better located in other sections.

# Unresolved questions
[unresolved]: #unresolved-questions

What parts of the design are still TBD or unknowns?

# Future work
[future]: #future-work

What future work, if any, would be implied or impacted by this feature without being directly part of the work?
