# Making "done" machine-checkable — forcing-function-driven completeness

*How I keep a large, feature-rich, solo-built control surface from silently half-shipping — by encoding every "this is what done means" rule as a lint gate or a test that fails the moment the rule is broken. The discipline behind the [systems work in 05](./05-systems-and-instrumentation.md), applied to product quality instead of crashes.*

> **Status:** engineering methodology. Told at the transferable level — the *discipline*,
> not the proprietary application it runs on. The tooling is generic and open
> (`ruff`, `mypy`, `vitest`, `axe-core`, `pip-audit`, GitHub Actions); the domain-specific
> control surface it guards is intentionally omitted.

## Context

I build and maintain a large, data-driven **operator control plane** for a low-level technical
domain: a Python/**FastAPI** backend and a **React + Vite + TypeScript** single-page app, with a
declarative feature registry. Dozens of capabilities, each spanning five layers — a descriptor,
an API route, a typed client, a UI component, and tests + docs. It's built **solo, across many
sessions**, often months apart.

## Problem

In a feature-rich app grown incrementally, **"done" rots silently.** None of these break a
build or a test, so none of them are visible:

- A backend route ships with **no UI** that can reach it (a capability nobody can actually use).
- A UI calls a client method that **doesn't exist** yet, or a route nothing calls.
- A new code path performs a state-mutating write that **bypasses the one audited write path**.
- Docs assert "**N** of X" counts that **drifted** from the code months ago.
- A risky action renders with **no badge**, so the operator can't see it's dangerous.

Each is a small, invisible gap. Accumulated across hundreds of features and dozens of sessions,
they produce a product that *looks* complete and isn't. **Discipline alone does not scale** —
not across that surface area, and not across a future session where "I'll remember the contract"
is a lie. The contract has to be encoded somewhere that *fails loudly*.

## Constraints

- **Solo + multi-session.** The rules can't live in my head; the next session (or the next
  engineer) has to inherit them as something executable, not tribal knowledge.
- **CI budget exhausted.** Mid-cycle the account's CI minutes ran out. Development couldn't
  stop, so the quality gate had to run **locally for $0** and merge with `[skip ci]` — while
  remaining the **single source of truth** (CI invokes the *same* gate target, so the two can
  never disagree).
- **Safety-critical writes.** The tool can mutate live state. **Exactly one** armed,
  rate-limited, audited write path must be the only way anything is written — *forever*, even
  as new code is added by someone who's never heard of the rule.

## Approach — turn every doctrine into a forcing function

The core move: **a rule you can't check is a rule that will be broken.** So every "we always do
X" became a lint gate or a test that goes red the instant X stops being true. "Cover everything"
stopped being a vibe and became a gate with an exit code.

- **Pair every fix with a check at the layer above.** This generalizes the mutual-coverage
  idea: fix a bug at layer *N*, then add a forcing function at layer *N-1* that catches its
  recurrence. The fix isn't done until the check that prevents its return exists.
- **A catalog of gates**, each guarding one invariant (generalized):
  - **route ↔ client ↔ UI coverage** — a backend endpoint can't ship UI-unreachable; matched at
    *path-skeleton* granularity, not a coarse prefix, with a printed allowlist for the genuinely
    server-only routes so nothing is *silently* skipped.
  - **single write path** — consumer layers can never make a raw write; every physical mutation
    must route through the armed + rate-limited + audited gate. A monkeypatched raw write in a
    test proves the gate catches it.
  - **capability badging** — any feature whose component touches a risky or state-mutating
    primitive must declare it; the UI renders the badge. Risk is *allowed everywhere, but never
    silent.*
  - **docs-counts self-gating** — count claims in the docs must equal reality, and the document
    that says "there are **N** gates" is itself one of the things gated (so this very write-up's
    successor can't drift).
  - **changelog-per-change** — a branch that touches product source must also move the changelog,
    so the history stays current by construction instead of 50 PRs behind.
  - **domain purity** — no cross-project or banned tokens leak into the shipped tree.
- **One command, one source of truth.** Everything runs under a single `make gates`; CI calls
  the identical target. "Green locally" is *designed* to predict "green in CI" because they
  execute the same code.

## The completeness audit, as a workflow

When the backlog is fuzzy — *"are we missing anything important?"* — discipline needs a
**process**, not a vibe. I run a multi-agent audit:

1. **Fan out** independent dimension-auditors over the live tree (coverage, security, tests,
   docs, half-wired features, claimed-but-not-built, drift).
2. **Adversarially verify** every finding against the actual code before trusting it — this
   *kills false positives*. (One audit "found" a missing export feature that already existed; I
   almost rebuilt it. Verifying first saved the work.)
3. **A completeness critic** asks what the auditors themselves missed.
4. **Synthesize** a ranked, `file:line`-grounded build plan and **persist it to a runbook**, so a
   build can resume from it across sessions without re-discovering the gaps.
5. **Ship in small green waves**, checkpointing the runbook after each, so progress is never lost.

## Verify by byproduct — the negative control

A forcing function is only real if it **bites**. A green suite that *can't* fail is theater. So
each gate ships with a check that proves it catches a real violation — e.g. an accessibility
smoke test paired with a deliberately broken element that **must** be flagged, or the write-path
gate's monkeypatched raw write. If the negative control ever stops failing, the gate has gone
inert and *that* is the signal. I confirm gates by their byproduct (a real red), never by
assuming the code "should" work.

## What generalizes

A portable recipe for any feature-rich app where "done" is hard to see:

1. **Encode the contract as gates** — if a rule matters, make it fail an exit code, not a memory.
2. **Pair every fix with a check one layer up** — prevent the *class*, not the instance.
3. **One gate target, run by both local and CI** — so local-green truly predicts CI-green.
4. **Audit completeness as a process** — fan out, adversarially verify, critique, rank, persist.
5. **Negative-control every gate** — prove it bites; a suite that can't go red proves nothing.
6. **Ship in checkpointed waves** — small, green, resumable; the plan survives the session.

## Outcome

- A **42-finding completeness audit reconciled to zero open items**, each finding shipped as a
  verified, gated, checkpointed change.
- **14 forcing-function gates** under one command — every capability operable end-to-end (no
  backend-only or UI-only half-features survive), risk always badged, docs that can't drift.
- A safety invariant — **one audited write path** — made *structurally* unbreakable rather than
  trusted to discipline.
- All delivered at **$0 CI spend** (local gate + `[skip ci]`), with the gate target as the single
  source of truth so nothing silently diverged.
- A methodology that **transfers** to any large app: the way to make "we covered everything"
  a checkable claim instead of a hopeful one.

*Deliberately at the engineering-capability level; happy to walk through the gates and the audit
workflow on a call under NDA.*
