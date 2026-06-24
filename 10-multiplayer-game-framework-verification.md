# Building a commercial real-time multiplayer game-server framework — and a verification ladder that caught what the mocks couldn't

*A ground-up, standalone server framework + a defensive, server-authoritative anti-cheat — and the hero of the story: a three-tier verification ladder where a 289/289-green mock suite hid eight real bugs that only a real-database gate exposed.*

> **Status:** built (commercial, in active development). Told at the architecture level —
> the *engineering*, not the platform it targets. The game-modding platform, the game, and
> any specific anti-cheat target are intentionally omitted; `Lua`, `TypeScript`, `React`,
> and `MariaDB` are named because they're generic. Anti-cheat work here is **defensive and
> server-authoritative only** — the architecture that makes the server, not the client, the
> source of truth.

## Context

I'm building a commercial **real-time multiplayer game-server framework** for a major
game-modding platform — and a **defensive, server-authoritative anti-cheat architecture** to
go with it. Both are **ground-up and standalone**: no off-the-shelf base framework, no
inherited event model, no borrowed schema. Every load order, every network contract, every
persistence guarantee is something I designed and own end to end.

That ownership is the point. On a platform where the client is untrusted by definition, a
framework that defers to it for anything that matters — money, identity, permissions — is a
framework that ships exploits. The whole architecture is organized around one inversion: **the
server is the only source of truth, and the client is treated as hostile input.**

To keep that bar enforceable rather than aspirational, the work sits on two written
foundations: a **37-spec architecture blueprint** (the *what* — domains, contracts, data
shapes) and a **50-doctrine engineering library** that functions as the project's constitution
(the *how* — a charter plus specialized doctrines for security, validation, performance,
testing, and AI-augmented development). A claim in a spec that no doctrine can enforce is a
claim that will rot; a doctrine with no spec to apply to is academic. The two halves keep each
other honest.

## Problem

A multiplayer game server is a uniquely hostile place to build correct software:

- **Every inbound message is adversarial.** Clients can be modified to send malformed,
  out-of-order, out-of-range, or simply fabricated events at any rate. There is no such thing
  as a trusted field on the wire.
- **State is money.** Player economy, identity, and permissions are persistent and
  consequential. A duplicated item or a negative balance isn't a cosmetic glitch — it's a
  value exploit that propagates through every player who touches it.
- **It runs hot, forever.** Hundreds of concurrent players, dozens of events per second each,
  no restart window. A handler that's correct but slow is a tick-budget regression that
  degrades the whole server.
- **And the cruelest one:** the tests can lie. A server that talks to a *mocked* database will
  happily report green while doing something the real database would reject outright. Green
  mocks feel like safety. They are not.

The job was to build the framework so the first three are *structurally* handled — and to build
a verification strategy that refuses to be fooled by the fourth.

## Constraints

- **Untrusted client, always.** No event handler may trust a client-supplied value for
  anything authoritative. The server validates, re-derives, or rejects — never "takes the
  client's word for it."
- **Fail closed, never open.** When persistence is unavailable or a write is ambiguous, the
  system must *deny* the operation, not optimistically proceed. A fail-*open* path on anything
  touching state is a latent exploit.
- **The ledger is the truth.** Economy state isn't a mutable number you overwrite — it's the
  reduction of an append-only ledger of transactions, so every balance is reconstructible and
  every change is auditable.
- **Mock-green is necessary but never sufficient.** The fast test tier must exist (you can't
  iterate against a real database every keystroke), but it can never be the *authoritative*
  gate. Something that talks to real persistence has to have the final say.
- **Compliance is schema-level, not bolted on.** Right-to-erasure has to be expressible in the
  data model itself (crypto-shred: destroy the key, the PII is gone), not a best-effort delete
  script run later.

## Architecture — a server built to distrust its clients

### The spine

A **module loader driving a phase state-machine** brings the framework up in a deterministic
order — no module observes another in a half-initialized state, because the phase machine won't
let it. This is the unglamorous foundation that makes everything above it reasoning-about-able:
load order is a contract, not an accident of file naming.

