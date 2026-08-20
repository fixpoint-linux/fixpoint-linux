# fixpoint-linux — a Dhall-specified, self-hosting Linux system

> **Status:** design / proposal (2026-08)
> **Thesis:** the whole system — its package derivations, the build graph, the
> content-addressed store, and the running configuration — is specified in
> **Dhall**, with **Datalog + DAFSA** as the computation and storage engine.
> It is *like Nix in its model* (pure derivations, a content-addressed store,
> hermetic builds) **without the Nix language**.

This document proposes the concrete architecture. It is a design writeup only —
no source is written yet.

---

## 1. Why this is the natural apex of the org

The org already owns every primitive a Nix-like system needs; this design is the
point where they all compose:

| Nix concept | Existing asset | Role here |
|---|---|---|
| Spec language + evaluator | `dhall-c` | parses/typechecks/normalizes the system spec |
| Build runner | `dhake` | executes typed derivations (action plans) |
| Store closure / GC fixed point | `datalog-dafsa` | computes the transitive dependency closure as a least fixed point |
| Store metadata (suffix-deduped) | `dafsa` | immutable, shared-suffix path/digest store |
| The artifacts themselves | `cosmocc` | one portable APE per package — fully self-contained |
| Hermetic build sandbox | bwrap (rattan) | isolates builds; purity nearly free for closed cosmocc builds |

Crucially, **Dhall has content-addressing built into the language**: every import
can carry a `sha256:` semantic integrity hash (`let x = ./foo.dhall sha256:…`).
Reproducibility is part of the type system, not grafted on after the fact the way
flake `?rev` locking had to be for Nix.

---

## 2. What "Nix's model" concretely is (the parts we copy)

1. **Derivations are pure.** A build output is a pure function of its inputs:
   `output = f(source, deps, builder)`. No hidden global state.
2. **A content-addressed store.** Outputs live at deterministic paths derived from
   hashing the full input set: `/fx/store/<hash>-<name>`. Same inputs → same path.
3. **Closure / GC.** The set of everything reachable from a root is computed as a
   fixed point over the dependency relation; the GC deletes unreachable paths.
4. **Hermetic, reproducible builds.** Sandboxed, deterministic, bit-identical
   across machines.

We copy all four. We do **not** copy the Nix language (untyped, lazy-override-heavy,
slow, unreadable) or the option-system machinery as-is.

---

## 3. The spec: `system.dhall`

The entire system is one Dhall expression. It normalizes to a **typed plan** that
`dhake` executes and `datalog-dafsa` closes over. Three layers:

### 3.1 Packages (derivations)

A derivation is a pure Dhall record — the unit Nix calls a "derivation" and this
design calls a `Package`:

```dhall
let Package =
      { -- identity & the pure input closure
        name : Text
      , version : Text
      , src : < Path : Text | Fetch : { url : Text, hash : Text } >
        -- src.hash is a dhall import-integrity sha256 when Fetch
      , deps : List Text                 -- package names this needs (build-time)
      , -- the build is a dhake target/action plan, not a shell string
        build :
          { target : Text                -- primary output file name
          , recipe : List Action
          }
      }
```

Notes:

- **The recipe is a `List Action`** (the exact union `dhake` already executes:
  `Shell | Copy | Mkdir | Rm | Touch | Move | Symlink | Chmod | Echo | Env | Run`).
  So *derivations are executable by the existing dhake binary* — no new executor.
- `deps` is what turns a set of packages into a *graph*; that graph is closed over
  by Datalog below.
- A `src` can be a `Path` (local tree) or a `Fetch` (URL + integrity hash). Fetch
  hashes reuse Dhall's `sha256:` integrity mechanism — a package whose source hash
  is pinned *cannot silently change*.

### 3.2 The module/override model (the hard design problem)

Dhall is **total and terminating** — there is no laziness, so nixpkgs-style lazy
overrides don't port directly. The design uses three Dhall-native mechanisms
instead:

1. **Overrides = function application.** A package is a *function*
   `Package -> Package` applied to a default. `let linux = default // { name = "x" }`
   uses record merge `//`; conditional fields use unions + `merge`. This is
   explicit, typed, and total — you can *never* leave an override dangling.
