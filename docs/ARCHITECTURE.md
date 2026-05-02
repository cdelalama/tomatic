<!-- doc-version: 0.1.3 -->
# Tomatic Architecture

> Version: 0.1.0-draft
> Last Updated: 2026-05-02
> Status: Design (no code yet)
> Source of truth: [`reference/Tomatic_v3_2.docx`](reference/Tomatic_v3_2.docx) (Carlos, May 2026). When this file and the source document disagree, the source document wins.

## Overview

Tomatic is a multi-layer agentic IoT system. A deterministic TypeScript control-core owns physical control and runs three basic loops (light, water, climate) without any LLM. An optional LLM agent observes the cabinet through an MCP server and proposes intents in cultivation terms; the control-core validates, plans physical commands, and emits them with `command_id` + TTL idempotency. The plant survives without the LLM.

- **Inputs**: ESP32 sensor publishes (SHT35, SCD41, BH1750, ADS1115 + 3× capacitive soil, 2× DS18B20), float switch GPIO, Aqara water leak via Z2M, optional camera frames.
- **Outputs**: peristaltic pump dispense commands (transactional ACK from ESPHome), Z2M `set` commands for 6× Sonoff T2 (light, extractor, intractor, heat mat, humidifier, CO₂ solenoid — last one disabled in V1.0), MQTT-published metrics, operator web UI on `:3001`, public dashboard on `:3000` (V1.2+).
- **Where it runs**: Docker on QNAP NAS as the production target. Z2M stays on the existing `zigbee` RPi (`10.0.0.139`). Windows + WSL2 for development.

## Non-negotiables (Hard rules R1–R12)

These are axioms from chapter 4 of the source design document. Any instruction that contradicts them must pause and ask.

- **R1 — The LLM never emits physical commands.** Every MCP tool that touches the physical world is `propose_*` and only registers an intent. The only component allowed to publish to a `commands/` topic is the control-core via mqtt-bridge.
- **R2 — Idempotency on every physical command.** Every command published to `tomatic/<cabinet>/commands/+/request` carries a unique `command_id`, `issued_at`, `ttl_ms`. The receiver dedups before acting. Re-emitting the same `command_id` produces zero second physical effect. `acceptance-test-pump.sh` must pass before any plant enters the cabinet.
- **R3 — `retain=false` on commands.** A retained command is poison: a reconnecting client could re-execute an old command. `commands/+/request`, `commands/+/ack`, `alerts/+` are all `retain=false`.
- **R4 — Transactional ACK vs state verification.** ESPHome firmware (the pump) emits a transactional ACK with `command_id`, `status`, `ts`. Z2M does not emit comparable ACKs; for Sonoff T2 the bridge does *state verification* (publish `desired_state`, observe `state` topic within timeout). Names in code, docs, and metrics: `ack_received`, `state_verified`, `state_timeout`. Never call a Z2M result an "ACK".
- **R5 — Default-off electrical.** On any failure mode (ESP32 reboot, watchdog reset, firmware exception, WiFi loss, Mosquitto down, Pi reboot, TTL expiry, safe state entry) hazardous actuators land OFF: pump, CO₂, humidifier, heat mat. Extractor: ON only if the ESP32 controls it locally with a direct cable; if it depends on Z2M/Sonoff, it is OFF and documented as `off_safe_documented`.
- **R6 — No sensor, no actuator.** No hazardous actuator is automated unless a sensor can verify its effect. Consequences: CO₂ disabled until SCD40/41 + independent local alarm; pump blocked if the float switch is not operational; heating blocked if the nearby DS18B20 is failing.
- **R7 — One physical write per cabinet at a time.** The agent-runner takes a lock on `agent_locks` keyed by `cabinet_id` before each tick. A critical alert may interrupt or queue, but no parallel ticks.
- **R8 — Idempotency guaranteed by mqtt-bridge.** The bridge holds the authoritative set of processed `command_id`s in SQLite (`commands` table). Before publish, before ACK, before state verification: dedup by `command_id`. TTL is enforced — if `issued_at + ttl_ms < now`, status becomes `timeout` and the command never executes. Defense in depth (V1.x): a small custom ESPHome component with NVS ring buffer of 128 `command_id`s for local dedup on the ESP32.
- **R9 — Observability from the first commit.** Bridge, control-core, agent-runner, and web expose `/healthz`, `/readyz`, `/metrics` from the very first version. No Prometheus required initially; the operator web reads them directly.
- **R10 — Graphical operation, no required CLI.** Every operation lives in `packages/core-ops` as a pure function. The operator web is the primary client; tests are clients; future CLI would be a client. A web endpoint must contain no business logic — only Zod input validation + a call to core-ops.
- **R11 — Acceptance tests close every hit.** Each hit on the roadmap ends with an automated acceptance test. No advance to the next hit without closing the current one.
- **R12 — Physical plane separation.** *Plano hija*: 3.3 V, sensors, MQTT inspector, dashboard, calibration, photos. *Plano adulto*: 220 V, Sonoff T2, power strip, RCD, water, CO₂, secrets, OTA. Physically separated by an electrical box with a 30 mA RCD.

