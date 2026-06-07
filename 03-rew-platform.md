# Real-estate wholesaling platform

*A multi-service software stack for real-estate wholesaling: web, seller outreach / telephony, AI lead handling, and durable deal workflows.*

> **Status:** private / in development. Architecture generalized; proprietary specifics removed.

## Context

Real-estate wholesaling runs on high-volume seller outreach, fast lead qualification, and
deal pipelines that can't drop state — which spans very different runtime needs at once: a
web product, real-time telephony, an AI layer, and long-running business workflows.

## Problem

Build a platform where the web app, the telephony service, the AI layer, and durable deal
workflows cooperate — without the whole thing collapsing into one brittle monolith or an
un-debuggable mesh.

## Constraints

- Mixed workloads: low-latency telephony, bursty AI calls, and long-running workflows.
- Operations that must survive restarts and partial failure (no lost calls, no dropped
  state mid-deal).
- Analytics over a high volume of events without bogging down the transactional path.

## Approach

- Pick the right runtime per job: **Go** for the latency-sensitive telephony service,
  **Python/FastAPI** for the AI layer, **Next.js** for the web surface.
- Make long-running deal logic **durable** with a workflow engine instead of ad-hoc
  cron + retries.
- Separate the **analytical** path from the **transactional** one so reporting never
  contends with live traffic.

## Architecture

- **Web:** `Next.js`.
- **Telephony:** a `Go` service for the real-time seller-outreach path.
- **AI:** `FastAPI` lead-handling / orchestration layer.
- **Workflows:** `Temporal` for durable, resumable deal processes.
- **Streaming & analytics:** `Kafka`/Redpanda for the event backbone; `ClickHouse` for
  fast analytical queries; `OpenSearch` for search.
- **Supporting:** object storage, local mail capture for dev parity.

## Outcome

- A wholesaling platform where each concern runs on the right tool and failures are contained.
- Durable deal workflows that resume correctly across deploys and outages.
- Real-estate domain logic built on production-grade distributed-systems patterns, end to end.

*Deeper architecture discussion available on request.*
