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

## API

The `builtins.earley` primitive operation should take two arguments: a Nix value
representing a BNF grammar and a string to be parsed. It should return a Nix
value representing a parsed abstract syntax tree.

### Definition and representation of regular expressions

For the purposes of this RFC, a _regular expression_ over an alphabet `Σ` is
one of the following:

1. `char(a)` for any `a ∈ Σ`.
2. `seq(r, s)`, where `r` and `s` are regular expressions.
3. `alt(r, s)`, where `r` and `s` are regular expressions.
4. `star(r)`, where `r` is a regular expression.

We will represent this in Nix in the following way:

- `char(a)` becomes `{ char = a; }`, assuming that `Σ` is a subset of the set
  of all Nix values.
- `seq(r, s)` becomes `{ seq = [r s]; }`.
  Note that the `seq` attribute maps to a list of `r` and `s`; this
  representation is "spineless", so `{ seq = [p q r s]; }` should be a
  valid representation of `seq(p, seq(q, seq(r, s)))`.
- `alt(r, s)` becomes `{ alt = [r s]; }`; this is similarly spineless.
- `star(r)` becomes `{ star = r; }`.

`Σ` cannot contain any attribute sets that include `char`, `seq`, `alt`, or
`star` as attributes; this makes our regular expression representation
unambiguous.

### Definition and representation of context-free grammars

Classically, a _context-free grammar_ is a 4-tuple `(V, Σ, P, S)`, where `V` is
a set of _nonterminals_, `Σ` is a set of _terminals_, `P ∈ V → R(V ∪ Σ)` is a
set of _productions_, `S ∈ V` is the _starting nonterminal_, and for any
set `X`, `R(X)` is the set of regular expressions with `X` as an alphabet.

More simply put, it is the combination of two things:

1. A set of equations between nonterminals and regular expressions over
   terminals and nonterminals, where every nonterminal has precisely one such
   equation (thus it can be thought of as a function of type `V → R(V ∪ Σ)`).
2. A distinguished start nonterminal, `S`.

There are many existing syntaxes for context-free grammars: [EBNF][], [ABNF][],
etc. However, it is better to have `builtins.earley` accept a structured Nix
value, as we can then implement any of these syntaxes by parsing the grammar
itself using `builtins.earley bnfGrammar`.

We will represent a BNF as an attribute set with two attributes: `start` and
`productions`. The value associated with `start` is a string that must be an
attribute of `productions`. The value associated with `productions` is an
attribute set containing nonterminals as attributes and representations of
regular expressions as values. The regular expressions are over an alphabet
comprising the set of all Nix strings along with the set of all attribute sets
of the form `{ ref = nt; }`, where `nt` is a Nix string corresponding to a
nonterminal. Here is a complete example:

```
{
  start = "foo";
  productions = {
    foo = { alt = [ { ref = "bar"; } { ref = "baz"; } ]; };
    bar = { seq = [ "aaa" { ref = "foo"; } ]; };
    baz = { alt = [ "bbb" "ccc" ]; };
  };
}
```

[EBNF]: https://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_form
[ABNF]: https://en.wikipedia.org/wiki/Augmented_Backus%E2%80%93Naur_form

### Parsed AST format

FIXME

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

- FIXME: Discuss PEG parser
- FIXME: Discuss writing primops allowing a parser combinator library

> What other designs have been considered? What is the impact of not doing this?

# Unresolved questions
[unresolved]: #unresolved-questions

> What parts of the design are still TBD or unknowns?

# Future work
[future]: #future-work

> What future work, if any, would be implied or impacted by this feature
> without being directly part of the work?
