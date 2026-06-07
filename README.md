# Case studies

Most of my highest-leverage work is proprietary or under NDA and lives in private
repositories. Rather than leave it invisible, I publish **architecture-level write-ups**:
the problem, the constraints, the approach, the system design, and the outcome — with
client-identifying details and proprietary specifics removed.

> If you want to go deeper than these summaries, I'm happy to walk through decisions and
> code on a call under NDA.

| # | Case study | Domain | Stack highlights |
|---|---|---|---|
| 01 | [Operational backend for a real estate investment firm](./01-real-estate-firm-backend.md) | AI automation · CRM · internal platforms | `Python` `FastAPI` `LLM orchestration` `Postgres` |
| 02 | [Virtual House Tours — full SaaS product](./02-virtual-house-tours.md) | Proptech SaaS · mobile + web + desktop | `TypeScript` `iOS` `Fastify` `Stripe` |
| 03 | [Real-estate wholesaling platform](./03-rew-platform.md) | Proptech · telephony · AI · workflows | `Next.js` `Go` `Temporal` `Kafka` `ClickHouse` |
| 04 | [Real-time hardware control plane](./04-realtime-control-plane.md) | Embedded-adjacent · live telemetry | `TypeScript` `React` `FastAPI` `HAL design` |
| 05 | [Custom Android platform & instrumentation R&D](./05-systems-and-instrumentation.md) | Low-level systems · emulation · reverse engineering | `C` `Kotlin` `Frida` `Vulkan` `Metal` |

---

**How to read these:** each follows the same shape — **Context → Problem → Constraints →
Approach → Architecture → Outcome**. Numbers marked `[…]` are intentionally generalized.