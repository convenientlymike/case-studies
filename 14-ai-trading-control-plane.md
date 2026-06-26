# An AI trading control plane with a 17-rail safety engine

*Letting an LLM research and place trades on a real brokerage account — without it being able to do
anything catastrophic, and without pretending it has an edge it hasn't proven.*

> **Status:** built (private — it touches a live brokerage account). A pnpm monorepo: a tested
> safety core, a SQLite "brain", a Fastify API, a React 19 dashboard, and a Discord bot.

## Context

A broker shipped a first-class *agentic trading* API (reachable from an LLM over MCP) and a dedicated,
permission-scoped account for it. Reading account state is safe. **Placing trades is not** — an order is
irreversible, and an LLM is, by default, a *poor* autonomous trader: training-cutoff knowledge, no
real-time market model, hallucinated tickers, and a tendency to chase narratives. The most rigorous
public benchmark finds most LLMs can't beat buy-and-hold; the base rate is brutal.

## Problem

Build a system where an AI can research and *propose* trades, but every catastrophic outcome —
ruin from negative-edge churn, a 90-day cash-account settlement lockout, a runaway double-order, a
hallucinated ticker filled as a real order — is **impossible by construction**. And refuse to dress
the agent up as a money machine it hasn't earned the right to be: the intellectual honesty is a
deliverable, not a footnote.

## Constraints

- **Irreversible, metered actions.** A placed order can't be un-placed; frequent trading is negative-sum
  after fees and taxes; a cash account that sells and re-buys against unsettled funds trips
  good-faith-violation lockouts.
- **LLM failure surface.** Hallucinated symbols (~1-in-10), stale prices, prompt-driven inconsistency,
  no manager between sessions.
- **An unproven channel.** The agent's broker API is only reliably reachable in an *interactive* session;
  whether a headless/scheduled run can reach it is unverified — so the design must not depend on it.
- **No proven edge.** The honest default has to be *paper-first*, because the risk-of-ruin math says small
  real-money sizing only postpones ruin if the edge is ≤ 0.

## Approach

- **The rails dispose; the LLM only proposes.** Every order passes a single engine of **17 deterministic
  rails** the model cannot edit or argue past — settled-cash, position/sector concentration, a drawdown
  circuit-breaker, a hallucinated-ticker sanitizer, wash-sale/DRIP, limit-only entries, a cost-EV gate,
  and a hard kill-switch. A blocked proposal is structurally un-approvable.
- **Every rail paired with a forcing function.** 26 tests prove each rail *bites* — and a deliberately
  broken rail turns the suite red (a passing gate is unproven until a negative control makes it fail).
- **Human-in-the-loop, by construction.** Approval is the *only* write path; nothing auto-executes.
  Approve/reject happens in a dashboard or from a phone (a Discord bot with buttons, operator-locked).
- **Paper-first as a code path, not a promise.** A `mode` flag *physically routes around* the real
  order call until a forward, cost-inclusive paper run proves a positive net-of-cost edge versus the
  benchmark. Until then, real orders are impossible.
- **Two-leg architecture around the unproven channel.** A read-only *research leg* (harmless if it
  silently fails) is separated from an interactive, human-gated *execution leg* — so the irreversible
  half never rides the unreliable path.
- **Honest by construction.** Returns are reported as a contribution-stripped *time-weighted return*
  versus a benchmark — never raw value growth that would count deposits as gains; paper fills are labeled
  *assumed at quote (real fills worse)*; win rates show *n=* because small samples are noise.

## Architecture

- **Safety core (`@rh/core`):** the 17 rails, a conservative paper-fill engine, a barbell strategy, and a
  cost model — pure, dependency-light, fully unit-tested with negative controls. (`TypeScript`)
- **The brain (`@rh/db`):** the broker API is a *sensor*; a local SQLite store is the *brain*. It owns
  everything analytical (FIFO tax lots, settled-cash, returns) plus an **append-only, hash-chained
  decision log** that's verified on every boot. (`node:sqlite`, zero native deps)
- **Control plane (`@rh/api` + `@rh/web`):** a `Fastify` backend (rails-on-every-proposal, local paper
  execution, a housekeeping cron, a notification spine) and a `React 19` dashboard — ten dark-fintech
  panels including the human approval queue.
- **Reach (`@rh/bot`):** a `discord.js` bot for out-of-band alerts and approve/reject from a phone —
  operator-locked, and mock-first (disabled until a token is set).

## Outcome

- The full loop — ingest live market state → score candidates → propose → rails evaluate → human approves
  → fill booked with correct tax-lot accounting — is **operable today in paper mode against real market
  data**, verified end-to-end through the actual UI.
- Every safety invariant is backed by a negative-control test; the blast radius is bounded to a swept
  weekly budget; the irreversible path is human-gated and paper-first.
- The honest conclusion is built into the product: **the boring automated index DCA does the real
  compounding; the agent is a capped, mostly-paper experiment that has to earn real money by proving a
  forward edge.** A confident proposal never substitutes for a verified one.

*Not investment advice. Numbers and specifics are intentionally generalized.*
