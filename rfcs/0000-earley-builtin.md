---
feature: earley-builtin
start-date: 2017-08-03
author: taktoa
co-authors: cleverca22
related-issues: https://github.com/NixOS/nix/issues/1491
---

# Summary
[summary]: #summary

A `builtin` function, tentatively named `earley`, should be added to Nix
as a way to easily parse text based on a given context-free grammar.
This will suffice for the vast majority of parsers one would want to write
in Nix, and will bring parsing up to feature parity with the ease of pretty
printing in Nix.

# Motivation
[motivation]: #motivation

> Why are we doing this? What use cases does it support? What is the expected
> outcome?

# Detailed design
[design]: #detailed-design

> This is the bulk of the RFC. Explain the design in enough detail for somebody
> familiar with the ecosystem to understand, and implement.  This should get
> into specifics and corner-cases, and include examples of how the feature is
> used.

# Drawbacks
[drawbacks]: #drawbacks

- Evaluation times
  - The Earley parsing algorithm has a worst-case `O(n³)` execution time, where
    `n` is the size of the input string. Most of the use cases for this are in
    parsing relatively short strings, so fixed costs will likely dominate.
  - The cost of compiling grammars repeatedly (since there is no caching of Nix
    evaluation) may increase evaluation times, though I expect that the majority
    of the uses of this `builtin` will be in NixOS modules, which make up a tiny
    fraction of the evaluation time for a Hydra server.
- Implementation complexity
  - The complexity of implementing the Nix language will increase, although it
    seems feasible to write an Earley parser in Nix that could be swapped in
    when the `builtin` is not available. Nevertheless, the amount and complexity
    of the code in `nix` and `nixpkgs` will increase. Effectively, we are
    trading implementation complexity off against correctness / robustness.
- If people start using `types.bnf` on `extraConfig`-style options, rather than
  adding separate `extraConfigValidated` options, the NixOS user experience may
  suffer in cases where the BNF is inconsistent with the true syntax.

# Alternatives
[alternatives]: #alternatives

> What other designs have been considered? What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

> What parts of the design are still TBD or unknowns?

# Future work
[future]: #future-work

> What future work, if any, would be implied or impacted by this feature
> without being directly part of the work?
