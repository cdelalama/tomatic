# Repository Structure Guide

This document describes the **planned** monorepo layout for Tomatic. As of 2026-05-02 the repo only carries the LLM-DocKit scaffold + homelab profile; application directories will land hit by hit (see [`llm/HANDOFF.md`](llm/HANDOFF.md)).

## Top-Level Layout (planned, source doc §21)

```
tomatic/
+- README.md
+- LICENSE                              (MIT)
+- AGENTS.md                            (homelab agent rules; CLAUDE.md is a symlink)
+- VERSION
+- CHANGELOG.md
+- VERSIONING_RULES.md → docs/VERSIONING_RULES.md
+- LLM_START_HERE.md
+- infra.contract.yml                   (homelab project contract; experimental)
+- pnpm-workspace.yaml                  (planned, H0)
+- biome.json                           (planned, H0)
+- tsconfig.base.json                   (planned, H0)
|
+- docs/
|  +- PROJECT_CONTEXT.md                vision, objectives, stakeholders
|  +- ARCHITECTURE.md                   layers, hard rules R1–R12, key flows, contracts
|  +- STRUCTURE.md                      this file
|  +- VERSIONING_RULES.md               version management policy
|  +- version-sync-manifest.yml         files tracked for version sync
|  +- reference/
|  |  +- README.md                      reference document index
|  |  +- Tomatic_v3_2.docx              canonical source design document
|  +- llm/
|  |  +- HANDOFF.md                     current focus
|  |  +- HISTORY.md                     append-only session log
|  |  +- DECISIONS.md                   ADRs
|  |  +- README.md                      LLM docs index
|  |  +- REVIEWS.md                     review notes
|  +- operations/
|     +- README.md
|     +- DEPLOY_PLAYBOOK.md             links to home-infra/docs/CONVENTIONS.md
|     +- API_CONTRACT.md                (planned)
|
+- apps/                                (planned, H0+)
|  +- web/                              Next.js 15 — operator web (V1.0+) + public dashboard (V1.2+)
|  +- agent-runner/                     Node.js + Vercel AI SDK + cron + systemd (V1.1+)
|  +- agent-stub/                       YAML loader, no LLM (V1.0)
|  +- claude-cockpit/                   .mcp.json + CLAUDE.md (dev only, V1.1+)
|  +- simulator/                        fake-esp32 + fake-plant (V1.0)
|
+- packages/                            (planned, H0+)
|  +- shared/                           Zod schemas + shared types
|  +- db/                               drizzle schema + migrations + WAL config
|  +- core-ops/                         pure TS API; the only door to bridge + SQLite (R10)
|  +- control-core/                     deterministic rules + planner + 3 loops (no LLM)
|  +- mqtt-bridge/                      ingest + command queue + state machine + idempotency
|  +- plant-mcp-server/                 MCP tools propose_* + reads (V1.1+)
|
+- firmware/                            (planned, H10)
|  +- esphome/
|     +- tomatic-esp32.yaml             single declarative YAML
|     +- secrets.yaml                   (gitignored)
|     +- README.md
|
+- presets/                             (planned, V1.0+)
|  +- trophy-tomato.yaml
|  +- cherry-tomato.yaml
|  +- lettuce.yaml
|  +- strawberry.yaml
|
+- prompts/                             (planned, V1.1+)
|  +- system.md
|  +- emergency.md
|  +- weekly-review.md
|
+- hardware/                            (planned, V1.0)
|  +- BOM-basic.md                      what Carlos already has
|  +- BOM-reliable.md                   defense-in-depth additions
|  +- wiring/                           pin maps + diagrams
|  +- safety-checklist.md
|  +- enclosure.stl
|
+- compose/                             (planned, H10–H11)
|  +- docker-compose.base.yml           common services + volumes
|  +- docker-compose.sim.yml            fake-esp32 + fake-plant + agent-stub
|  +- docker-compose.prod-nas.yml       QNAP NAS production
|  +- docker-compose.prod-pi.yml        Raspberry Pi 5 alternative
|  +- mosquitto/
|  |  +- mosquitto.conf
|  |  +- acl.tomatic
|  +- caddy/Caddyfile
|  +- .env.example
|
+- scripts/
|  +- bump-version.sh                   manage doc-version markers
|  +- check-version-sync.sh
|  +- pre-commit-hook.sh
|  +- dockit-validate-session.sh
|  +- dockit-generate-external-context.sh
|  +- (planned) install.sh
|  +- (planned) flash-esp32.sh          invokes ESPHome Builder
|  +- (planned) acceptance-test-pump.sh   R2 acceptance
|  +- (planned) acceptance-test-retain.sh R3 acceptance
|  +- (planned) backup.sh
|
+- tests/                               (planned, H0+)
|  +- e2e/
|  +- integration/
|  +- unit/
|
+- .claude/
|  +- settings.json
|  +- checklists/
|  |  +- homelab-project.md             installed by the homelab profile
|  +- rules/
|  |  +- require-docs-on-code-change.md
|  +- skills/
|     +- adopt-dockit/SKILL.md
|     +- update-docs/SKILL.md
|
+- .github/
|  +- ISSUE_TEMPLATE/
|  +- PULL_REQUEST_TEMPLATE.md
|  +- workflows/
|     +- doc-validation.yml             CI doc validation (DocKit default)
|     +- (planned) test.yml             pnpm install + pnpm test
|     +- (planned) sim-soak.yml         100 simulated days at 100× speed
|     +- (planned) release.yml          changesets + ghcr.io image publish
```