## High-Level Architecture

```text
  CAPA 8  PUBLIC DASHBOARD       Next.js, Recharts, SSE                  (V1.2+)
                                            ▲
  CAPA 7  OPERATOR WEB           Next.js, NextAuth, shadcn/ui            (V1.0)
                                            ▲
  CAPA 6  AGENT-RUNNER           Node.js, Vercel AI SDK, systemd        (V1.1+)
                                 propose intents (no commands)
                                            │
                                            ▼
  CAPA 5  PLANT MCP SERVER       @modelcontextprotocol/sdk + Zod        (V1.1+)
                                 tools: propose_* (write), get_*/read_* (read)
                                            │
                                            ▼
  CAPA 4  CONTROL-CORE           rules · safety · planner · 3 loops (no LLM)
                                 the plant survives here
                                            │
                                            ▼
  CAPA 3  CORE-OPS               pure TS, the only door to bridge + SQLite
                                            │
                                            ▼
  CAPA 2  MQTT-BRIDGE            ingest · command queue · state machine
                                 SQLite (WAL) as source of truth
                                            │
                                            ▼
  CAPA 1  HARDWARE               ESP32 (ESPHome) · Sonoff T2 (Z2M) · Pi · Logitech 4K
                                 autonomous local safe-state (R5)

  ─── In parallel ────────────────────────────────────────────────────
  Simulator       fake-esp32 + fake-plant (CI, dev, contributions without HW)
  Cockpit         Claude Code with --permission-mode default and propose-only tools
  Observability   /healthz /readyz /metrics on bridge, control-core, agent-runner, web
```

## Key Flows

### Flow 1 — LLM intent → physical command (the canonical loop, V1.1+)

1. ESP32 publishes sensor readings every 60 s on `tomatic/<cabinet>/sensors/+` (QoS 0, `retain=false`).
2. `mqtt-bridge` keeps a memory cache and persists the relevant readings into `sensor_readings`.
3. `agent-runner` wakes on its cron schedule (with jitter), takes a lock on `agent_locks` for the cabinet, and calls MCP read-tools (`get_sensor_status`, `check_safety_state`, optional `capture_camera_image`).
4. The LLM reasons and emits one or more `propose_intent` calls. The MCP server inserts each intent into `intents` with `proposed_by='llm'`.
5. `core-ops.submitIntent` forwards the intent to the **control-core**.
6. The control-core validates against the rules in chapter 7 of the source doc (cooldown, daily max, safe state, dependencies). The decision (`accepted` / `rejected` / `deferred`) is recorded back on the intent row.
7. If accepted, the planner produces concrete commands (exact ml, exact seconds, ordering) with `command_id`, `issued_at`, `ttl_ms` and inserts them into `commands` with `status='requested'`.
8. `mqtt-bridge` publishes each command to `tomatic/<cabinet>/commands/<actuator>/request` (QoS 1, `retain=false`).
9. ESP32 dedups by `command_id` against its NVS set, executes, and publishes a transactional ACK on `commands/<actuator>/ack`. Z2M-based actuators get state-verified instead.
10. The bridge transitions the row through `requested → accepted → running → succeeded | failed | timeout`.
11. `agent-runner` releases the lock. Web operativa, public dashboard, and metrics read from SQLite.

### Flow 2 — Local emergency (autonomous, no LLM, no network)

1. The ESP32 runs an `interval: 30s` block in ESPHome with safety predicates (`temp_air > 35 °C`, `temp_air < 5 °C`, tank empty).
2. On a true predicate, the firmware turns the pump switch off (in addition to the global default-off restore mode) and publishes `tomatic/<cabinet>/alerts/emergency` with severity `critical`.
3. `agent-runner` is subscribed to `alerts/+`. If a tick is in progress, it interrupts or queues; if not, it runs an emergency tick with the emergency prompt.
4. The plant is already protected by the time the LLM responds, not after.

### Flow 3 — Catch-up after downtime

1. On startup, `agent-runner` reads the latest `last_successful_tick_timestamp` from SQLite.
2. If more than N intervals have passed, it runs **one** catch-up tick with a summary of the missed window (sensor min/max/avg, critical events) — not N ticks. Avoids burst behavior.

## Contracts

### MQTT topic catalog (excerpt — full table in source doc §6)

All topics carry the cabinet prefix `tomatic/<cabinet-id>/...` from V1.0 to keep multi-cabinet additive.

