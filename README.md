# fixpoint-linux

A Dhall-specified, self-hosting Linux system.

The system is *like Nix in its model* (pure derivations, a content-addressed
store, hermetic builds) **without the Nix language** — the whole spec is written
in [Dhall](https://dhall-lang.org/), with Datalog + DAFSA as the computation and
storage engine, and every artifact built as a single portable cosmocc APE.

See **[`DESIGN.md`](DESIGN.md)** for the full architecture.
