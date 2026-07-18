# A read-only network assessment practice — and a recon tool that refuses to lie or run unauthorized

*How I turned "map a network, find what's exposed, fix it" into a repeatable consulting engagement — built
around two disciplines enforced in code: it **never sends a state-changing packet**, and it **never presents a
guess as a fact**.*

> **Status:** an active freelance practice, told at the methodology level. The reusable tool + methodology are
> open source ([`lan-recon`](https://github.com/convenientlymike/lan-recon)); real client engagements stay
> private. All addresses below are in the RFC 5737 documentation range and all device names are generic — no
> real network is shown. Everything described is **authorized, read-only, defensive** work.

## Context

Most small and mid-sized businesses run networks nobody has ever inventoried. A consumer mesh router, a decade
of accumulated devices, a forgotten port-forward, an unknown always-on box in the closet. They don't need a
Fortune-500 red team — they need someone to honestly map what's there, find the handful of things that
actually matter, and fix them. I built a practice to fill that gap, and the tooling + methodology to make each
engagement faster and more consistent than the last.

## Problem

Two failure modes make most "network scans" worthless to a business owner:

1. **They're unsafe or reckless.** A scanner that logs in, writes, or floods can knock over a fragile device
   (a POS terminal, a medical or industrial box) — the assessment causes the outage. And a tool that runs
   against whatever you point it at, with no authorization discipline, is one typo away from being a crime.
2. **They're dishonest by omission.** A scanner prints 200 lines of raw output and calls a device a "macOS
   host" because a MAC prefix said Apple. Half of it is inference dressed as fact. The owner can't tell which
   findings are real, so they trust none of them — or worse, all of them.

I wanted an assessment a business owner (and their insurer, and their IT person) could actually rely on.

## Constraints

- **Authorization first, always.** No packet leaves a tool until the network owner has signed a scoped
  consent. This is the line between security work and a computer-crime — and documenting it is a selling
  point, not friction.
- **Read-only by default.** Discovery must be non-state-changing and non-disruptive. Most SMB value is found
  without ever sending a write.
- **Honest to the byproduct.** Every claim must be traceable to the evidence that proves it, and graded when
  it can't be.
- **Repeatable, not bespoke.** The methodology and tooling must make engagement N+1 cheaper than N.

## Approach

A four-phase methodology — framework-aligned (NIST SP 800-115 / PTES) but right-sized for a five-person office
instead of a compliance binder:

```
1. SCOPE & AUTHORIZE    written scope, rules of engagement, verified ownership, read-only vs. active
2. DISCOVER & INVENTORY  enumerate every host, identify it, map the topology  ← the tool lives here
3. ASSESS VULNERABILITY  exposed/misconfigured/unpatched surface, ranked by real business impact
4. REPORT & REMEDIATE    prioritized plain-English report, remediation plan, re-test to green
```

Two disciplines run through all four phases and are the whole differentiator:

**Authorization as a forcing function.** The reconnaissance tool ([`lan-recon`](https://github.com/convenientlymike/lan-recon))
literally refuses to run until scope is affirmed — an interactive re-type of the target subnet, or a recorded
`--authorized-by` in automation. The discipline is built into the tooling, not left to memory, and it's
negative-control-tested in CI (a run without an authorizer must fail).

**Honesty-banding.** Every observation is graded by the strength of its evidence:

- 🟢 **VERIFIED** — a byproduct proved it (a service banner, an mDNS advertisement, a NetBIOS name).
- 🟡 **INFERRED** — reasoned from indirect signal (a MAC vendor, a TTL, a port pattern). Useful, but labeled.
- ⚪ **UNKNOWN** — live but not identifiable from the wire. Honestly flagged, never guessed.

The rule: a claim is never promoted a band without a byproduct, and findings inherit the weakest band in their
chain. An honest "unknown" is worth more than a confident wrong answer — it's exactly the device an assessor
recommends *physically* identifying.

## Architecture (the tool)

`lan-recon` automates Phase 2 as a read-only "technique ladder" — cheapest, least-intrusive signal first —
grading each host by the strongest evidence obtained:

```
  live host?   ping sweep + ARP          →  VERIFIED (host is live)
  vendor?      MAC OUI lookup            →  INFERRED (NIC maker — a hint)
  services?    TCP connect-probe         →  VERIFIED (port is open)
  identity?    banner / mDNS / NetBIOS   →  VERIFIED (device named + pinned)
  OS family?   TTL                       →  INFERRED (spoofable hint)
```

Design properties, each a deliberate guardrail:

- **Read-only by construction** — only ping, ARP, TCP connect-probes, banner reads, mDNS browse, anonymous
  NetBIOS status, and vendor lookup. No path in the code logs in, writes, or authenticates. An open port is
  only ever reported as *open* — never promoted to a confirmed service until a banner proves it.
- **Bounded** — every probe is timeout-wrapped; nothing loops or hangs.
- **Never hardcoded** — subnet, jump host, output dir, and port list are all parameters. It can probe from an
  on-site jump host over SSH so traffic originates inside the target network, not from a laptop across the
  internet.
- **Authorization-gated** — the forcing function above.

## Outcome

Applied to a real small-office network (a ~30-host consumer-mesh SOHO — sanitized here), the methodology
produced a legible, evidence-graded inventory and a short, honest findings register: **zero critical or
high findings** (no internet-facing remote access, no exposed databases, no telnet — remote access cleanly
consolidated on a single VPN rather than scattered port-forwards), and a handful of **defense-in-depth**
improvements that actually matter for that business:

- A **flat network** — consumer IoT (camera, speakers, streaming players) sharing one L2 segment with the
  business PCs. One compromised smart device could reach the file share directly. → *Segment IoT onto the
  guest network.*
- One **unidentifiable always-on device** (a withheld-vendor OUI, every port closed, no advertisement) that
  couldn't be accounted for from the wire. → *Physically identify it.*
- Standard hardening on the file-share host, the network printers, and the remote-management surface.

The headline of the report was as valuable as the findings: *your network is in reasonable shape — here are
the two things to do first.* That honesty — separating the verified from the inferred, and stating plainly
what's already good — is what makes an assessment something a business acts on instead of files away.

**What made it work:** the two disciplines enforced in code. A tool that can't run unauthorized and can't dress
a guess as a fact produces an assessment that's both safe to run and safe to trust — which is the entire job.

---

*Reusable tool + full methodology: **[`lan-recon`](https://github.com/convenientlymike/lan-recon)** (open
source, MIT). Happy to walk through the approach or a scoped engagement on a call.*
