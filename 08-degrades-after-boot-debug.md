# Root-causing a "passes at boot, silently degrades" reliability bug on a custom Android platform

*Another debugging war-story from [the platform R&D in 05](./05-systems-and-instrumentation.md): how rigorous layered diagnosis took an opaque "it works right after boot, then quietly stops" bug to a precisely-bounded root cause — fixing the tractable layers and **honestly banding** the last mile instead of claiming a clean win.*

> **Status:** active research & engineering, told at the methodology level — the *transferable*
> debugging discipline, not any specific application. General Android internals (`zygote`,
> `system_server`, SELinux, `app_process`) are named because they're public platform knowledge;
> the workload that runs on top is intentionally omitted.

## Context

I run a custom Android emulator on Apple Silicon. A privileged userspace **daemon** injects
per-process instrumentation into apps as they fork from `zygote`, and a **diagnostic probe**
verifies that instrumentation is actually live across ~250 checks. A green probe means the
platform is doing its job.

## Problem

The probe passed cleanly **right after boot**, then **silently degraded** minutes later — every
later app launch came up un-instrumented. "Works at boot, decays after" is one of the nastiest
reliability classes: the obvious test (launch once at boot) shows green, so the real bug hides
behind a **false-green**.

## Constraints

- **Each hypothesis costs ~8 minutes** — a full cold-boot cycle (boot → platform bring-up →
  probe). Iteration is expensive, so *guessing* is expensive.
- **The probe's own boot-time auto-run is the false-green trap** — it runs when instrumentation
  is still live. Measuring there hides the failure; the bug only appears on a *relaunched* process.
- **The live re-establish path is destabilizing** — the only runtime way to re-attach
  instrumentation cycles a core system service, which on this platform doesn't reliably come
  back. "Just restart it" makes things *worse*.
- **Churn degrades the box** — repeated reboots themselves erode stability, so a runaway
  experiment loop is self-defeating.

## Approach — the methodology that did the work

- **Kill the false-green first.** The single highest-leverage move was refusing to trust the
  boot-time pass: every measurement went on a **relaunched** process, after the daemon had gone
  quiet. That one discipline turned "intermittent and mysterious" into "100% reproducible."
- **Necessary vs. sufficient signals.** "Daemon process alive" (`pidof` returns a PID) is
  *necessary* but not *sufficient* — and it had been misleading me. The sufficient signal was the
  per-process bind actually firing, read from logs and `/proc/<pid>` forensics, not from the
  daemon merely existing.
- **`ps`-args forensics over assumption.** The fix hinged on a single launch flag. Reading the
  daemon's *actual* `argv` from `ps` — not the truncated `/proc/cmdline`, not the source I
  *assumed* was running — is what proved which value was live.
- **Adversarially verify your own hypotheses.** A parallel analysis proposed a slick
  "native-layer" fix for part of the problem. Before building it, I checked the assumption
  against how the probe *actually* reads that value — and **refuted my own plan**: the native and
  framework read-paths are different code, so the hook would have missed entirely. Disproving your
  own promising idea before spending a day on it is the cheapest win in debugging.
- **Multi-agent decomposition for the map.** I fanned out independent reads of the platform's
  bring-up and the probe's checks to map every failing signal to its owning mechanism, then
  synthesized a ranked, layer-by-layer fix plan — each layer's confidence explicitly banded.

## Root cause — a five-layer chain

The single symptom resolved into a stack of independent causes, each verified before moving on:

1. A **SELinux denial** blocked the daemon from compiling the code it injects → it never loaded.
   *Fixed* — a scoped policy grant, hosted in a tracked module so it survives a re-flash.
2. A benign-but-flagged **emulator marker** property → *fixed*, deleted natively at boot (closed
   with zero dependency on the daemon).
3. The daemon launched with a **reconnect cap of 3**. A core-service cycle during bring-up
   exhausted those 3 retries and the daemon **exited** — and nothing respawned it. *Fixed* by
   raising the cap so it waits the service out (verified: lifetime ~5 min → **34+ min**, surviving
   the cycle).
4. The per-process bind is established **only at a fork-time event while the daemon is alive** — a
   daemon respawned *after* that event is alive but **orphaned** (proven: a post-mortem respawn
   yields zero instrumentation).
5. The only *live* way to re-trigger that event **crashes a core service** on this platform — so
   re-establishment is a cold-boot-only operation.

## The honest frontier

This is the part most write-ups omit: I **band what's proven vs. what isn't.** Layers 1–3 are
**fixed and verified by byproduct.** The last mile — making the bind reliably survive bring-up's
final restart — is **modeled, not yet proven.** I have a specific, evidence-backed hypothesis (drop
a now-redundant second restart that was only added back when the daemon still died early) and a
fallback (move the tractable checks to the always-on injection layer, which caps coverage at a
known, honest number). Calling that "done" would be a guess; surfacing exactly where the proven
part ends — in a resume doc the next session starts from — is the deliverable.

## What generalizes

A target-agnostic discipline for "passes at boot, degrades after" bugs:

1. **Distrust the convenient measurement** — find the false-green and measure past it (relaunch,
   don't reuse the boot instance).
2. **Separate necessary from sufficient** — "process alive" ≠ "feature working"; verify at the
   sufficient layer.
3. **Read the live truth** (`ps` args, `/proc`, logs) instead of the source you *assume* runs.
4. **Refute your own RCA** before you write code against it.
5. **Peel layered causes one at a time**, verifying each by byproduct before touching the next.
6. **Band every claim** — VERIFIED vs MODELED — and say plainly where the proof ends.

## Outcome

- The bug went from **opaque and intermittent** to a **precisely-bounded, reproducible five-layer
  chain** — three layers fixed-and-verified, the frontier documented to the line.
- A **reusable "degrades-after-boot" debugging playbook** (the six steps above) that transfers to
  any layered daemon/instrumentation system.
- A complete **resume artifact** — root cause, verified fixes, the next experiment, and the
  fallback — so the work continues without re-walking discovery.

*Deliberately at the engineering-capability level; happy to go deeper on a call under NDA.*