2. **The package set is a record of functions**, applied at the top level to
   produce the concrete `List Package` for a system. The `mapKey`/`mapValue` list
   form (already proven in dhake) carries the concrete set.
3. **Configuration is a plain typed record** (`system.dhall` = `{ packages : …,
   services : …, rootFs : … }`), type-checked by `dhall-c` before anything runs.

This is deliberately *smaller* than the NixOS module system. We trade the
infinite-laziness convenience of nixpkgs for **totality**: a configuration either
typechecks or it doesn't, and every derivation's inputs are fully known at
evaluation time. That is the entire point — *the "shitty" parts of Nix (untyped
overrides, runtime surprises) are what totality removes.*

### 3.3 The running system

A second Dhall expression, `config.dhall`, specifies the *runtime* surface the
built root filesystem is assembled into:

```dhall
{ kernel : Text                 -- path into the store
, init : Text
, hostname : Text
, users : List { name : Text, uid : Natural, groups : List Text }
, services : List { name : Text, command : Text, on : Text } -- e.g. on="tcp:53"
  -- each service is a store path -> one of the org's APE binaries (dnsd, visage, …)
}
```

This is evaluated by `dhall-c` at build time to *scaffold* `/etc`, `/bin` symlinks
into the store, init scripts, and service manifests. It is a template the build
fills from store paths — the store, not a hand-edited tree, is the source of truth.

---

## 4. The store: `fxstore` (new C work)

This is the genuinely new component. A content-addressed path store in the spirit
of `/nix/store`, but much smaller because every artifact is a single closed
cosmocc APE:

```
/fx/store/<sha256-of-input-closure>-<name>
```

### 4.1 Path computation (the fixed point)

The store path for a package is a function of its **entire input closure** —
`name`, `src` hash, `deps` store-paths, and the builder recipe. Because deps are
store-paths which themselves embed their own closures, the path is *recursive*:

```
path(p) = "<hash of (name, src.hash(p), [path(d) for d in deps(p)], build(p))>-<name>"
```

Computing `path` for every package *is a least-fixed-point computation over the
dependency graph*. **This is exactly what `datalog-dafsa` is for.**

### 4.2 The role of datalog-dafsa

Load `deps` as a relation and close over it:

```datalog
dep(x, y) :- package(x), deps(x, y).          -- declared edge
closure(x, y) :- dep(x, y).                   -- base
closure(x, y) :- dep(x, z), closure(z, y).    -- transitive (least fixed point)
```

- **Store-path assignment:** evaluate the fixed point once; every package gets its
  content-addressed path. Re-running with the same `system.dhall` yields the same
  closure and the same paths — reproducibility is *the result of the computation*.
- **GC:** roots = `config.dhall`'s referenced packages. Everything reachable via
  `closure` from the roots is live; everything else in `/fx/store` is garbage and
  deletable. `dl_query(goal_rel="closure")` enumerates the live set directly.
- **Build ordering:** a topological order of `closure` is the build plan — the
  `deps` relation is acyclic by construction (a package can't depend on itself
  transitively), so `dhake` executes leaves-first.

The store's own metadata (path → name, path → hash, path → mtime for
incrementality) is stored in a **DAFSA** — the shared-suffix minimization naturally
dedups the long common prefixes of store paths, and DAFSA lookup is exact and fast.

### 4.3 Incrementality

`datalog-dafsa` has transactions and CAS revisions (`dl_txn_begin/commit`,
`dl_cas_revision`). A package is "up to date" iff its store path exists and its
mtime ≥ max(mtime of inputs). `dhake` already does mtime-based up-to-date skipping;
the store just makes the *inputs* be store paths instead of loose files. Change a
source file → its package hash changes → downstream paths change → only the
affected slice rebuilds.

---

## 5. The build pipeline (end to end)

