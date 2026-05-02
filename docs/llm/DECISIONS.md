# Decision Log

Durable architectural decisions for Tomatic.

Format:
- IDs: `D-001`, `D-002`, ...
- Each decision is self-contained.
- Facts and tradeoffs over narration.

The first nine decisions below are extracted from the source design document [`../reference/Tomatic_v3_2.docx`](../reference/Tomatic_v3_2.docx) chapter 20 ("Decisiones de diseño y trade-offs"). They are pre-recorded here so the very first implementation session can move directly into H0 without re-litigating them. Any later contradiction should be captured as a new ADR (D-010+) explaining the deviation, never as a silent edit to these entries.

---

## D-001 — Control determinista primero

A plant is a continuous biological system over 100 days. An LLM is an intermittent network call. Betting the plant's survival on the LLM being available and reasoning correctly every 30 minutes for 100 days is unnecessary risk. Tomatic inverts the priority: the deterministic control-core keeps the plant alive; the LLM contributes judgment quality (fine transitions, visual diagnosis, pattern observations).

**Consequence**: `packages/control-core` runs three basic loops (light, water, climate) without any LLM dependency. If you turn off the LLM, the simulator, and all front-ends, leaving only `control-core + mqtt-bridge + ESP32`, the plant survives.

## D-002 — LLM proposes intents, control-core decides

LLMs hallucinate, get prompt-injected, and occasionally produce nonsense. Turning their output into physical commands without a deterministic filter exposes hardware to model failure. With `propose_intent`, the worst case is a rejected intent with a logged reason; without it, the worst case is dispensing 1.5 L of water in one go.

**Consequence**: every MCP write-tool that affects the physical world is named `propose_*` and only inserts a row in the `intents` table. The control-core validates against safety rules (cooldowns, daily max, safe state, dependencies) before producing any command.

## D-003 — ESPHome único (no MicroPython)

The premise of the project is that Carlos's daughter does not touch firmware code. Under that constraint, MicroPython loses its only theoretical advantage (being "readable") and inherits all the pain: maintaining drivers, hand-rolling the I²C plumbing, the MQTT client, the LWT, the watchdog. ESPHome eliminates all of that. Native components for SHT3x, SCD4x, BH1750, ADS1115, DS18B20. OTA over WiFi after the first USB flash. Logs through a local web UI. The firmware becomes ~80–120 lines of declarative YAML that any LLM can generate and maintain.

**Consequence**: the firmware lives in a single file, `firmware/esphome/tomatic-esp32.yaml`. Custom C++ is allowed only when ESPHome cannot natively express what we need (e.g. the JSON-payload pump command parser may need a tiny custom component).

## D-004 — NAS-first deployment

The QNAP NAS has more RAM, more CPU, and robust storage compared to the Pi. SQLite with WAL and a long historical dashboard fit there better. The Pi 5 stays dedicated to Z2M (where the Zigbee radio already lives). Moving Z2M off the Pi would mean buying another coordinator. The NAS is always on and designed for 24/7 services. Snapshots and replication on the NAS are trivial.

**Consequence**: the production target for V1.0 is QNAP Container Station with `compose/docker-compose.prod-nas.yml`. The Pi 5 holds Z2M *and* the shared MQTT broker. The bridge runs a single MQTT client against `zigbee.home.arpa:1883` (D-010 / `home-infra` ADR-0011); both `tomatic/+` and `zigbee2mqtt/+/set` live there. Windows is dev only — no persistent services.

**Resolved**: D-010 (2026-05-02) supersedes the original "NAS Mosquitto + Pi Mosquitto cross-broker bridge" sketch from this consequence. The bridge now uses a single broker.

## D-005 — SQLite + WAL + drizzle + litestream

One writer (the bridge) and many readers (web, agent-runner, tests). WAL allows concurrent reads with writes. drizzle gives versioned migrations from the first day. litestream provides continuous backup. Migrating to Postgres later is trivial if the project grows.

**Consequence**: `PRAGMA journal_mode=WAL; synchronous=NORMAL; foreign_keys=ON` from H0. Migrations land at H0. litestream replicates `/var/lib/tomatic/tomatic.db` to `/share/Backups/tomatic/` on the NAS.

## D-006 — Cabinet prefix in every MQTT topic from V1.0

Multi-cabinet support arrives sooner than expected. Starting with `tomatic/<cabinet-id>/...` from day one avoids a painful migration later. The cost is one additional path segment.

**Consequence**: every topic in source doc §6 carries `tomatic/cabinet-a/...` even though only one cabinet exists today. Tests and code use `cabinet_id` as a first-class parameter everywhere.

## D-007 — Operator web in V1.0 (no required CLI)

Programming with an LLM reduces the cost of building a UI to almost nothing. A rich CLI for calibrating capacitive soil sensors (which are noisy by nature) is worse UX than a web page with live SSE. The architectural discipline a CLI would force is achieved instead through R10: every operation is a pure function in `core-ops`; the web is just a client.

