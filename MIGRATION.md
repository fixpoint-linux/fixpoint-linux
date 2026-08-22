# fixpoint-linux — Cosmocc/C → Zig substrate migration

> **Status:** in-flight decision (2026-08). Companion to
> [`DESIGN.md`](./DESIGN.md) — this is a **substrate** migration, not a redesign.
> The Dhall-specified package set, the `datalog-dafsa` closure, the
> content-addressed store, and the timeline are all **substrate-agnostic** and
> remain exactly as designed. What changes is *how each artifact is produced*: the
> implementation language and toolchain, from **C11 + cosmocc (GCC/Clang → single
> APE)** toward **Zig**.
>
> The thesis — *a Linux system that is a fixed point, deterministically built from
> source by itself* — is preserved and, in one dimension, served better (see §1).

---

## 1. Why migrate the substrate (not the design)

`DESIGN.md` leans on cosmocc as the artifact substrate ("one portable APE per
package"). Migrating to Zig changes the substrate while keeping every architectural
promise:

1. **A cleaner self-hosting bootstrap.** cosmocc is a large prebuilt binary
   toolchain (GCC 14 + Clang 19 + LLVM + libcxx, assembled by `ahgamut/superconfigure`).
   Building *that* from source inside the fixed point is heavy. **Zig is
   self-hosting** and bootstraps from a tiny C shim (bootstrap.c → stage2 → stage3).
   A system whose goal is "rebuilt from source by itself" fits a self-hosting,
   light-bootstrap compiler more naturally than a huge C/C++ toolchain stack.
2. **Safety on the system-critical path.** PID1/init, the store daemon, and the
   rootfs tools are exactly where memory-safety matters most. Zig gives
   bounds/overflow checks by default, explicit allocators, and no hidden UB in the
   core language.
3. **One toolchain.** Zig bundles the build system, the compiler, the
   cross-compiler, and the linker — the org's "one toolchain, no framework sprawl"
   instinct, but with the builder in the target language (see §5 Stage 2).

**The trade:** Zig cannot cleanly produce Cosmopolitan APEs (`apelink` expects
GNU-ldbfd ELFs; the Zig↔cosmo path is hack territory). So this is a *forfeit* of
the single-polyglot-APE format for at least part of the org — a conscious decision
covered in §7.

---

## 2. Scope & inventory (surveyed 2026-08)

Own-code C footprint (excluding vendored git submodules and third-party libraries):

| Repo | Own C LOC | Files | Vendored C deps | Role |
|---|---|---|---|---|
| **dafsa** (carrasco-forcada-poc) | ~5.5k | 11 | — (leaf) | minimal incremental DAFSA engine |
| **dhall-c** | ~8.3k | 24 | — (leaf) | Dhall interpreter |
| **dhake** | ~2.9k | 1 | dhall-c | build runner |
| **compendium** | ~9.9k | 34 | dhall-c, dhake | authoritative DNS server |
| **shen** | ~12.9k | 34 | dafsa, qbe* | self-hosting sequent-calculus Lisp |
| **datalog-dafsa** | ~64.7k | 101 | dafsa, dhake, dhall-c, ggml* | the VM/fixpoint core |
| **fxstore** | ~4.8k | 7 | datalog-dafsa, dhall-c | content-addressed store |
| **visage** | ~15.8k | 28 | datalog-dafsa, dhake, dhall-c, mbedtls* | UI/visualization |
| **fx-init** | ~4.2k | 13 | dafsa, datalog-dafsa, dhake, dhall-c, fxstore | M4 init / PID1 |
| **total** | **~129k** | | | |

\* `qbe`, `ggml`, `mbedtls` are **third-party C — never migrated**; linked via FFI
or static.

Two facts that shape the whole plan:

1. **`datalog-dafsa` is ~50% of all own-code** (64.7k of ~129k LOC). It must be its
   own, heavily-gated stage — a "wholesale" one-pass is exactly how that repo
   silently breaks.
2. **The repos form a strict bottom-up dependency DAG** via git submodules (see §3).
   You cannot cleanly migrate a consumer before its dependencies.

---

## 3. The dependency DAG (why order matters)

```
dafsa (leaf)        dhall-c (leaf)
   │                     │
   │                     ├───────────┐
   │             dhake (2.9k)        │
   │                │                │
   │                ├────────┐       │
   │        datalog-dafsa (64.7k)  compendium (9.9k)
   │             │  │  │            │
   │   ┌─────────┘  │  └────────────┘
   │   │            │
 shen(12.9k)      fxstore (4.8k)
   │                │
   │              visage (15.8k)
   │                │
   └────────────── fx-init (4.2k) ← top of the DAG
```

Leaf dependencies migrate first; consumers migrate after. `fx-init` sits at the top
and is migrated **last** (which conveniently aligns with its being nearly complete
in C today — §7).

---

## 4. Two-track strategy

Strict bottom-up migration puts the **distro priority** (`fxstore` extension,
`fx-init`) at the top of the DAG, i.e. last. That would stall the M3/M4 work the
org cares about most. So the migration runs as **two interleaved tracks**:

- **Track A — net-new system code in Zig (start now).** Any *genuinely new*
  system component (fxstore extension beyond M5, future daemons, rootfs tools not
  yet written) is built directly in Zig, linking the still-C cores
  (`datalog-dafsa`, `dhall-c`) through **Zig's C-FFI**. Greenfield → no double
  work → the Linux-system layer ships in Zig immediately.
- **Track B — the library migration (parallel tail).** The bottom-up stages in §5.
  As each core lands in Zig, Track A's FFI boundary is swapped for native Zig calls.

Track A applies only to **net-new** code. Near-done C (fx-init, fxstore) is *not*
restarted — see §7.

---

## 5. Staged plan

### Stage 0 — Foundations & the agent loop
- **Pin one Zig version** and freeze it for the migration window (std lib churns
  0.11 → 0.15+; agents emit against whatever is pinned).
- **Build the differential-test harness:** compile the C module *and* its Zig
  translation, feed identical inputs, assert identical output. Hang it on existing
  harnesses (`datalog-dafsa`'s `eq_to_bound`, property tests, `fxinit_boot.sh`).
- **Prove the loop on one pilot** (dafsa) so later stages run a rehearsed workflow.

### Stage 1 — Leaves (bottom of the DAG)
- **dafsa** (~5.5k): pilot + first real migration. Differential vs C original +
  property tests against the published Carrasco–Forcada reference. Tier: `coder`.
- **dhall-c** (~8.3k): validate against the **official Dhall conformance suite** —
  the org's strongest correctness gate. Interpreter semantics are subtle: tier
  `pro-coder` + `reviewer`.

### Stage 2 — First consumers + the ethos win
- **dhake** (~2.9k, single file): migrating the *builder* early means the
  self-hosting build is driven by Zig — a large ethos win for tiny cost.
  Tier: `coder`.
- **compendium** (~9.9k): depends dhall-c + dhake. Tier: `coder` + `reviewer`.

### Stage 3 — The crown jewel: datalog-dafsa (~64.7k)
The VM / fixpoint / semi-naive machinery. Highest risk, ~half the codebase. Must be
staged *within* the repo: parser → intern/tupleset → compiler → VM → magic-sets.
Tier: `pro-coder` + `reviewer` at every module boundary, differential-tested.
**Do not rush this** — this is where "wholesale" is fatal.

### Stage 4 — Application/distro layer (top of the DAG)
- **shen** (~12.9k, self-hosting Lisp + moving generational GC — subtle; qbe stays
  C). Tier: `pro-coder` for the GC.
- **fxstore** (~4.8k), **visage** (~15.8k, mbedtls stays C), **fx-init** (~4.2k,
  the PID1). Tier: `coder`; `reviewer` for fx-init.

---

## 6. Verification strategy

The migration is safe **only because** the org has differential-testing assets —
this is what makes agent-assisted translation tractable:

| Asset | Used to validate |
|---|---|
| **Dhall conformance suite** | the Zig `dhall-c` against the canonical spec |
| **Carrasco–Forcada reference + property tests** | the Zig `dafsa` |
| **`eq_to_bound` + property tests** | the Zig `datalog-dafsa` at each module boundary |
| **`fxinit_boot.sh` (green end-to-end boot)** | the Zig `fx-init` — run the same boot scenario, assert identical behavior |

Golden rule: **no module merges until its Zig translation passes the same harness
as the C original.** `reviewer` is mandatory on every merge that touches the
correctness-critical core.

---

## 7. Decisions taken & open

**Taken:**
- **`fx-init` and `fxstore` finish in cosmocc/C; do not restart in Zig.** They are
  sunk, working code (fx-init on `m4-init` with a green boot harness). Finishing in
  C is strictly cheaper than restarting; both are tiny (4.2k / 4.8k) and migrate
  trivially in Stage 4. The green `fxinit_boot.sh` is retained as the future
  differential harness (§6).
- **Third-party C never migrates:** `qbe` (shen), `ggml` (datalog-dafsa), `mbedtls`
  (visage) — linked via FFI or static.
- **The Dhall/store/timeline architecture is untouched** by the substrate change.

**Open (to resolve before Stage 3):**
- **Dev-facing tools vs the APE format.** `compendium`, `visage`, `dhake` ship as
  single polyglot APEs to developers on any OS. Options: (a) keep cosmocc/APE for
  these (system layer + store + builder in Zig, dev tools stay APE), or (b)
  go all-Zig and accept **native static binaries per OS** (Zig cross-compiles; you
  lose the single-file polyglot). Default recommendation: **(a)** — the APE
  portability earns its keep on the dev-facing tools, and the system layer doesn't
  need it.
- **Pin the exact Zig version** (Stage 0).

---

## 8. Guardrails / risks

1. **`datalog-dafsa` is 50% of the code — never a one-pass.** Stage it internally,
   gate every module.
2. **Pin Zig.** Unpinned Zig means agents and CI emit against drifting `std`;
   everything breaks at once.
3. **Staged, not wholesale.** Agents lower the *cost per module*; they don't delete
   the modules. A gated, module-by-module push beats a one-shot rewrite.
4. **The subtle code goes to the top tier.** VM/fixpoint, the DAFSA minimizer, and
   shen's GC must route to `pro-coder` + `reviewer`, never the default tier on
   trust.
5. **Track A FFI lifetimes.** Borrowing C objects into Zig and swapping to native
   later needs a stable boundary contract (mirrors the earlier R2 zero-copy
   lifetime lesson in datalog-dafsa).

---

*This migration is a substrate change in service of the same fixed-point thesis. It
leaves the architecture of `DESIGN.md` intact, adds a self-hosting-bootstrap
alignment to the story, and keeps the org on "one toolchain" — just with the
toolchain, and eventually the builder, written in Zig.*
