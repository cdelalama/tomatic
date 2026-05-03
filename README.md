<!-- doc-version: 0.1.5 -->
# Tomatic

**Agentic indoor tomato grow system — deterministic control-core, LLM-proposed intents, safe by default.**

**Version:** see [VERSION](VERSION) | [CHANGELOG](CHANGELOG.md)

## Overview

Tomatic is an autonomous indoor cultivation system designed to keep 2–3 tomato plants alive in a 120×120×200 cm tent for 100 days without ordinary manual intervention. It is built as a father-and-daughter educational IoT project, but engineered with production-grade safety so the plants survive even if the LLM, the API, or the network goes away.

The core idea separates judgment from supervival. A deterministic TypeScript **control-core** owns physical control: it runs light/water/climate loops without any LLM, validates safety rules (cooldowns, daily limits, default-off on failure), and is the only component allowed to emit physical commands. An optional **LLM agent** observes the cabinet through an MCP server and proposes *intents* expressed in cultivation terms ("increase humidity", "the plant looks dry"); the control-core decides whether and how to act on them. The LLM never touches actuators directly.

Hardware is intentionally simple: an ESP32 running a single ESPHome YAML, sensors on I²C and 1-Wire, a peristaltic pump with transactional ACK, and Sonoff T2 actuators driven through the existing Zigbee2MQTT mesh. The whole stack runs in Docker on a QNAP NAS with a Raspberry Pi 5 hosting Z2M as today.

## Status

Scaffolded 2026-05-02. Source design document: [docs/reference/Tomatic_v3_2.docx](docs/reference/Tomatic_v3_2.docx) (Carlos, May 2026). No application code yet — see [docs/llm/HANDOFF.md](docs/llm/HANDOFF.md) for the current milestone (H0 — schemas + DB).

## Quick Start

The simulator (`pnpm dev:sim`) is the planned zero-hardware entry point: it brings up Mosquitto, the bridge, the control-core, a fake ESP32, a fake plant, and the operator web on `http://localhost:3001`. Until V1.0-kernel lands, this Quick Start is aspirational.

```bash
# Planned (V1.0-kernel — H6):
pnpm install
pnpm dev:sim
# open http://localhost:3001
```

## Architecture in one paragraph

Eight layers, top to bottom: **public dashboard** (V1.2+) → **operator web** (Next.js, calibration + safety + override) → **agent-runner** (Vercel AI SDK + cron + systemd) → **plant MCP server** (`propose_*` tools only) → **control-core** (deterministic rules, planner, loops without LLM — *the plant survives here*) → **core-ops** (the only door to SQLite + bridge from web/cli/tests) → **mqtt-bridge** (ingest + command queue + state machine, SQLite WAL as source of truth) → **hardware** (ESP32 + Sonoff T2 + Pi + Logitech, ESPHome firmware with autonomous local safe-state). Read [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for the full picture.

## Documentation

| Document | Purpose |
|----------|---------|
| [LLM_START_HERE.md](LLM_START_HERE.md) | Entry point for LLM contributors |
| [AGENTS.md](AGENTS.md) | Homelab integration rules (read-first for any agent) |
| [docs/PROJECT_CONTEXT.md](docs/PROJECT_CONTEXT.md) | Vision, objectives, stakeholders, current state |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | Layers, hard rules R1–R12, key flows, contracts |
| [docs/STRUCTURE.md](docs/STRUCTURE.md) | Planned monorepo layout |
| [docs/reference/Tomatic_v3_2.docx](docs/reference/Tomatic_v3_2.docx) | Canonical source design document |
| [docs/VERSIONING_RULES.md](docs/VERSIONING_RULES.md) | Version management policy |
| [docs/llm/HANDOFF.md](docs/llm/HANDOFF.md) | Current work state |
| [docs/llm/DECISIONS.md](docs/llm/DECISIONS.md) | Architectural decision log |

## Hardware (May 2026)

Already at Carlos's place: Spider Farmer 120×120×200 tent, SF4000 LED 450W, 6" extraction kit, G628 peristaltic pump, FZONE 2.5L CO₂ + solenoid (disabled in V1.0), Tayg 20 L tank, IP54 enclosure. ESP32 DevKit V1, FS400-SHT35, Sensirion SCD41, BH1750/GY-302, ADS1115 + 3× capacitive soil v1.2, 2× DS18B20, Aqara water leak. 6× Sonoff T2 already paired with Z2M. Raspberry Pi 5 already running Z2M; QNAP NAS already running Docker; Logitech 4K USB available.

Pending blocker for V1.0 plants: float switch for the 20 L tank.

## License

Released under the MIT License. See [LICENSE](LICENSE).

---

*Documentation scaffold powered by [LLM-DocKit](https://github.com/cdelalama/LLM-DocKit). Homelab integration via [home-infra-protocol](https://github.com/cdelalama/home-infra-protocol).*
