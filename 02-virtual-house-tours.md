# Virtual House Tours — full SaaS product

*A complete property-tour platform: capture on mobile, view anywhere, sell and bill.*

> **Status:** in active development. Built front to back — mobile, web, desktop, API, billing, infra.

## Context

Property tours are a high-value, visual, mobile-first problem: the content is captured on
a phone in the field, consumed by buyers on the web, and managed by agents and teams.

## Problem

Ship a real SaaS — not a demo — spanning a **native capture app**, a polished **web
viewer + dashboard**, **desktop tooling**, a hardened **API**, billing, and the operational
backbone (storage, queues, observability) to run it reliably.

## Constraints

- Multi-surface: a single coherent product across iOS, web, and desktop.
- Media-heavy: large assets captured in the field need fast, secure delivery.
- A real business: subscriptions, tenants, and uptime — not a side project.

## Approach

- **Native iOS** capture app for the field.
- **Web viewer + dashboard** for buyers and operators.
- **Tauri desktop** tooling for power workflows.
- A **Fastify** API with **signed-URL** media delivery, background workers for processing,
  and **Stripe** for subscriptions.
- A full local-parity stack (Postgres, Redis, object storage, metrics) so "works on my
  machine" means "works in prod."

## Architecture

- **API:** `Fastify` (`TypeScript`) — REST + signed-URL issuance + tenant model.
- **Apps:** native `iOS`; `Vite`/React web (marketing + dashboard); `Tauri` desktop.
- **Data & infra:** `PostgreSQL`, `Redis` (queues + rate limiting), S3-compatible object
  storage, background workers, `Prometheus`/`Grafana` observability, `Stripe` billing.
- **Delivery:** SSR public/embed pages for shareable tours.

## Outcome

- A multi-surface product — **iOS + web + desktop + API** — built front to back, with billing and observability already in place.
- Currently in active development toward launch.

*Walkthrough available on request.*