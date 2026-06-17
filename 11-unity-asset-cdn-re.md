# Capturing a game's runtime-resolved asset-CDN — from a binary const to a passive memory pointer-chase

*A reverse-engineering war-story: how I recovered the exact CDN base URL a shipping Unity game computes at
runtime — without guessing, without touching the running app's behavior, and verified by byproduct — then
turned it into an infallible, integrity-checked asset pipeline.*

> **Status:** research & engineering, told at the methodology level — the *transferable* RE discipline, not
> any specific title. Upstream tech is named (Unity, IL2CPP, Addressables, UnityPy, Google Cloud Storage);
> the game and vendor are intentionally omitted. The work is first-party asset datamining (the same thing
> open datamine communities do): no exploit, no account interaction — observe-only.

## Context

I was building a companion dashboard on a game's *real* art — sprites, 3D models, FX — extracted **first-party**
from the device instead of leaning on a third-party mirror. Modern Unity games stream assets on demand via
**Addressables**, so the device's on-disk cache only holds the fraction you've encountered (here ~35%). The
rest lives in the vendor's CDN, addressed by a base URL the game resolves **at runtime**. To get the full set
first-party, I needed that exact base URL — as the game itself forms it.

## Problem

The base wasn't a literal I could grep. The Addressables catalog and the app's `aa/settings.json` both
referenced it only as a **profile-variable placeholder** (the CDN base), substituted at load. A naïve guess
(host + the obvious path) returned `404 NoSuchKey`: the host was right, the path was incomplete. And the
*resolved* string wasn't sitting in memory either, because Addressables substitutes the variable at
**request** time, not catalog-load time — so scraping the heap for the finished URL finds nothing.

## Constraints

- **Capture, don't guess.** A wrong or hand-assembled URL is fragile and a tell. I wanted the exact string the
  binary produces, derived from the binary.
- **Observe-only.** No injection, no hooking, no perturbing the live process — a passive read or nothing.
- **Verify by byproduct.** "I think this is the URL" isn't done; a ground-truth check is.

## Approach — the methodology that did the work

1. **String-hunt the IL2CPP dump.** The dump yielded a host const and a base-path const
   (`https://…/<AppPrefix>`) — but the *full* base was assembled in code, which is why the static guess 404'd.
2. **Read the static config for the shape.** The app's Addressables `settings.json` gave the URL **template**
   (`{CdnBase}/bundles/<platform>/catalog_<n>.hash`) — so I knew exactly what the runtime value slots into.
3. **Disassemble the getter.** I disassembled the CDN-base accessor (located via the IL2CPP method table). It
   wasn't returning a constant — it **concatenates a static string with a suffix pulled from a service
   singleton at runtime.** That single read explained *both* mysteries: why it's not a literal, and why the
   resolved string is transient in memory.
4. **Passive memory pointer-chase.** With the accessor's exact field offsets in hand, I read the value the
   safe way: a read-only walk of `/proc/<pid>/mem`. The non-obvious part — the IL2CPP library loads as
   **non-contiguous segments**, so a flat `load_base + virtual_address` reads garbage. I converted each
   static-field VA → file offset (via the local binary's ELF program headers) → runtime address (via the
   matching `/proc/maps` segment's file offset), then dereferenced the two managed-string fields.
   **Reading the *known* base const at the first field was a built-in correctness gate:** it came back exact,
   which proved the address math — so I could trust the second field, the unknown runtime suffix.
5. **Verify by byproduct.** Concatenated base + suffix → fetched the resulting catalog-hash URL → it returned
   the **exact hash the device had cached on disk.** Byte-for-byte ground-truth match. Captured, not guessed.

## From capture to a pipeline that can't fool itself

A captured URL is only useful if the bulk fetch is trustworthy, so I built the fetcher to the same bar:

- **Replicate exactly** — the game's real `User-Agent` (down to the Unity build) and the catalog-resolved path.
- **Verify every byte** — each download is checked against the CDN's `x-goog-hash` MD5 **and** content-length
  **and** the Unity bundle magic. A **negative control** proves the check bites: a deliberately-tampered buffer
  is *rejected*, an authentic one accepted — run every time, so the gate can't silently rot into a no-op.
- **Resumable + idempotent** — a manifest checkpoints state; re-runs skip already-verified bundles.
- **Minimize + blend** — throttled with jitter at the game's own cadence; the static public CDN is not the
  game's monitored RPC, but discipline is free.

## What generalizes

A target-agnostic recipe for any Unity/IL2CPP/Addressables title — and, more broadly, for *any* value a
program computes at runtime rather than storing as a constant:

1. **Static-hunt first** (strings, config) to bound the shape and find the building blocks.
2. **Disassemble the accessor** when the value isn't a literal — learn *how* it's built, not just *what* it is.
3. **Prefer a passive read** (`/proc/mem`) over injection when you only need to observe.
4. **Respect the real memory layout** — non-contiguous segments mean VA→file-offset→runtime, not flat math.
5. **Build a correctness gate into the read** — recover a value you *already* know alongside the unknown, so a
   wrong offset fails loud instead of returning plausible garbage.
6. **Verify by byproduct** — round-trip the result against independent ground truth before you trust it.
7. **Make the downstream infallible** — exact replication, per-item integrity checks, a negative control that
   bites, resumable + reversible.

## Outcome

- The runtime-resolved CDN base **recovered and verified** — fully passive, observe-only, zero perturbation of
  the running app.
- An **integrity-checked, resumable fetcher** that completes the first-party asset set (hundreds of bundles:
  sprites, 3D meshes, FX, audio) — version-pinned and ours, no third-party dependency.
- A **reusable methodology** (the seven steps above) that holds for any runtime-computed value in a native or
  managed binary.

*Deliberately at the engineering-capability level; happy to go deeper on a call under NDA.*