**Consequence**: `apps/web` is V1.0-kernel scope (H9), with `/calibration`, `/safety`, `/logs`, `/metrics` and an *Override* button. CLI is not planned but, if it ever lands, it will be a thin client over `core-ops`.

## D-008 — CO₂ disabled by default until independent alarm is in place

CO₂ without a real sensor is controlling a variable we are not measuring. CO₂ without an independent alarm is a human-safety risk in a closed garage. The reference exposure limit is 5,000 ppm TWA over 8 h; a hobby project should not automate CO₂ without (a) a real sensor, (b) a local audible alarm independent from the software, and (c) a physical interlock.

**Consequence**: in V1.0, the SCD41 publishes CO₂ to the dashboard but the FZONE solenoid is `enabled: false` in the config. Activation depends on Carlos buying an independent CO₂ alarm (BOM Pendiente).

## D-009 — Vercel AI SDK from V1.1 for multi-provider agnosticism

Building the agent-runner against a multi-provider client from day one preserves the agnosticism promised by MCP. The extra cost is low: the Vercel AI SDK API is among the cleanest in the ecosystem. Before V1.1, `agent-stub` validates the path without an LLM, so the SDK is not on the critical path of V1.0.

**Consequence**: provider switch is config-driven (`agent.provider: anthropic|openai|ollama`). The agent-runner reads model + provider from `/etc/tomatic/config.yaml` and instantiates the right client. Switching providers is "change three lines and restart".

---

## D-010 — Reuse the shared Mosquitto on the `zigbee` RPi instead of running a Tomatic-dedicated broker

- **Date:** 2026-05-02
- **Status:** accepted
- **Upstream reference:** `home-infra` ADR-0011 (canonical homelab decision; this entry mirrors the project-side consequences).

### Context
The source design document (`reference/Tomatic_v3_2.docx`, ch. 13 "Despliegue dockerizado") proposes a Tomatic-dedicated Mosquitto on the NAS, with the bridge holding two MQTT clients (NAS for `tomatic/+`, the existing `zigbee` RPi broker for `zigbee2mqtt/+/set`). Live verification on 2026-05-02 showed that the NAS no longer runs an MQTT broker — the previous container was stopped on 2026-02-21 and later removed (absent from `docker ps -a`). Only an orphan compose dir remains at `/share/Container/compose/mqtt/`. The only live broker on the homelab is the Mosquitto on the `zigbee` RPi (`zigbee.home.arpa:1883`, IP `10.0.0.139:1883`), already used by Z2M and Home Assistant.

This was Open Question 1 in `HANDOFF.md`. Carlos resolved it on 2026-05-02 in favor of reusing the shared broker.

### Decision
Tomatic uses the existing Mosquitto on the `zigbee` RPi as its only broker. No NAS broker is created or revived. The decision is recorded as the canonical homelab fact in `~/src/home-infra/docs/DECISIONS.md` ADR-0011; this D-010 documents the project-side consequences inside the Tomatic repo.

### Consequences
- `packages/mqtt-bridge` (lands at H1) opens a single MQTT client against `zigbee.home.arpa:1883` and uses it for both `tomatic/<cabinet>/+` topics and `zigbee2mqtt/+/set` commands to Sonoff T2 actuators. The "two MQTT clients in the bridge" sketch from the source design document is dropped.
- The `infra.contract.yml` at the repo root declares `mqtt.broker_service_id: mosquitto` and lists the ACL topic shapes the Tomatic clients need (publish: `tomatic/+/commands/+/request`, `zigbee2mqtt/+/set`, `tomatic/+/agent/+`; subscribe: `tomatic/+/sensors/#`, `tomatic/+/commands/+/ack`, `tomatic/+/alerts/#`, `zigbee2mqtt/+`). The dedicated Mosquitto user `tomatic_bridge` is provisioned and the ACL is applied on the existing broker at H1, not in this repo.
- `home-infra/catalog/services.yml` declares `mosquitto` as a first-class shared service (host `zigbee`, category `infra`, status TCP probe on `:1883`) and adds `deps: [mosquitto]` to `zigbee2mqtt`, `ha`, and the planned `tomatic-web` / `tomatic-public-dashboard`. `infra-portal` will render the dependency graph.
- A `zigbee` RPi reboot interrupts both Z2M and the Tomatic bus simultaneously. R5 (default-off electrical), R8 (idempotency by `command_id` + TTL in `mqtt-bridge` SQLite), and the autonomous local safe-state in the ESPHome firmware (source doc ch. 4) make this acceptable: the plant survives the broker being unreachable.
- R3 (`retain=false` on commands) and the topic catalog in source doc ch. 6 remain enforced unchanged on the shared broker.
- Open Question 1 in `HANDOFF.md` is resolved and removed from the open list.

---

## D-011+ (future)

Add new ADRs here when the implementation forces a deviation from the source design document or when an open question is resolved (e.g. public dashboard host, Doppler project layout).