| Topic | Direction | QoS | Retain | Payload shape |
|---|---|---:|---:|---|
| `tomatic/<cab>/sensors/+` | ESP32 → bridge | 0 | `false` | `{value, unit, sensor_id, raw, ts}` |
| `tomatic/<cab>/commands/<actuator>/request` | bridge → ESP32/Z2M | 1 | **false** | `{command_id, type, params..., issued_at, ttl_ms, reason, intent_ref}` |
| `tomatic/<cab>/commands/<actuator>/ack` | ESP32 → bridge | 1 | **false** | `{command_id, status, ts}` |
| `tomatic/<cab>/state/+` | bridge → DB | 0 | `true` (last) | last observed state per actuator |
| `tomatic/<cab>/alerts/emergency` | ESP32 → agent | 1 | **false** | `{type, value, severity}` |
| `tomatic/<cab>/agent/decisions` | agent → DB | 1 | **false** | `{prompt, intents, ts}` |
| `tomatic/<cab>/agent/heartbeat` | agent → bridge | 0 | `true` | `{last_tick_ts, status}` |
| `zigbee2mqtt/<dev>/set` | bridge → Z2M | 1 | **false** | `{state: "ON"\|"OFF"}` |
| `zigbee2mqtt/<dev>/` (state) | Z2M → bridge | 0 | `true` (last) | last observed device state |

Retain policy summary (R3): `sensors/* commands/* alerts/* = false`; `config/* heartbeat last_known_state = true`. CI test publishes a retained message on a `commands/+` topic and the job fails if found.

### Command lifecycle state machine

```
requested → accepted (ESP32 ACK in time | Z2M observed state matches desired)
          → rejected (firmware error: safe state, tank empty, dedup hit)
accepted  → running   (ESP32 only; Z2M does not have this state)
running   → succeeded (final ACK | state verification stable)
          → failed    (ACK error)
          → timeout   (no ACK / state mismatch within TTL)
```

Invariants:

1. `command_id` is globally unique. Re-issuing it produces no second effect.
2. An action counts as executed only when the final state is `succeeded`.
3. TTL expires: after `issued_at + ttl_ms < now` without `succeeded`, the control-core compensates (e.g. `pump timeout → safe state`).
4. The ESP32 keeps a local set of processed `command_id`s in NVS (R8). Survives reboots.

### Plant MCP Server tool surface (V1.1+)

Read tools (always available): `get_sensor_status`, `get_sensor_history`, `calculate_vpd`, `capture_camera_image`, `read_journal`, `get_plant_config`, `query_decision_history`, `check_safety_state`.

Write tools — **only intents, never commands**:
- `propose_intent(kind, target, reason)`
- `propose_watering(plant_ids, urgency, reason)` — *no ml, no seconds; control-core decides quantity*
- `log_observation(text, severity, tags[])`
- `request_human_review(reason, severity)`

### `infra.contract.yml`

The repo ships an experimental homelab project contract (`infra.contract.yml`) per `home-infra-protocol`. As of 2026-05 the portal does not consume it; `home-infra/catalog/services.yml` remains authoritative. The contract is kept up to date as the project moves toward V1.0 deployment so the protocol shape is validated on a real project.

## Storage & Data Layout

- **SQLite + WAL** (`better-sqlite3` + `PRAGMA journal_mode=WAL`). One writer (the bridge) and many readers (web, agent-runner, tests). Migrations from H0 with `drizzle-kit`. Continuous replication via `litestream` to `/share/Backups/tomatic/` on the NAS.
- **Tables (V1.0 minimum)**: `cabinets`, `sensor_readings`, `intents`, `commands`, `actuator_events`, `agent_locks`, `journal`, `calibration_sessions`, `safety_events`. Full DDL in source doc §10.
- **On-disk layout (NAS production)**:
  - `/share/Container/compose/tomatic/` — compose files + `.env` + Caddyfile
  - `/share/Container/compose/tomatic/data/sqlite/tomatic.db` (mounted into bridge + control-core + web)
  - `/share/Container/compose/tomatic/firmware/esphome/` — ESPHome Builder config
  - `/share/Backups/tomatic/` — litestream replicas
  - No Mosquitto data directory on NAS — the broker lives on the `zigbee` RPi (D-010 / `home-infra` ADR-0011); MQTT persistence is owned by that host.
- **Retention**: `sensor_readings` and `actuator_events` rolled by time partition (TBD in V1.x). Decision deferred — H4 acceptance does not depend on it.
- **No PII**: only environmental telemetry and operator-authored journal entries.

## Security & Privacy Notes

