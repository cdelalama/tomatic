<!-- doc-version: 0.1.6 -->
# Project Context — Tomatic

## Vision

Tomatic is an autonomous indoor tomato grow system that keeps 2–3 plants alive for 100 days inside a 120×120×200 cm tent without ordinary manual intervention, designed as a father-and-daughter educational project that does not compromise on engineering rigor.

The system is built around a single conceptual inversion: **the LLM proposes, the deterministic core decides**. A TypeScript control-core owns safety, validates intents, plans physical commands, and runs light/water/climate loops without any LLM. The LLM agent supplies judgment quality (fine transitions, visual diagnosis, pattern observations) through an MCP server whose write-tools only register intents — they never emit physical commands. The plant survives even when the LLM, the API, or the network is unavailable.

The longer-term goal is to publish Tomatic as a reproducible open-source reference: a stack any homelab operator can clone, run end-to-end in simulation in five minutes, and adapt to a different cultivar by swapping a YAML preset.

## Objectives

- **V1.0-kernel**: deterministic control + idempotent commands + simulator + operator web + ESPHome firmware on real hardware. Acceptance: 100 simulated days without killing the plant; pump acceptance test passes against real hardware.
- **V1.1-agentic**: real LLM in the loop via MCP + Vercel AI SDK + systemd, with provider-switch (Anthropic/OpenAI/Ollama) behind one config line.
- **V1.2-public**: storytelling-grade public dashboard for Carlos's daughter and visitors; live camera, timelapse, narrative timeline.
- **Educational success**: Carlos's daughter completes the five "modos hija" modules (the plant talks, the plant is thirsty, the robot decides, plant journal, the AI is wrong) without ever touching 220 V.

Success criteria across all phases:

- Plants survive 100 days unattended in soak simulation with the control-core only (no LLM in the path).
- A duplicated `command_id` never produces a second physical effect. `acceptance-test-pump.sh` passes with `count=1`.
- No retained MQTT message ever lives on a `commands/` topic. `acceptance-test-retain.sh` passes with `count=0`.
- All hazardous actuators (pump, CO₂, humidifier, heat mat) default to OFF on every failure mode.

## Stakeholders

