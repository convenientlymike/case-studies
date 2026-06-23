# A verified-reputation network for the skilled trades

*A reputation layer for trades hiring — turning a worker's reliability into a portable, verified, ownable asset so an owner can tell the grinders from the flakes **before** they hire. Design + a single-player v0.*

> **Status:** private / early — design + a v0 MVP. Client-identifying details removed; numbers marked `[…]` are intentionally generalized. This write-up is about the product thinking and the system design, not a shipped platform.

## Context

Skilled trades that run on heavy equipment — concrete pumping, excavation, paving, CDL trucking — share one chronic, expensive problem: finding and keeping operators who actually show up and grind. The work is hard (long hands-on weeks on dangerous machinery), the reliable ones are rare, and a single no-show can cascade into a blown, weather-gated job and a lost client.

## Problem

Owners feel it as a hiring problem, but it isn't a *pay* problem: even at well-above-market wages `[…]`, owners still couldn't keep reliable people — most hires turned out flaky. That reframes it as a **selection** problem, not scarcity or compensation: you can't tell the reliable grinders from the flakes *before* you hire, and you bleed the good ones afterward. The obvious software answer — a two-sided labor marketplace — is a known graveyard: you can't software your way out of a supply shortage, matching has no moat, and even a **>$750M-funded incumbent exited construction staffing**. The real problem was to build something *defensible* that actually attacks selection, without walking into the marketplace trap.

## Constraints

- **No cold-start liquidity to lean on.** A two-sided marketplace needs both sides on day one; there were neither.
- **The buyers won't adopt software.** Small trades owners run on phone + paper + memory — a `[…]`-person back office for dozens of field operators — and they buy *outcomes*, not tools to configure.
- **It's a legal minefield.** Anything that scores or screens workers walks straight into EEOC adverse-impact, FCRA (background / driving records), defamation (owner reviews), BIPA (identity), and CCPA. Get the data design wrong and it's a lawsuit, not a feature.
- **Intellectual honesty required.** It's easy to over-claim an "AI reliability score" or a "data moat" — neither is real at small scale. The design had to separate what's genuinely defensible from wishful thinking.

## Approach

- **Sell the outcome; bootstrap single-player.** Instead of a marketplace, a single-player **owner tool** that's useful with *zero* other users ("come for the tool, stay for the network") — intake, structured screening, a paid trial-shift scorecard, and a per-worker record of verified facts. Reputation accretes as a *byproduct* of the owner's own bookkeeping; the network emerges on top of a tool that already works solo.
- **Make reputation the product, not matching.** The defensible core is a **verified, provenance-stamped, disputable reliability record** — built only from owner-attested facts against real scheduled shifts, never worker self-report. Portable across employers, it becomes a career asset the worker wants to keep current, and a cross-shop graph that incumbents (who hand off at the intro and store nothing durable) structurally can't accumulate.
- **Design the compliance *in*, not *on*.** The reputation/verification schema was built around the law from the first line: reputation is **informational decision-support, never an automated hiring gate** (EEOC); background / driving data is **vendor-isolated and consent-gated** (FCRA, with the §615 adverse-action two-step); negative events are **disputable factual flags** (showed-up Y/N), never free-text character attacks (defamation); identity and privacy gated for BIPA / CCPA. The legal design *is* the product design.
- **Multi-agent design + adversarial critique.** The product was designed by a panel of agents, then stress-tested by three independent adversarial critics — a *simplicity* critic (cut anything a solo founder couldn't ship in ~4 weeks), a *marketplace* critic (cold-start / disintermediation / no-moat), and a *legal* critic (the landmines above). The critics killed the "AI data-moat" narrative and the inflated economics, and forced the honest framing below.

## Architecture

- **v0 app:** `Next.js` + `TypeScript` + `Tailwind`, a `Prisma`/`Postgres` schema (workers, employers, work history, scheduled shifts, trial-shift scorecards, an append-only reliability-event store, a decision audit log). **Mock-first** — the whole MVP runs on typed fixtures with *no database required*, so the instruments are usable before any integration exists.
- **The reliability record:** an **append-only, provenance-stamped event store** — every signal (showed-up, no-show, callout, completed-term, trial-shift score, background status) rooted in a real scheduled shift and an attesting owner, behind a write-guard with a test that *bites* (a shiftless no-show insert is rejected).
- **Guardrails enforced in the UI, not just the docs:** no computed score in v0 (a score at n=1 is theatre — it's a raw readout of verified facts), a neutral default sort, **show-all** (thin records surfaced, not buried), background data shown as status only, disputable flags, and a "decision-support · human-in-the-loop" banner on every reliability surface.
- **Honest sequencing:** v0 is a genuinely useful single-player ops tool; the cross-shop **network — the actual moat — is deliberately deferred**, gated behind a counsel review and a second paying owner.

## Outcome

- A shippable v0 MVP plus a complete design package — product spec, screens, data model, the reputation/verification schema, and a counsel-ready legal checklist — with the defensible wedge (reputation-as-portable-asset) separated cleanly from the parts still unproven.
- **The intellectual honesty is the headline:** the design states plainly that v0 proves the *tool*, not the *network*; that the "data moat" isn't real until cross-shop scale exists; and that the economics are unvalidated until a second owner pays. Surfacing what can't yet be proven — rather than dressing an MVP as a launched platform — was treated as a deliverable.

*Deeper architecture and the reputation/verification schema available on request.*
