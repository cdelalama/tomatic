# Decision Log

Durable architectural decisions for Tomatic.

Format:
- IDs: `D-001`, `D-002`, ...
- Each decision is self-contained.
- Facts and tradeoffs over narration.

The first nine decisions below are extracted from the source design document `~/src/Tomatic_v3_2.docx` chapter 20 ("Decisiones de diseño y trade-offs"). They are pre-recorded here so the very first implementation session can move directly into H0 without re-litigating them. Any later contradiction should be captured as a new ADR (D-010+) explaining the deviation, never as a silent edit to these entries.

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

**Consequence**: the production target for V1.0 is QNAP Container Station with `compose/docker-compose.prod-nas.yml`. The Pi 5 becomes a peer that holds Z2M; the bridge crosses brokers (NAS Mosquitto for Tomatic topics, Pi Mosquitto for `zigbee2mqtt/+/set`). Windows is dev only — no persistent services.

**Open question**: whether to run a Tomatic-dedicated Mosquitto on the NAS or reuse the existing one on `zigbee`. Default for now: dedicated. Revisit before H1.

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

## D-010+ (future)

Add new ADRs here when the implementation forces a deviation from the source design document or when an open question is resolved (e.g. Mosquitto topology, public dashboard host, Doppler project layout).
