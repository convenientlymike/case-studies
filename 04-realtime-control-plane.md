# Real-time hardware control plane

*A premium localhost control plane for hardware controllers — live telemetry + a clean hardware-abstraction layer.*

> **Status:** built. The kind of low-level/UI-bridging work most full-stack engineers don't touch.

## Context

Working with hardware controllers means a stream of fast-moving, low-level data that a
human needs to see, interpret, and act on in real time — across very different transports
(a simulator, a probe, PCIe, serial, a remote bridge).

## Problem

Build a control plane that renders live device telemetry at a high refresh rate, presents
it cleanly enough to actually operate from, and abstracts over wildly different hardware
backends behind one interface — without the UI choking on the data rate.

## Constraints

- **30 Hz** live telemetry to a wall display without dropping frames or leaking memory.
- Multiple transports (sim → probe → PCIe → serial → remote) that must be swappable
  behind one contract.
- A control surface that's premium and legible, not an oscilloscope-grade wall of numbers.

## Approach

- A **hardware-abstraction layer (HAL)** with one interface and pluggable backends, so the
  UI and the device logic never care which transport is live.
- A streaming pipeline tuned for sustained 30 Hz with backpressure, not naive polling.
- A hardened API and a dense, dark, real-time operator UI.

## Architecture

- **Backend:** `FastAPI` (`Python`) — streaming endpoints + the HAL.
- **HAL backends:** simulator, probe, PCIe, serial, remote — one contract each implements.
- **Frontend:** `React` + `TypeScript`, a real-time telemetry view + a 30 Hz canvas render.
- **Cross-cutting:** a hardened, auth-gated localhost control plane.

## Outcome

- Sustained 30 Hz live telemetry with a clean operator experience.
- A swappable HAL that made adding a new transport a small, contained change.

*A sanitized, dummy-data public reference of this pattern is in progress.*