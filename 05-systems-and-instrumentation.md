# Custom Android platform & instrumentation R&D

*Where the abstraction ends: a custom Android platform on a hardened kernel, a GPU translation layer, and the instrumentation/RE to understand closed systems.*

> **Status:** research & engineering. Described at the capability level — the *transferable*
> engineering, not any specific application.

## Context

Running and analyzing real Android workloads on Apple Silicon means crossing several hard
boundaries at once: a Linux kernel, the Android platform, a GPU stack built for one API
that has to land on another, and closed binaries with no source or documentation. This is
the layer most application engineers never have to touch — and exactly where I go when a
higher-level approach hits a wall.

## Problem

Stand up a controllable, observable Android platform on Apple Silicon, get demanding
GPU-bound workloads rendering correctly through a non-native graphics stack, and build the
tooling to understand and instrument software whose internals aren't documented.

## Constraints

- A custom **GKI 5.15 Linux kernel** that boots clean and stays stable under real load.
- A graphics path that bridges **Vulkan** (guest) to **Metal** (host) on Apple Silicon
  without the renderer wedging.
- Closed, optimized binaries (including IL2CPP-compiled native code) with no source — so
  understanding them requires real reverse-engineering methodology, not docs.

## Approach

- **Kernel engineering:** built and patched a custom GKI kernel; debugged boot, driver,
  and stability issues at the source level.
- **GPU translation:** worked through a Vulkan↔Metal translation path (Mesa-family driver
  work) so guest rendering lands correctly on Apple's GPU.
- **Dynamic instrumentation:** used `Frida`-based instrumentation to observe and modify
  running behavior, with a disciplined, idempotent hooking methodology.
- **Reverse engineering:** a repeatable pipeline — static disassembly + xref analysis +
  runtime tracing — to map closed binaries, validated by re-running against the live system.

## Architecture / methodology

- **Kernel:** custom `GKI 5.15`, source patches, `C`.
- **Platform:** Android platform engineering, `Kotlin` for platform-layer modules.
- **Graphics:** Vulkan→Metal translation (Mesa-family), GPU debugging on Apple Silicon.
- **Instrumentation:** `Frida` (compiled/bundled agents), arm64 disassembly tooling.
- **RE methodology:** a six-step, target-agnostic pipeline (discover → hypothesize →
  instrument → verify → harden → document) that survives version drift via checksum and
  expected-bytes gates.

## Outcome

- A bootable, observable custom Android platform with a working GPU translation path on
  Apple Silicon.
- A reusable reverse-engineering and instrumentation methodology that transfers to any
  closed Android/native target.

*This entry deliberately stays at the engineering-capability level.*