### The net choke point — one hardened path for every inbound event

Every client→server event is forced through a **single middleware chain** before any handler
ever sees it. Nothing bypasses it. In order:

**malformed-shape rejection → unknown-event rejection → protocol/version check → resolve →
rate-limit → validate → dispatch.**

The power of the choke point is that adversarial input dies as *early* as possible and as
*uniformly* as possible. A garbage payload is rejected at stage one before it can allocate
anything interesting. An event the server doesn't recognize never reaches dispatch. A client
flooding a handler is rate-limited centrally, not per-handler-if-the-author-remembered.
Validation is the *last* gate before dispatch, so by the time a handler runs, the event is
well-formed, known, version-compatible, within rate, and type-valid — every handler inherits
that guarantee for free, and no handler can opt out of it.

### Domains on a ledger-as-truth foundation

Identity, permissions, and economy are modeled as **domains** over real **MariaDB**:

- **Write-through, fail-closed persistence.** Authoritative state is persisted on the write
  path; if the store is unavailable, the operation is refused rather than completed in memory
  and lost.
- **An append-only, partition-ready ledger** as the economy's source of truth — balances are
  *derived*, transactions are *immutable*, and the ledger is structured to partition as volume
  grows.
- **A GDPR crypto-shred schema** — personal data is keyed such that erasing the key renders it
  unrecoverable, making right-to-erasure a property of the schema rather than a fragile cleanup
  job.

### The premium presentation layer

The in-game UI layer is a hand-built **React 19 + Vite 6 + Tailwind** surface, **gamepad-first**
with fully rebindable keybinds and a focus-tracking interaction model (so input ownership is
never ambiguous between game and UI). It's browser-verified during development against a real
headless Chrome build — the UI is held to the same evidence standard as the server.

## The hero: a three-tier verification ladder

This is the part a senior reader should take away. The framework's correctness rests not on one
test suite but on a **ladder of three tiers, each catching what the one below it structurally
cannot.**

**Tier 1 — mock-native load test (license-free, sub-second).** The fast tier. It stands the
framework up against in-memory stand-ins for persistence and the platform, exercises the load
order, the net choke point, and the domain logic, and runs in under a second with no external
dependencies. This is what runs on every change — the iteration tier.

**Tier 2 — real-database boot-smoke (the authoritative gate).** A real server process against a
*real* **MariaDB** in Docker. No mocked persistence — the actual SQL, the actual schema, the
actual NOT-NULL constraints, the actual types. This tier is slower and heavier, and it is the
one whose verdict *counts*. Tier 1 can pass; only Tier 2 can certify.

**Tier 3 — synthetic-player population.** A tool that drives real, concurrent client sessions
against the running server, so N-player concurrency, rate limits under real load, and the choke
point under contention are exercised by actual traffic rather than asserted in a unit test.

### What the ladder caught — 289/289 green, eight real bugs

The lesson that justifies the entire ladder: **Tier 1 reported 289/289 green the whole time.
Tier 2 — the real-database gate — exposed eight distinct, real bugs that the mock suite was
constitutionally incapable of seeing.** Among them:

- A **money sign error** — a value-affecting arithmetic mistake the in-memory store happily
  accepted because it never enforced the real ledger's invariants.
- **Hydrate-vs-DB seeding gaps** — code that assumed in-memory hydration state that a fresh real
  database simply didn't have, so the mock's convenient pre-population was masking a real
  cold-start bug.
- A **fail-*open* create path** — a creation route that, against a real store, could proceed
  when it should have denied. Exactly the class of bug the "fail closed" constraint exists to
  prevent, and exactly the class a forgiving mock hides.
- A **column / NOT-NULL mismatch** — the code's notion of the row didn't match the real schema's
  constraints; the mock had no constraints to violate, so it never complained.

None of these were exotic. Every one was the *kind* of bug that ships when "the tests pass"
means "the tests that don't touch reality pass." The real-database gate caught **all eight** —
and that result is the single most load-bearing piece of evidence in the whole project:
**mock-green is a necessary checkpoint, never a sufficient one.** The headline a project earns
from a green mock suite is no stronger than the strongest tier that has actually run.

