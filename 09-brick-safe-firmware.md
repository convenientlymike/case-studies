# Brick-safe firmware writes

*Making it impossible to brick a device — a safe-by-construction write path + a mock-first firmware
lifecycle, with the core extracted as an open-source library.*

> **Status:** built. The write-safety layer of the [real-time hardware control plane](./04-realtime-control-plane.md),
> hardened and then distilled into a public reference: **[`bricksafe`](https://github.com/convenientlymike/bricksafe)**.

## Context

The control plane in [case study #04](./04-realtime-control-plane.md) abstracts over many hardware
transports (sim → probe → PCIe → serial → remote). Reading telemetry is safe. **Writing firmware is
not.** A flash to real hardware is irreversible and unforgiving — and the operator needed the full
lifecycle from the dashboard: detect a device on plug-in → read what's on it → back it up → load new
firmware → flash → verify → onboard.

## Problem

Build that lifecycle so that the catastrophic outcome — **a bricked device** — is impossible *by
construction*, not by careful operators remembering to be careful. And do it before the physical
hardware was even in hand, so the day the hardware arrived, day-one worked.

## Constraints

- **Irreversible writes.** Every failure mode is permanent: flashing before a backup exists; writing
  a *placeholder* (the synthetic stub used for hardware-free development) to a real device; clobbering
  an unexpected state; a write that "succeeds" but was never read back; a real-device call that
  silently no-ops because the hardware wasn't actually there.
- **No hardware yet.** The entire control plane had to be built and tested against stand-ins, with a
  seam that real transport drivers drop into later with *zero* rewiring.
- **Operator-grade.** A guided flow a human drives from a dashboard, with a hard stop at every unsafe
  step — not a pile of CLI footguns.

## Approach

- **One write-gate, many checks.** Every physical write goes through a single gate; nothing else
  touches a device's raw write. A write must clear, in order: *armed → rate-limit → verified-backup →
  not-a-placeholder → read-before-write (CAS) → undo-snapshot → write → confirm-readback → audit.* Any
  failed check is a typed refusal, logged.
- **Backup-before-write as a HARD gate.** A device can't be written until a *verified* backup of its
  current image exists — "verified" meaning it passed a restore-test (re-read, re-hash), because a
  backup that can't be read back is not a backup. Restore-to-as-shipped is always possible.
- **A byte-level placeholder guard.** A placeholder stub is refused on its *bytes*, not a label — so a
  development stub can never masquerade as real firmware and reach real hardware.
- **Fail-loud real backends.** A real-device backend is *unavailable* and *raises* until its transport
  is present. It never silently no-ops — the worst failure is thinking you flashed when you didn't.
- **Mock-first lifecycle.** A complete in-memory device backend runs the whole loop with zero
  hardware, behind the same interface a real driver implements — so the lifecycle was fully walkable
  (and the safety gates fully tested) before any device existed.
- **A guided onboarding wizard** that composes all of the above into a gated, step-by-step flow, with
  auto-launch when a device is detected plugging in.

## Architecture

- **Safety core (extracted, open-source):** [`bricksafe`](https://github.com/convenientlymike/bricksafe) —
  the write-gate, the restore-tested backup store, the byte-level placeholder guard, and mock + fail-loud
  device backends. Zero runtime dependencies, `mypy --strict`, every safety invariant proven by a
  negative-control test. (`Python`)
- **Control plane:** `FastAPI` backend exposing the lifecycle (discover / read-current+backup / import
  / preflight / flash / verify / undo) over the same HAL as #04; a `React` + `TypeScript` dashboard
  with the guided onboarding wizard + hotplug auto-launch.
- **The seam:** real transport drivers implement one device contract; discovery surfaces a real device
  only when it actually enumerates, so the mock fleet is unchanged until hardware arrives — then the
  real path lights up with no further wiring.

## Outcome

- The entire detect → backup → import → flash → verify → onboard lifecycle is **operable today against
  a synthetic fleet**, gated so the irreversible-write footguns are structurally impossible.
- Shipped as `[…]` small, independently-verified increments — the brick-safety defenses (mock-byte
  guard, backup hard gate, fail-loud backends) deliberately landed *before* any real-flash capability.
- The reusable core is public as **[`bricksafe`](https://github.com/convenientlymike/bricksafe)** —
  domain-neutral, runnable in one `python examples/demo.py`, applicable to any FPGA / MCU / embedded /
  IoT fleet tool.

*Numbers marked `[…]` are intentionally generalized.*