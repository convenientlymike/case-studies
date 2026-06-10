# Root-causing a GPU-driver crash in a from-source Metal↔Vulkan stack

*A concrete reverse-engineering war-story behind [the platform R&D in 05](./05-systems-and-instrumentation.md): how I took a one-line `SIGABRT` to a verified, driver-level fix — and the methodology that got there without chasing the wrong bug.*

> **Status:** research & engineering. Told at the methodology level — the *transferable*
> debugging discipline, not any specific application. Open-source components are named
> (they're upstream: Mesa **KosmicKrisp**, **gfxstream**, QEMU); the workload that runs on
> top is intentionally omitted.

## Context

I run a custom Android emulator on Apple Silicon whose graphics path is built **from source**:
guest **Vulkan** → host **Metal**, via a Mesa-family driver (**KosmicKrisp**) paired with the
**gfxstream** host backend. Because I own both ends of that stack, a crash in it isn't
someone else's bug to wait on — it's mine to fix at the driver level.

## Problem

The emulator was `SIGABRT`-ing intermittently — a recurring class of crashes (≈8 in a week)
that killed the *entire* VM mid-session, with no clean repro. "It crashes sometimes" is the
hardest kind of bug: rare, fatal, and easy to mis-attribute.

## Constraints

- **No repro on demand** — the trigger was a transient GPU stall, so the fix had to be
  reasoned from crash artifacts + source, not a debugger sitting on the fault.
- **A shared driver across configs** — bumping the guest OS had *not* fixed it (the crash
  lives in the host GPU translation layer, which the guest version doesn't touch) — a
  distinction that's easy to get wrong and waste days on.
- **Asserts ship live** — the from-source driver build had `NDEBUG` unset, so a bad handle
  hit a hard `assert()` → `abort()` of the whole process, not a recoverable error.

## Approach — the methodology that did the work

- **Crash-report-first triage.** The macOS crash report (`.ips`) named the faulting frame in
  one read — the compositor's fatal log path and the driver's fence cold-paths. The lesson I
  keep relearning: *read the crash log before forming a hypothesis* (I'd burned ~10 rebuilds
  on wrong guesses on a prior bug before internalizing this).
- **Parallel RE + adversarial verification.** I fanned out independent reads of the
  open-source driver/glue to root-cause each crash signature — then **verified every cited
  `file:line` against the actual source before trusting it.** That step *refuted* a
  plausible-but-wrong root cause (a "two-mutex race" theory that collapses once you see the
  function actually holds both locks). Verifying findings against source — not accepting a
  confident-sounding RCA — is the difference between a fix and a guess.
- **A completeness check that re-targeted the fix.** The loud, frequent crashes turned out to
  be an artifact of a *deprecated, already-disabled* config path. The crash that actually
  mattered was a different one entirely — a fatal fence-wait timeout in the host compositor.
  One check saved me from building the right fix for the wrong bug.
- **A systemic bug found along the way.** A mismatch between a device's *logical* name and its
  *process* name had silently blinded an entire class of per-device tooling (the crash
  watchdog, the graceful-stop helper) — they were matching the wrong identifier and quietly
  doing nothing. Fixed at the **root** by aligning the names, so every tool works without
  per-tool patches.

## Root cause & fix

The crash: a deferred lambda waited on a "compose-complete" fence and treated a **`VK_TIMEOUT`
/ device-lost** as fatal (`VK_CHECK` → `abort()`). So a *transient* GPU stall — recoverable by
nature — was being turned into a hard crash of the whole emulator.

The fix matched the layer and the severity: a fence-wait timeout is **recoverable**, so log it
and drop the frame instead of aborting. I changed it at the host compositor (the right layer,
not a band-aid in the driver), rebuilt the backend **from source** (incremental — one
translation unit + a relink), deployed the rebuilt dylib, and **verified by byproduct**:
confirmed the *running* process had actually mapped the patched binary (not just that the file
on disk changed), then soaked it.

As defense-in-depth, an always-on **watchdog** auto-recovers a full VM crash (graceful restart,
rate-limited) — the net that keeps a session alive while a root fix lands, and the forcing
function that means the next occurrence self-heals instead of needing a human.

## What generalizes

A target-agnostic crash-RE discipline that transfers to any native/driver crash:

1. **Crash-report-first** — the tombstone/`.ips` usually names the fault; stop guessing.
2. **Adversarially verify the RCA against source** before changing code — refute it if you can.
3. **Run a completeness check** — make sure the loud crash is the one that matters; re-target.
4. **Classify recoverable vs fatal** and fix it at the right layer (recover, don't `abort()`).
5. **Verify by byproduct** — confirm the live process loaded the fix; soak, don't assume.
6. **Pair every fix with a net** at the layer above (here, the auto-recovery watchdog).

## Outcome

- The recurring crash **eliminated**: from crashing within a session → **hours of continuous
  stability**, soak-clean on the same process, zero new crash reports.
- A **reusable GPU-driver crash-RE methodology** (the six steps above) that holds for any
  closed or from-source native target.
- Reliability tooling **hardened** — a whole class of "silently does nothing" tooling bugs
  closed at the root.

*Deliberately at the engineering-capability level; happy to go deeper on a call under NDA.*