```
┌──────────────────────────────────────────────────────────────┐
│  system.dhall   config.dhall                                  │
│  (typed spec: packages + runtime config)                      │
└──────────────┬───────────────────────────────────────────────┘
               │  dhall-c: parse → infer_type → normalize
               ▼
        typed plan: List Package + config record
               │
               ▼
   datalog-dafsa: load deps relation, compute closure (fixed point)
               │  → assign content-addressed store paths
               │  → derive topological build order
               ▼
        /fx/store  (content-addressed paths)
               │
               ▼
        dhake: execute each package's List Action in a bwrap sandbox
               │  → produce store-path artifacts
               ▼
   assemble rootfs from config.dhall (symlinks into store, /etc, init, services)
               │
               ▼
        boot: cosmocc APE kernel/init → services are org APE binaries
```

Each stage is a **pure function** of the previous stage's typed output. There is no
hidden state anywhere in the pipeline.

---

## 6. Reproducibility (why it's cheap here)

Nix's hardest engineering is making arbitrary builds bit-reproducible. This
design mostly sidesteps it:

- **Every artifact is a single static cosmocc APE** — fully closed, no runtime
  deps, no shared-library resolution, no `$PATH` surprises at runtime.
- **All build inputs are store paths** — no network access in the sandbox, no
  hidden files, no ambient state.
- **The toolchain is fully controlled** (your own cosmocc + dhall-c + dhake +
  datalog-dafsa). Set `SOURCE_DATE_EPOCH=0`, pin `$PATH`, and builds are
  deterministic by construction rather than by heroic effort.
- **Content addressing is in the spec language** (`sha256:` import integrity), so
  a pinned source *cannot* drift.

The one genuinely-hard, boring part that remains: **sandbox isolation**. bwrap
already provides it, and the rattan sandbox is proof the pattern works. The design
keeps the blast radius small by building only a handful of self-hosting packages.

---

## 7. Scope — a personal distro, not a second nixpkgs

This is the most important guardrail. The failure mode of Nix-like systems is
trying to package *everything*. This design explicitly targets a **small,
self-hosting set**:

- `dhall-c`, `dhake`, `datalog-dafsa`, `dafsa`, `compendium`, `visage`, `shen-meta`
- the minimal kernel/init userspace they need
- nothing else

