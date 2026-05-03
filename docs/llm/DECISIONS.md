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

## D-011 — Pull ESPHome Builder out of H10 to Pre-H0

- **Date:** 2026-05-02
- **Status:** accepted
- **Supersedes (partially):** the source design document roadmap step at H10
  ("Levantar ESPHome Builder en Docker") — only the *provisioning* of the
  Builder moves earlier. Everything else in H10 (real `tomatic-esp32.yaml`,
  first USB flash via `web.esphome.io`, pump acceptance test on real
  hardware) stays at H10.

### Context

The source design document `reference/Tomatic_v3_2.docx` chapter 19 places
the ESPHome Builder container provisioning inside H10 (Firmware ESP32),
together with writing the actual `tomatic-esp32.yaml`, the first USB flash
from the operator's PC, and the `acceptance-test-pump.sh` against real
hardware. That grouping is convenient ("everything ESP32-related in one
hit") but it conflates two independent pieces of work:

- **Provisioning the Builder container on the NAS** — pure infrastructure.
  No application code, no firmware logic, no acceptance test. Validates
  Container Station, the compose path, network mode, and the project's
  filesystem layout under `/share/Container/compose/tomatic/`.
- **Writing and flashing the firmware** — application work that depends on
  the Builder being up and on hardware being wired in the cabinet.

The Builder itself has zero dependency on H0–H9. It is a single Docker
container holding a YAML editor and an OTA target, with `network_mode:
host`, port `:6052`, and a single mounted volume for `firmware/esphome/`.
Provisioning it before H0 was discussed and approved by Carlos on
2026-05-02 with three motivations:

1. **Validate the NAS deploy chain early.** It is the first Tomatic
   container to land on the NAS. If `network_mode: host` collides with
   anything, if Container Station has quirks, if the compose path is
   wrong, those problems surface now — with a low-stakes service — rather
   than at H10 when the bridge, control-core, and web operativa all land
   together.
2. **Enable YAML exploration before firmware work begins.** The Monaco
   editor is now reachable at `http://10.0.0.220:6052/`. Carlos can
   sketch ESPHome YAML for the sensors he already has at home (FS400-SHT35,
   SCD41, BH1750, ADS1115 + soil capacitive ×3, DS18B20 ×2) and learn the
   ESPHome dialect well before H10 needs to ship the real
   `tomatic-esp32.yaml`. The "modo hija" educational angle benefits too:
   the YAML editor is a visible, low-risk artifact for early sessions.
3. **First registered service in the homelab catalog.** The Builder lands
   in `home-infra/catalog/services.yml` with `interface: web` (per
   `home-infra-protocol` 0.2.0), so `infra-portal` renders it from day
   one. This validates the operator's ability to *find* Tomatic services
   in the portal before any of them carry plant-survival weight.

### Decision

Provision `tomatic-esphome` (image `ghcr.io/esphome/esphome:latest`,
container name `tomatic-esphome`, host network on `:6052`, compose path
`/share/Container/compose/tomatic/`) **before H0**, registered as a
catalog service in `home-infra` and rendered by `infra-portal`. H10 keeps
all the firmware-writing work; H10 no longer installs the Builder.

### Implementation deltas vs the source design document

The compose block in chapter 14 of the source doc is taken nearly
verbatim, with two deliberate differences:

- **`privileged: true` removed.** Per chapter 12 of the source doc, the
  first ESP32 flash is performed from the operator's PC via
  `https://web.esphome.io` with a USB cable. The NAS never holds the
  USB. Subsequent flashes are OTA over WiFi from this Builder. There is
  no scenario where the container needs `privileged`, so the flag is
  dropped (least-privilege).
- **Compose file is single-service today.** The compose file currently
  contains only the `esphome-builder` service. Subsequent hits (H1+)
  extend the *same* compose file with the bridge, control-core, web,
  etc. — additive, not a rewrite. This keeps the source-doc structure
  intact while letting the deploy chain validate one container at a
  time.

### Consequences

- `home-infra/catalog/services.yml` declares `tomatic-esphome` as a
  first-class service with `interface: web` and an HTTP `:6052` status
  probe. `infra-portal` renders it; live verification on 2026-05-02 at
  23:28 UTC: `state=up`, HTTP 200.
- `home-infra/docs/INVENTORY.md`, `docs/SERVICES.md`, and `docs/PROJECTS.md`
  reflect the new container and the Pre-H0 status of Tomatic.
- The roadmap text in `docs/PROJECT_CONTEXT.md` and `docs/ARCHITECTURE.md`
  is still accurate at the hit-by-hit granularity — H0 is still next, the
  acceptance tests for H1–H11 are unchanged. The only change is that the
  Builder service is up *during* H0–H9 instead of arriving at H10.
- The operator (Carlos) can now open the Builder at
  `http://10.0.0.220:6052/` and start exploring the ESPHome YAML dialect
  on a scratch file. Any scratch YAML lives under
  `/share/Container/compose/tomatic/firmware/esphome/` and will be
  superseded (or kept as an example) when H10 ships the real
  `tomatic-esp32.yaml`.
- `secrets.yaml` for the firmware does NOT exist yet and is not needed
  until H10 (when the real YAML lands). The Builder runs without
  secrets.
- This decision is `home-infra`-side a homelab inventory change, not an
  ADR; the corresponding `home-infra` commit `87400d8` registers the
  service.

### Addendum 2026-05-03 — rename to `esphome-builder` (drop the `tomatic-` prefix)

Audit feedback on 2026-05-03 pointed out that `tomatic-esphome` is a
misleading name: ESPHome is a generic ESP32 firmware development tool,
and Tomatic is just its first adopter. Following the homelab
convention (cf. `mosquitto` on the `zigbee` RPi, which is shared and
not called `tomatic-mosquitto` even though Tomatic is one of its
consumers), the container, compose dir, and catalog id are renamed:

- container_name: `tomatic-esphome` → `esphome-builder`
- compose dir on NAS: `/share/Container/compose/tomatic/` →
  `/share/Container/compose/esphome-builder/`
- `home-infra/catalog/services.yml` id: `tomatic-esphome` →
  `esphome-builder`; tags drop `tomatic`; description rewritten as a
  shared homelab service with Tomatic as first adopter
- tomatic project no longer claims this container as one of its
  owned `services:`; instead it is referenced from the new
  `consumes:` block in `infra.contract.yml` alongside `mosquitto`
- The original compose path `/share/Container/compose/tomatic/` is
  reserved for tomatic-owned containers (bridge, control-core,
  web operativa) starting at H1; it does not exist on the NAS today

The portal picked up the rename at 2026-05-03 14:30 UTC: catalog
shows `esphome-builder`, `tomatic-esphome` is gone, `state=up` HTTP
200. `home-infra` commit `15c0a1c` carries the catalog and INVENTORY
edits.

This addendum does not change D-011's substance — the Pre-H0
decision and its rationale stand. It only fixes the name and
ownership model that surfaced under audit.

---

## D-012+ (future)

Add new ADRs here when the implementation forces a deviation from the source design document or when an open question is resolved (e.g. public dashboard host, Doppler project layout).
