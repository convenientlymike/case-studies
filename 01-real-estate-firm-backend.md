# Operational backend for a real estate investment firm

*AI automation · CRM · custom internal platforms · training. Owned end to end.*

> **Status:** production · proprietary. Architecture generalized; client details removed.

## Context

A real estate investment firm runs on a high volume of inbound leads, deals in flight,
follow-ups, documents, and team coordination. Off-the-shelf SaaS covered fragments of
this but left the seams — and the most valuable logic — as manual work spread across
tools that didn't talk to each other.

## Problem

Build and **own the entire backend** that runs the business: capture and qualify leads,
move deals through a pipeline, automate the repetitive communication and data entry,
keep every system in sync, and train the team — without a patchwork of disconnected SaaS.

## Constraints

- One engineer owning the full surface — so the architecture had to favor leverage and
  maintainability over headcount.
- AI in the loop where it earns its place (qualification, drafting, extraction,
  summarization), with deterministic guardrails — not "AI for its own sake."
- Real money and real client relationships flow through it: correctness and auditability
  are non-negotiable.

## Approach

- **AI automation pipelines** that handle the repetitive, judgment-light work end to end,
  with human checkpoints where stakes are high.
- A **CRM and deal-flow system** modeled around how the firm actually operates, not a
  generic template.
- **Custom internal platforms** that replace the spreadsheet-and-glue layer with owned
  software.
- A **courses / training platform** so the team scales on consistent process.
- Integrations that make the previously-disconnected tools behave as one system.

## Architecture

- **Services:** `Python` + `FastAPI` for the core APIs and automation workers; `TypeScript`
  / `Node` for product-facing surfaces.
- **AI layer:** LLM orchestration for agents and pipelines (qualification, extraction,
  drafting, summarization) with structured outputs and validation at the boundary.
- **Data:** `PostgreSQL` as the system of record; queues/workers for async automation.
- **Frontend:** `React` for internal tools and dashboards.
- **Cross-cutting:** auth, audit logging, and observability so automated actions are
  always explainable.

## Outcome

- **200+ automations** running in production — built in **under a year**.
- The equivalent of **several full-time people's** worth of manual work, automated away.
- The firm's full lead/deal pipeline and CRM now run on owned systems instead of disconnected spreadsheets and SaaS *(exact volumes under NDA)*.
- A backend the firm owns outright — extendable in days, not vendor-renewal cycles.