## What exists today (2026-05-02)

Only the DocKit scaffold + homelab profile:

- Top-level: `README.md`, `AGENTS.md`, `CLAUDE.md → AGENTS.md`, `LICENSE`, `LLM_START_HERE.md`, `VERSION`, `CHANGELOG.md`, `infra.contract.yml`, `.gitignore`.
- `docs/PROJECT_CONTEXT.md`, `ARCHITECTURE.md`, `STRUCTURE.md`, `VERSIONING_RULES.md`, `version-sync-manifest.yml`.
- `docs/reference/README.md`, `Tomatic_v3_2.docx` (canonical source design document).
- `docs/llm/HANDOFF.md`, `HISTORY.md`, `DECISIONS.md`, `README.md`, `REVIEWS.md`.
- `docs/operations/README.md`, `DEPLOY_PLAYBOOK.md`, `API_CONTRACT.md`.
- `scripts/` with the DocKit version + validate scripts and the pre-commit-hook template.
- `.claude/` with settings, the homelab checklist, rules, and skills.
- `.github/` with issue/PR templates and `doc-validation.yml`.
- `src/`, `tests/` (gitkeep only).

## Directory Descriptions

| Path | Purpose | Notes |
|------|---------|-------|
| `apps/web` | Next.js 15 operator web (V1.0) and public dashboard (V1.2). Two apps sharing components | Lands at H9 |
| `apps/agent-stub` | YAML-driven intent injector to validate the path without LLM | Lands at H8 |
| `apps/agent-runner` | Vercel AI SDK + cron + lock-per-cabinet + catch-up | Lands at H13 |
| `apps/simulator` | `fake-esp32` + `fake-plant` for CI and zero-hardware development | Lands at H6 |
| `packages/shared` | Zod schemas (`Intent`, `Command`, `AckTransactional`, `StateVerificationResult`, `SafetyState`, `OverrideInput`) | Lands at H0 |
| `packages/db` | drizzle schema + migrations + better-sqlite3 + WAL config | Lands at H0 |
| `packages/core-ops` | Pure TS API. R10: zero `mqtt.connect` or `better-sqlite3` outside this package | Lands at H5 |
| `packages/control-core` | Deterministic rules, planner, 3 loops (light/water/climate). Vitest with 100-day soak | Lands at H4 |
| `packages/mqtt-bridge` | MQTT ingest, command queue, state machine, idempotency by `command_id` | Lands at H1 |
| `packages/plant-mcp-server` | MCP server with `propose_*` write tools and reads | Lands at H12 |
| `firmware/esphome/tomatic-esp32.yaml` | Single declarative YAML for the ESP32 firmware | Lands at H10 |
| `presets/` | Per-cultivar YAML configs | Lands with V1.0 |
| `prompts/` | LLM prompts (system, emergency, weekly review) | Lands at H13 |
| `hardware/` | BOM, wiring, safety checklist, enclosure | Lands with V1.0 |
| `compose/` | Docker Compose with `sim` / `prod-nas` / `prod-pi` profiles | Lands at H10–H11 |

## Generated / Runtime Directories (not committed)

- `node_modules/`, `dist/`, `build/`, `.next/` — pnpm + Next.js outputs.
- `coverage/` — vitest output.
- `firmware/esphome/.esphome/` — ESPHome build cache.
- On the NAS only: `data/sqlite/`, `data/esphome/` — runtime data, replicated by litestream. No `data/mosquitto/` on NAS — the shared MQTT broker lives on the `zigbee` RPi (D-010 / `home-infra` ADR-0011); the simulator profile `docker-compose.sim.yml` still spins its own Mosquitto with `compose/mosquitto/` config for zero-hardware development.

## Naming Conventions

- TypeScript packages prefixed with `@tomatic/...` in `package.json`.
- MQTT topics always carry the cabinet prefix `tomatic/<cabinet-id>/...` from V1.0 (R-section §6 of the source doc).
- ESPHome device name: `tomatic-cabinet-<id>` (e.g. `tomatic-cabinet-a`).
- `command_id` format: `cmd_<YYYYMMDD>_<HHMMSS>_<short-hash>` (example in source doc §6).
- Z2M friendly names already in use: `luz_armario`, `extractor`, `intractor`, `manta_termica`, `humidificador`, `co2_solenoide`.

## Onboarding Notes

- Read `docs/reference/Tomatic_v3_2.docx` end-to-end before touching code. Chapter 4 is axiomatic.
- For deploy work, **AGENTS.md** is the read-first document and `~/src/home-infra/docs/CONVENTIONS.md` covers NAS quirks (no `docker compose` subcommand, `/usr/local/lib/docker/cli-plugins/docker-compose`, no `bash`, deploy by `docker save | ssh ... 'docker load'`).
- The homelab checklist `.claude/checklists/homelab-project.md` is the gate before declaring deploy done.
- The pre-commit hook enforces version bumps; run `scripts/bump-version.sh <new>` rather than editing markers by hand.