- **Product owner / technical owner / sole operator**: Carlos de la Lama-Noriega.
- **Primary educational user**: Carlos's daughter (within the "plano hija" — 3.3 V, sensors, dashboard, calibration; never 220 V).
- **LLM agents**: Claude Code (cockpit + development), agent-runner LLM (Anthropic/OpenAI/Ollama at Carlos's choice).
- **Future users**: homelab operators who want a reproducible LLM-assisted grow system on commodity hardware.

## Architectural Overview

Eight layers, runtime data flow top → bottom:

```text
CAPA 8  PUBLIC DASHBOARD     Next.js, Recharts, SSE             (V1.2+)
CAPA 7  OPERATOR WEB         Next.js, NextAuth, shadcn/ui       (V1.0+)
CAPA 6  AGENT-RUNNER         Node.js, Vercel AI SDK, systemd    (V1.1+)
CAPA 5  PLANT MCP SERVER     @modelcontextprotocol/sdk + Zod    (V1.1+)
CAPA 4  CONTROL-CORE         deterministic rules, planner, loops  ← the plant survives here, with no LLM
CAPA 3  CORE-OPS             pure TypeScript: the only door to bridge + SQLite
CAPA 2  MQTT-BRIDGE          ingest + command queue + state machine + idempotency
CAPA 1  HARDWARE             ESP32 (ESPHome) + Sonoff T2 (Z2M) + Pi + Logitech 4K
```

In parallel: simulator (`fake-esp32` + `fake-plant`) for CI and zero-hardware development; Claude Code as cockpit for inspection, prompt experiments, education sessions; observability endpoints (`/healthz` `/readyz` `/metrics`) on every service from the first commit.

The full layered diagram, hard rules R1–R12, and key flows are in [`ARCHITECTURE.md`](ARCHITECTURE.md).

## Key Components

| Component | Purpose | Stage | Notes |
|-----------|---------|-------|-------|
| `packages/control-core` | Deterministic rules, planner, light/water/climate loops without LLM | V1.0 | The plant must survive here. Pure TS, vitest, no network. |
| `packages/mqtt-bridge` | MQTT ingest + command queue + ACK/state-verification + idempotency by `command_id` | V1.0 | SQLite WAL as source of truth. R8 enforced here. |
| `packages/core-ops` | Pure TS API; the only doorway from web/agent/tests to bridge + SQLite | V1.0 | R10: zero `mqtt.connect` or `better-sqlite3` outside this package. |
| `packages/db` | drizzle schema + migrations + WAL config | V1.0 | Migrations from H0; never edit schema by hand. |
| `packages/shared` | Zod schemas for `Intent`, `Command`, `AckTransactional`, `StateVerificationResult`, `SafetyState`, `OverrideInput` | V1.0 | Shared between bridge, control-core, core-ops, web. |
| `apps/agent-stub` | YAML loader injecting fixed intents to validate the path end-to-end without an LLM | V1.0 | Replaced by agent-runner in V1.1. |
| `apps/web` | Operator web (calibration, safety panel, override, logs) and public dashboard | V1.0 / V1.2 | Two Next.js apps sharing components. |
| `packages/plant-mcp-server` | MCP server with tools `propose_*` (write) and reads (`get_*`, `read_journal`) | V1.1 | Write-tools register intents, never emit commands. |
| `apps/agent-runner` | Vercel AI SDK + cron + lock-per-cabinet + catch-up after downtime | V1.1 | systemd unit, multi-provider config-driven. |
| `apps/simulator` | `fake-esp32` + `fake-plant` modeling soil moisture, VPD, light, sensor failures | V1.0 | CI runs 100 simulated days at 100× speed. |
| `firmware/esphome/tomatic-esp32.yaml` | Single declarative YAML for the ESP32 firmware | V1.0 | Sensors + transactional pump ACK + autonomous local safe-state (R5). |

## Current Status (2026-05-02)

- Repository scaffolded from LLM-DocKit 4.6.1 with the homelab profile applied.
- Public GitHub repo: <https://github.com/cdelalama/tomatic> (MIT).
- No application code yet. The next step is **H0 — schemas + DB**: Zod types in `packages/shared`, drizzle schema in `packages/db`, SQLite WAL configured, vitest green.
- Hardware available at Carlos's home (see README and [`ARCHITECTURE.md`](ARCHITECTURE.md) §Hardware). Pending purchase before real plants enter the cabinet: float switch for the 20 L tank.
- Source design document: [`reference/Tomatic_v3_2.docx`](reference/Tomatic_v3_2.docx) (Carlos, May 2026). The hard rules in chapter 4 are axiomatic — any contradicting instruction must pause and ask.

## Upcoming Milestones

V1.0-kernel runs in chained hits H0 → H11 (each with its acceptance test):

1. **H0 — Schemas + DB**. Zod + drizzle + SQLite WAL. Migrate/rollback works; vitest green.
2. **H1 — mqtt-bridge**. Ingest sensors → SQLite + command queue + state machine.
3. **H2 — Idempotency**. `command_id` + TTL + dedup. `acceptance-test-pump.sh` passes (count=1).
4. **H3 — Retain policy**. `acceptance-test-retain.sh` passes (count=0 retained on `commands/+`).
5. **H4 — control-core**. Rules + planner + 3 loops without LLM. 100 simulated days without killing the plant.
6. **H5 — core-ops**. Pure API; grep proves no `better-sqlite3` or `mqtt.connect` outside.
7. **H6 — Simulator**. `pnpm dev:sim` brings the full stack up; six scenarios pass.
8. **H7 — Observability**. `/healthz` `/readyz` `/metrics` on bridge, control-core, web.
9. **H8 — agent-stub**. End-to-end intent → command → ACK → SQLite path validated without LLM.
10. **H9 — Operator web**. Calibration wizard, safety panel, override with TTL, logs, metrics.
11. **H10 — ESP32 firmware**. ESPHome YAML on real hardware; pump acceptance against real pump.
12. **H11 — V1.0 close-out**. 100-day soak with real firmware connected; safety checklist signed.

V1.1-agentic (H12–H16) and V1.2-public (H17–H20) follow once V1.0 is closed. Detailed acceptance criteria per hit in the source design document, chapter 22.

## References

- **Source design document**: [`reference/Tomatic_v3_2.docx`](reference/Tomatic_v3_2.docx) (canonical; chapter 4 = hard rules, chapter 22 = roadmap, chapter 23 = step-by-step execution).
- **Homelab source of truth**: `~/src/home-infra/docs/` (INVENTORY, SERVICES, PROJECTS, CONVENTIONS).
- **Homelab protocol**: <https://github.com/cdelalama/home-infra-protocol>.
- **Documentation scaffold**: <https://github.com/cdelalama/LLM-DocKit> (v4.6.1).
- **External upstreams**: ESPHome, Mosquitto, Zigbee2MQTT (running on `zigbee.home.arpa` 10.0.0.139), `@modelcontextprotocol/sdk`, Vercel AI SDK, drizzle-orm, litestream.
