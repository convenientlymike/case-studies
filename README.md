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
| 06 | [Root-causing a GPU-driver crash (Metal↔Vulkan)](./06-gpu-driver-crash-re.md) | GPU driver RE · crash debugging · Apple Silicon | `C/C++` `Vulkan` `Metal` `Mesa` `gfxstream` |
| 07 | [Making "done" machine-checkable](./07-forcing-function-completeness.md) | Engineering methodology · quality gates · large apps | `Python` `FastAPI` `TypeScript` `React` `lint gates` |
| 08 | [Root-causing a "degrades-after-boot" reliability bug](./08-degrades-after-boot-debug.md) | Low-level debugging · Android platform · daemon lifecycle | `Android internals` `SELinux` `zygote` `multi-agent RE` |
| 09 | [Brick-safe firmware writes](./09-brick-safe-firmware.md) | Embedded safety · firmware lifecycle · open-source | `Python` `FastAPI` `React` `safe-by-construction` |
| 10 | [Building a commercial real-time multiplayer game-server framework — and a verification ladder that caught what the mocks couldn't](./10-multiplayer-game-framework-verification.md) | Real-time multiplayer · server-authoritative · verification ladder | `Lua` `TypeScript` `React` `MariaDB` `Docker` |
| 11 | [Capturing a game's runtime-resolved asset CDN (binary RE → passive memory pointer-chase)](./11-unity-asset-cdn-re.md) | Game asset RE · Unity/IL2CPP · first-party datamining | `Python` `IL2CPP` `Unity Addressables` `UnityPy` `ELF/ARM64` |
| 12 | [A verified-reputation network for the skilled trades](./12-steeltoe-reputation-network.md) | Trades hiring · reputation graph · marketplace design · compliance-first | `Next.js` `TypeScript` `Prisma` `Postgres` |

---

**How to read these:** each follows the same shape — **Context → Problem → Constraints →
Approach → Architecture → Outcome**. Numbers marked `[…]` are intentionally generalized.