## Biting gates — forcing functions that go red on a real violation

A gate that can't fail is theater. Every gate in the project ships with proof that it *bites*.

- **An 11-check static gate** runs on the tree — secrets, server-authority violations,
  unguarded events (a handler reachable without passing the choke point), focus-arbiter misuse
  in the UI, performance-budget regressions, and more. Each of the eleven ships **with a
  negative-control fixture**: a deliberately-broken input the check **must** flag.
- **A `--self-test` meta-forcing-function** runs every check against its bad fixture and asserts
  every one goes **red**. This is the gate that guards the gates: if a check ever goes inert
  (passes its own negative control), the self-test fails and *that* is the alarm. The check
  isn't trusted because I wrote it carefully — it's trusted because I watched it fail on the
  thing it's supposed to catch.
- **A cross-language golden-vector parity gate.** One difficulty-curve calculation exists in two
  implementations — a **`Lua`** server-side version and a **`TypeScript`** UI port — and they
  must never disagree. Both assert the *same* shared golden vector (**29 rows**, including the
  `.5`-rounding edge cases that are exactly where two independent implementations silently
  drift). If the server and the UI ever compute a different answer for the same input, the
  parity gate goes red. Two implementations of one truth, structurally pinned together.

## Building the layers out of order — the forward-compatible fold

The framework ships in **verified-green increments**, but a layered architecture has a
chicken-and-egg problem: a higher layer often needs a lower one that isn't designed yet. Both
obvious answers are traps. Stub the lower layer's API and freeze it early, and you've committed
to a contract before any real consumer has stress-tested its shape — a SemVer one-way door.
Block the higher layer until the lower one exists, and the work stalls.

The technique instead is a **forward-compatible fold**: when the lower mechanism doesn't exist
yet, **fold it inside the consumer** — hand-rolled, private — *behind the public surface the
consumer will keep*. The world/instancing subsystem needed a state-replication layer and an
instance allocator that weren't built, so it hand-rolled an internal allocator and wrote global
replicated state directly — **but only ever behind its own stable surface.** Nothing else in the
codebase saw the hand-rolled internals; everything saw the surface that would survive.

Later, when the real mechanisms were built as their own layer, the consumer's internals were
**un-folded** onto them — swapped to delegate — one subsystem at a time, never big-bang: the
world clock, then the instance allocator, then weather and the per-instance overrides. The part
that makes it safe:

**The proof that a swap is non-breaking is the regression suite staying byte-identically green
across it.** If the public surface truly didn't change, every existing assertion still passes,
unchanged — and *that is* the "surface unchanged" guarantee. No separate test is needed; the
unchanged suite is the contract. Across the entire un-fold — every raw global-state write in the
subsystem rerouted through the real replication layer — the world-simulation regression suite
stayed green, assertion for assertion, while the assertion count only ever grew. A regression
below that line fails the build.

The subtle bit: when you swap an implementation behind a frozen interface, you need an observable
that *distinguishes* the two, or your "it still works" test is testing nothing. Both the old raw
write and the new layered write left the same replicated value, so reading the value back proved
nothing. The distinguishing observable was the **coalesced-replication dirty-mark** the new path
sets and the raw path doesn't — that single difference is what the swap-proof asserts, and the
negative control reverts the swap and watches that assertion go red.

### The sharpest version of "mock-green is necessary, never sufficient"

The verification ladder above tells the eight-bugs-past-a-green-mock story. The fold work
produced an even sharper one.

One increment added a **server-authority guard** to the core state-write primitive — the gate
that drops a client trying to write a server-owned value. The fast mock suite went **fully
green.** But the guard's realm check compared a platform call against the literal boolean `true`,
and the real runtime returns a *truthy non-boolean* there. So on the real server, the guard
misread the server's **own** boot-time state write as a client spoof, refused it, and the
framework **failed to boot** — every downstream check failing because nothing came up.