- **AuthN/AuthZ**: operator web behind `NextAuth` credentials provider, password from env (Doppler-injected). Public dashboard is read-only with no auth.
- **MQTT ACL**: enforced on the shared Mosquitto on the `zigbee` RPi (D-010 / `home-infra` ADR-0011). Per-client ACL — `tomatic_bridge` can publish to `tomatic/+/commands/+/request`, `zigbee2mqtt/+/set`, `tomatic/+/agent/+`; the ESP32 client can publish to `tomatic/+/sensors/+`, `tomatic/+/commands/+/ack`, `tomatic/+/alerts/+` only. No Tomatic client may publish to a command topic directly. Z2M and HA users keep their existing ACLs untouched.
- **Secrets**: Doppler project `tomatic` (to be created), config `dev`. Variables: `MQTT_PASSWORD`, `ANTHROPIC_API_KEY`, `ADMIN_PASSWORD`, `OTA_PASSWORD`, `WIFI_SSID`, `WIFI_PASSWORD`, optional `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`.
- **OTA**: ESPHome OTA password lives in `firmware/esphome/secrets.yaml`, `.gitignored`. First flash is USB; every subsequent firmware update is OTA over WiFi.
- **External exposure**: only `tomatic.lamanoriega.com` (V1.2 dashboard, behind `edge-caddy` on the NAS, public Let's Encrypt wildcard). Operator web is internal-only on `:3001`.
- **Physical safety**: see R5, R6, and the BOM safeguards in source doc §11 (float switch, check valve, max-runtime hardware relay deferred to V1.x defense-in-depth, IP54 enclosure, 30 mA RCD on the 220 V plane).

## Deployment Topology (planned)

| Component | Host | Port (host) | Notes |
|---|---|---|---|
| Mosquitto MQTT (shared) | `zigbee` RPi 10.0.0.139 | `1883` | **Reused — not a new broker.** Existing homelab Mosquitto already serving Z2M and HA. Tomatic-scoped ACL + dedicated `tomatic_bridge` user provisioned at H1. See `home-infra` ADR-0011 / project D-010. |
| `mqtt-bridge` | NAS | `8081` (`/healthz` `/readyz` `/metrics`) | Single MQTT client against `zigbee.home.arpa:1883`; reads/writes SQLite |
| `control-core` | NAS | `8082` | Pure logic, exposes metrics |
| `agent-runner` | NAS | (no port) | systemd-style (Docker `restart: unless-stopped`); cron schedule |
| `web-operativa` (Next.js) | NAS | `3001` | Internal only; behind `edge-caddy` if exposed publicly |
| `public-dashboard` (Next.js) | NAS | `3000` | V1.2+; exposed via `edge-caddy` as `tomatic.lamanoriega.com` |
| `esphome-builder` | NAS | `6052` (`network_mode: host`) | Editor + OTA host |
| `litestream` | NAS | (no port) | Continuous SQLite replication |
| Z2M (existing) | `zigbee` RPi 10.0.0.139 | `8080` (UI) | Already running 56 paired devices including 6× Sonoff T2 used by Tomatic; shares the broker on the same host |
| ESP32 cabinet-a | inside the tent | — | Publishes to `zigbee.home.arpa:1883` over WiFi |

The MQTT broker decision is settled (D-010 / `home-infra` ADR-0011): Tomatic reuses the existing shared Mosquitto on `zigbee.home.arpa:1883`. The `tomatic-bridge` runs a single MQTT client and uses the same broker for both `tomatic/<cabinet>/+` topics and `zigbee2mqtt/+/set` commands targeting Sonoff T2 actuators — no cross-broker plumbing is needed. The defense-in-depth for a `zigbee` reboot is R5 default-off + R8 TTL-based timeouts + the ESP32 autonomous local safe-state.

## Roadmap

- **V1.0-kernel** — H0 schemas/DB → H1 bridge → H2 idempotency → H3 retain policy → H4 control-core → H5 core-ops → H6 simulator → H7 observability → H8 agent-stub → H9 operator web → H10 ESPHome firmware on real hardware → H11 100-day soak with real firmware + safety checklist signed.
- **V1.1-agentic** — H12 plant-mcp-server → H13 agent-runner with Vercel AI SDK + systemd + catch-up → H14 Claude Code cockpit with restricted permissions → H15 provider switch (Anthropic/OpenAI/Ollama) → H16 100-day soak with real LLM in the loop.
- **V1.2-public** — H17 camera service → H18 public dashboard → H19 timelapse + gallery → H20 open-source polish + GitHub release 1.0.
- **Future** — V1.3 Ollama local · V1.4 multi-cabinet · V1.5 community plant presets · V1.6 quantum PPFD sensor · V2.0 computer vision (pests, deficiencies, ripe fruit) · V2.1 robotics (pruning, harvesting via ROS 2 over MQTT).

Each hit ends with an acceptance test (R11). The full list with exact acceptance criteria is in source doc §22.
