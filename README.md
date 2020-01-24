[![Build Status](https://travis-ci.org/WebAssembly/spec.svg?branch=master)](https://travis-ci.org/WebAssembly/spec)

# GC Proposal for WebAssembly

This repository is a clone of [github.com/WebAssembly/spec/](https://github.com/WebAssembly/spec/).
It is meant for discussion, prototype specification and implementation of a proposal to add garbage collection support to WebAssembly.

See the [overview](proposals/gc/Overview.md) for an informal summary of the proposal.
See the [MVP](proposals/gc/MVP.md) for a (more) formal description of the proposal.
See [NomOO](proposals/gc/NomOO.md) for an in-depth illustration of how a Java-like language would be supported by the proposal.
See [TypedFun](proposals/gc/TypedFun.md) for an in-depth illustration of how a Haskell/OCaml-like language would be supported by the proposal.

Original `README` from upstream repository follows...

# spec

This repository holds a prototypical reference implementation for WebAssembly,
which is currently serving as the official specification. Eventually, we expect
to produce a specification either written in human-readable prose or in a formal
specification language.

It also holds the WebAssembly testsuite, which tests numerous aspects of
conformance to the spec.

View the work-in-progress spec at [webassembly.github.io/spec](https://webassembly.github.io/spec/).

At this time, the contents of this repository are under development and known
to be "incomplet and inkorrect".

Participation is welcome. Discussions about new features, significant semantic
changes, or any specification change likely to generate substantial discussion
should take place in
[the WebAssembly design repository](https://github.com/WebAssembly/design)
first, so that this spec repository can remain focused. And please follow the
[guidelines for contributing](Contributing.md).