The mock had stubbed that platform call to return a clean literal `true`, so the strictness bug
was invisible: 100% of the fast suite passed against a server that couldn't start. The real-boot
gate caught it; the fix was a one-line change to a truthy check — the idiom the rest of the
codebase already used. The lesson became a *standing forcing function*: a later,
security-critical guard now ships with a negative control whose entire job is to prove that an
over-strict version of it would drop a legitimate caller — the same failure class, now caught by
a test that bites.

## What generalizes

A portable recipe for verifying any system where the cheap tests can lie:

1. **Make the untrusted boundary a single choke point.** One hardened path every inbound message
   must traverse beats per-handler vigilance — adversarial input dies early, uniformly, and no
   handler can forget to guard itself.
2. **Derive truth from an append-only ledger; never overwrite it.** Auditable, reconstructible,
   exploit-resistant — and it makes "what was the balance at time T" a query, not a guess.
3. **Fail closed on everything authoritative.** When in doubt, deny. A fail-open path on state is
   a latent exploit waiting for the conditions that trip it.
4. **Build a verification ladder, and let the *real* tier certify.** Fast mocks for iteration;
   a real-dependency gate for truth; real concurrent traffic for load. Mock-green is necessary,
   never sufficient — and the proof is the eight bugs a 289/289 mock suite never saw.
5. **Negative-control every gate, then meta-test the controls.** A `--self-test` that proves
   each check goes red on its bad fixture is how you keep a green suite from quietly going inert.
6. **Pin cross-language duplicates to one shared golden vector** — including the rounding-edge
   rows — so two implementations of one rule cannot drift in silence.
7. **Fold an unbuilt lower layer inside its consumer, behind the surface you'll keep — then
   un-fold it non-breakingly.** Don't stub-and-freeze an API before a real tenant has shaped it;
   defer the freeze to the highest-confidence moment, and let the unchanged regression suite be
   the proof the swap changed nothing.
8. **When you swap behind a frozen interface, find the observable that distinguishes the two
   implementations** — or your "still works" test is green for the wrong reason.
9. **A clean stub can hide a real-boot bug.** Verify any change to a shared, boot-time primitive
   against the real runtime, and never compare a platform call against a literal `true`/`false` —
   mock-green can mean "passes against a server that can't start."

## Outcome

- A **ground-up, standalone** real-time multiplayer game-server framework with a **server-
  authoritative** core — every inbound event funneled through one hardened middleware choke
  point, every authoritative write fail-closed, economy state derived from an **append-only,
  partition-ready ledger** on real **MariaDB**, and right-to-erasure expressible as
  crypto-shred at the schema level.
- A **three-tier verification ladder** (mock-native → real-database boot-smoke → synthetic-player
  population) whose authoritative tier **exposed eight real bugs that a 289/289-green mock suite
  hid** — the project's strongest evidence that mock-green is necessary but never sufficient.
- **Biting quality gates**: an 11-check static gate, each with a negative-control fixture; a
  `--self-test` that proves every check goes red on its bad input; and a cross-language
  golden-vector parity gate (**29 rows**, rounding edges included) that pins a `Lua` server
  implementation and a `TypeScript` UI port to one shared truth.
- A **premium, gamepad-first React 19 + Vite 6 + Tailwind** in-game UI with rebindable keybinds
  and a focus-tracking interaction model, browser-verified to the same evidence standard as the
  server.
- A **forward-compatible fold** build discipline: lower layers folded inside their consumers
  behind a stable surface, then un-folded onto the real mechanisms one subsystem at a time — each
  swap proven non-breaking by a byte-identically-green regression suite, and one swap exposing a
  mock-green-yet-won't-boot bug that hardened into a standing forcing function.
- All of it anchored to a written **37-spec blueprint** and a **50-doctrine engineering
  constitution**, so the standard is enforceable rather than aspirational.

*Deliberately at the engineering-capability level; happy to walk through the choke-point design,
the verification ladder, and the biting gates on a call under NDA.*