Each of these builds itself (dhake's `Dhakefile.dhall` is the proof), so the system
is a **fixed point**: it can rebuild itself from its own Dhall spec. That is the
org's thesis, and it stays in your head because the package set stays small.

---

## 8. Open design questions (to resolve before implementation)

1. **Store path hash granularity.** Hash the *entire input closure* in one digest
   (Nix's approach) vs. a DAFSA-structured incremental digest. Closure-hash is
   simpler and matches Datalog naturally; incremental is more work. **Proposal:**
   start with the full-closure hash (one `sha256` over the normalized derivation
   record + dep paths), as Datalog computes it in one fixed-point pass.
2. **Module/override ergonomics.** Confirm Dhall record `//` + function-application
   overrides cover the cases needed (they should, for a small fixed set), and
   whether a thin "options" layer is worth building at all in v1. **Proposal:** no
   module system in v1 — a flat typed record + per-package override functions.
3. **Sandbox transport.** bwrap is available locally; remote/CI hermeticity can be
   deferred. Decide whether the bwrap sandbox wraps *every* dhake action or only
   `Shell`/`Run`.
4. **`/fx/store` physical layout.** The DAFSA metadata store must be transactionally
   consistent with the artifacts on disk (a path must never exist without its
   metadata). The `dl_txn_*` API already gives atomicity — confirm commit ordering
   (metadata last) against crash.
5. **Kernel/init sourcing.** Whether the kernel itself is built from a Dhall
   derivation (full self-hosting) or fetched as a pinned, integrity-hashed
   artifact in v1. **Proposal:** fetch-with-hash in v1, Dhall-derive in v2.

---

## 9. Milestones (suggested)

- **M0 — store-path closure (the proof of the fixed point).** A tiny C tool
  (`fxpath`) that reads a Dhall package set, loads `deps` into datalog-dafsa,
  computes the closure, and prints content-addressed paths. *Proves the core idea
  end-to-end with almost no new surface.*
- **M1 — `fxstore`.** The content-addressed store on disk: write artifacts under
  `/fx/store/<hash>-<name>`, DAFSA metadata, transaction/CAS, `fxstore gc <root>`.
- **M2 — dhake integration.** Teach dhake to resolve `deps` to store paths and
  build into the store, in a bwrap sandbox, with mtime incrementality over store
  inputs.
- **M3 — the self-hosting system.** Dhall-spec all 7 org packages as derivations;
  the system rebuilds itself from `system.dhall`. **This is the org's thesis
  delivered.**
- **M4 — rootfs assembly + boot.** `config.dhall` → rootfs → a bootable image;
  services are the org's APE binaries.
- **M5 — the system timeline & rollback.** `fxstore timeline` / `fxstore rollback`
  on top of `datalog-dafsa`'s native snapshot time-travel (see §10).

---

## 10. System timeline & rollback (via `datalog-dafsa` time travel)

`datalog-dafsa` ships real time travel, so the timeline needs **no new engine
work** — `fxstore` just surfaces it. The relevant API, all present today:

- `dl_publish_snapshot()` — atomically saves the interner + all relations to a
  versioned snapshot dir and flips the `snapshots/CURRENT` pointer; reads then go
  through mmap.
- `dl_snapshot_versions()` — enumerate every published version in ascending order.
- `dl_query_version()` / `dl_query_bound_version()` — **as-of queries** that
  bypass the live path and never mutate `snap_version`: non-destructive time travel.
- `dl_set_snapshot_retain(n)` — prune all but the `n` most-recent versions.
- `dl_cas_revision()` + `dl_txn_*` — per-entity compare-and-swap + atomic batches
  for conflict-safe concurrent edits.

On-disk layout: `<store>/snapshots/<version>/manifest.txt` + per-relation mmap
views, with `<store>/snapshots/CURRENT`.

### 10.1 The timeline

The `/fx/store` is **one** `datalog-dafsa` DB, so every relation — `package`,
`deps`, `closure`, the installed set, `config` — is captured by a snapshot. **One
`dl_publish_snapshot()` per successful activation = one complete system state.**
`CURRENT` always points at the live system; the timeline is `dl_snapshot_versions()`.

### 10.2 Rollback semantics: **roll-forward** (decided)

To roll back to version *v*:

- **Roll-forward (default):** re-publish version *v*'s state as a *new* version.
  `CURRENT` stays monotonic and append-only — the timeline is a pure ledger, the
  rollback itself is recorded as history, and it is undoable (roll forward again).
- **True revert (escape hatch):** point `CURRENT` directly at version *v*
  (`fx rollback --hard <v>`). Breaks monotonicity; only for explicit recovery.

### 10.3 What falls out

- **Generation GC = `dl_set_snapshot_retain(N)`.** Keep the last N bootable
  generations, prune older — one call.
- **Boot rollback.** On boot, init reads the timeline, inspects a prior state via
  as-of query, and if the latest activation failed to come up, rolls forward to the
  last good version. Snapshot reads are mmap + bypass the live path, so inspecting
  "the system two weeks ago" is just an as-of query.
- **Conflict-safe concurrent edits = CAS.** Two agents both read `rev=5` of the
  config entity, both edit; only one `dl_cas_revision(db,"config",5,6)` wins, the
  loser gets `DL_E_CONFLICT` and must rebase — the "my plan is based on a stale
  system" detection a multi-agent-built system needs.
- **The closure is part of the snapshot.** Because `closure` is a relation, a
  snapshot captures *which* packages were live and *why*. GC deletes only paths
  unreachable from the **current `CURRENT` snapshot's** closure; historical
  versions keep their artifacts alive until pruned.

This strengthens the thesis: **a system that can inspect and restore any point in
its own history using only the primitives the storage engine already provides.**

---

*Everything in this design is built from existing, proven assets. The only
genuinely new C work is `fxstore` (and even it is a thin composition of
datalog-dafsa + dafsa + bwrap). The language, the evaluator, the build runner, the
closure computation, and the storage primitive are all already in the org. This is
a composition problem, not an invention problem — which is exactly why it fits.*
