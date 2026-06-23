# Operating a headless Android emulator from a phone — host-side WebRTC when the encoder path is broken

**Domain:** low-level systems · real-time media · WebRTC · remote access
**Stack highlights:** `Python` `aiortc` `WebRTC` `gRPC` `VP8/Opus` `Tailscale` `React/TypeScript`

---

## Context

I run a **custom Android emulator platform** for systems R&D — a Pixel-class image on a custom GKI kernel,
with the guest GPU translated to the host's Metal stack (a Vulkan→Metal layer). During long development
sessions I'm often away from the workstation, so I wanted to **view and fully operate the live emulator from
my phone** — real-time video *and* audio, touch and hardware-button input — from any network, as a lightweight
web app rather than a native build.

It sounds like a solved problem. It wasn't, for two reasons.

## Problem

**1. The standard tool can't encode.** `scrcpy` — the usual "mirror an Android device" answer — connects fine,
but **every codec errors out with zero-byte output** at every resolution. The cause is the custom GPU layer:
`scrcpy` asks *Android's* hardware encoder (`MediaCodec`) to encode the display surface, and the translated
render surface never reaches the encoder's input surface. All the on-device encoders are software-only and
fail the same way. There is no fixing this from the device side — the capture point itself is wrong.

**2. The control surface lives on a remote phone I can barely observe.** Whatever I built, I could only see it
through the phone's screen — so every layout or quality bug was a slow round-trip: change code, ask "how does
it look now?", squint at a screenshot, guess again.

The bar I set: **native-resolution 60fps video, with audio, full input, low latency, on any network, private,
and a Progressive Web App** I add to the home screen.

## Constraints

- **Capture must avoid the broken Android encoder entirely** — encode somewhere the translated frames already
  exist.
- **Cross-network by default** — I might be on a phone hotspot while the host is on home WiFi. No public
  exposure of the host.
- **Lightweight** — no heavyweight media server / proxy stack.
- **Debuggable** despite being a remote-only UI.

## Approach

**Capture host-side, not device-side.** The emulator exposes a gRPC control plane that serves the
*host-rendered* framebuffer (the frames the host already composited, post-translation), plus audio and input.
I built a small **`aiortc` (Python WebRTC) bridge** that:

- pulls the host framebuffer stream and encodes it **host-side to VP8** (libvpx, multi-threaded), and audio to
  **Opus** — never touching the broken on-device encoder;
- serves both as a standard WebRTC stream the browser **hardware-decodes**;
- routes touch and hardware-button input back over a **WebRTC data channel** to the emulator's input API.

Then three decisions that turned a demo into something usable:

**Adapt on the browser's *measured health*, not the send-bitrate.** The framebuffer stream is delta-driven, so
a *static* screen sends almost no data. A naive "low bitrate ⇒ slow link, drop the resolution" controller
reads that as a bad link and flaps the resolution down **on a perfectly fast connection**. Instead the browser
reports its real inbound health every couple of seconds (`getStats`: packet loss, frame freezes, decode drops);
the bridge steps the source resolution down one tier only on genuine congestion and back up after it stays
clean. Packet loss and freezes only happen when the link truly can't keep up — content-independent.

**Remote access over a private mesh, not a public tunnel.** WebRTC media is peer-to-peer UDP, negotiated from
each side's network candidates. A plain HTTP tunnel forwards the *page* and signaling but **not the UDP media**
— so the page loads and the video never connects. The clean fix is a **WireGuard mesh (Tailscale)**: both
devices get an address on one virtual network, the host advertises that address as a connection candidate, and
the media flows **directly over the encrypted mesh** — private, no port-forwarding, no public surface,
near-LAN quality from a different physical network.

**Debug the remote UI by telemetry, not screenshots.** I made the page **measure its own state and report it
to the host** — viewport and safe-area insets, every element's real rectangle, the decoded video dimensions,
the playback health — written to a file I read directly. That single change collapsed a five-screenshot
guessing loop into one read, and it caught the actual bugs by *number*: a video element silently rendering at
its intrinsic size (the "zoomed in" complaint), an iOS web-app viewport API under-reporting its height, and —
later — proof that a connection was riding the mesh rather than a fluke. The same measurement feeds an
on-screen debug overlay I can toggle.

## Architecture

```
   phone browser (PWA)            host (workstation)
  ┌──────────────────┐    WebRTC   ┌────────────────────────┐  gRPC  ┌─────────────────┐
  │ fleet selector   │◀═ VP8/Opus ═│ aiortc bridge          │◀──────▶│ Android emulator│
  │ stream + control │             │  • host framebuffer→VP8│        │ (custom GPU     │
  │  ▸ touch/keys ───┼── data ch ─▶│  • audio→Opus          │        │  translation)   │
  │  ▸ getStats ─────┼── health ──▶│  • input→emulator API  │        └─────────────────┘
  └──────────────────┘             │  • health-driven tiers │
        ▲ Tailscale mesh           └────────────────────────┘
        └ direct P2P over WireGuard ──┘     ▲ dashboard: launch/stop · open-stream
```

A fleet selector lists every running emulator and routes to the one you pick; a dashboard control panel
starts/stops the bridge and opens any device's stream; the whole surface is access-key gated, and the
emulator's own control plane is token-authenticated (a hardening step I found and closed by **verifying it
adversarially** — driving the raw control plane myself to confirm it was reachable, then confirming the fix
rejected the same call).

## Outcome

- **Native-resolution 60fps video + audio + full input**, on the LAN and — over the mesh — from an entirely
  different network, **private and end-to-end encrypted**, as an add-to-home-screen PWA.
- A **fleet-aware** bridge (one process serves every device) with adaptive quality, host-side telemetry, a
  dashboard lifecycle control, and a test + static-analysis suite guarding it.
- Three reusable methods I now reach for elsewhere:
  1. **Telemetry-beacon debugging** for any UI I can't directly inspect — have it *measure and report itself*
     instead of debugging by screenshot.
  2. **Adapt streaming on measured playback health, not bitrate**, when the source is delta-driven.
  3. **A mesh, not a tunnel,** for cross-network WebRTC.

The honest open edge: over a **cellular hotspot**, carrier-grade NAT usually defeats peer-to-peer traversal
and the mesh falls back to a relay — adding latency the loss/freeze controller doesn't yet weight. The next
iteration makes adaptation **round-trip-time and relay aware**: detect the relayed path, clamp resolution and
bitrate harder, and back off on latency growth rather than only on loss.

---

*Details generalized; proprietary specifics removed.*
