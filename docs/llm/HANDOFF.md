<!-- doc-version: 0.1.6 -->
# LLM Work Handoff

This file is the current operational snapshot. Long-form rationale lives in
`docs/llm/DECISIONS.md`.

## Current Status

- Last Updated: 2026-05-03 - Claude Opus 4.7 (1M context) + GPT-5 (concur) + Carlos (arbiter)
- Session Focus: Adopted LLM-DocKit 4.7.0's new SessionStart-side enforcement primitive directly into this repo (script `scripts/dockit-bootstrap-context.sh` + `.claude/settings.json` SessionStart hook entry) so the failure mode named in upstream **DF-033** is closed for the project under active work. Trigger: 2026-05-03 incident in a sibling Codex CLI session inside `home-infra-protocol` where the agent gave a partial ecosystem opinion because it had not read `LLM_START_HERE.md` despite the rule being declared at lines 9 and 87. Fix is mechanical (hook-injected `additionalContext` at session start), not more prose; rationale recorded as upstream LLM-DocKit D-007.
- Status: **Pre-H0 still done (clean), no application-code change in this session.** `esphome-builder` continues `state=up` at `http://10.0.0.220:6052/`. Tomatic still owns no container; consumes the shared `esphome-builder` and (planned) `mosquitto`. Ready for **H0 — Schemas + DB** next session, with the new SessionStart hook now active so the next session loads onboarding before answering. **Version bumped 0.1.5 → 0.1.6 (patch)** because the pre-commit hook correctly classified the two new scaffold-path files (`scripts/dockit-bootstrap-context.sh`, `.claude/settings.json`) as code/config that introduces new behavior in this repo and therefore deserves a version bump — my earlier reasoning that "scaffold-managed paths are excluded" was a misreading of the hook's intent (the hook excludes `docs/`, `.github/`, `.claude/` rules-only paths, but `.claude/settings.json` carries actual hook configuration that changes runtime behavior). Side observation: the hook's exclusion regex `'^\\.claude/'` has a stray double-backslash and would never match anyway — captured as a follow-up note for a separate session, not addressed in this commit. The upstream version that landed the primitive is LLM-DocKit 4.7.0 — when next `dockit-sync` runs in this repo it will be a no-op for both files (already up to date).

## Project Summary

Tomatic is an autonomous indoor tomato grow system. A deterministic TypeScript control-core owns physical control and runs three loops (light, water, climate) without any LLM. The LLM proposes intents through an MCP server; the control-core validates, plans, and emits commands with `command_id` + TTL idempotency. The plant survives without the LLM.

Source design document: [`../reference/Tomatic_v3_2.docx`](../reference/Tomatic_v3_2.docx) (Carlos, May 2026). Chapter 4 (hard rules R1-R12) is axiomatic — any contradicting instruction must pause and ask.

## Next Steps (V1.0-kernel, hit by hit)

1. **H0 — Schemas + DB** (next session, ~0.5 weeks):
   - Set up `pnpm-workspace.yaml`, `biome.json`, `tsconfig.base.json` with `strict: true` and `noUncheckedIndexedAccess: true`.
   - Create `packages/shared` with Zod schemas: `SensorReading`, `Intent`, `Command`, `AckTransactional`, `StateVerificationResult`, `SafetyState`, `OverrideInput`.
   - Create `packages/db` with the drizzle schema from source doc §10 (`cabinets`, `sensor_readings`, `intents`, `commands`, `actuator_events`, `agent_locks`, `journal`, `calibration_sessions`, `safety_events`).
   - Configure better-sqlite3 with `PRAGMA journal_mode=WAL; synchronous=NORMAL; foreign_keys=ON`.
   - Add migration `0001_init.sql`. Verify `pnpm --filter @tomatic/db migrate` and `migrate:rollback` work.
   - Vitest tests for schema validation (accepts valid payloads, rejects malformed).
   - GitHub Action `.github/workflows/test.yml` running `pnpm install && pnpm typecheck && pnpm test`.
2. **H1 — mqtt-bridge** (~1 week): MQTT ingest into SQLite, command queue with publishing.
3. **H2 — Idempotency** (~1 week): `command_id` dedup + TTL + `acceptance-test-pump.sh`.
4. **H3 — Retain policy** (~0.2 weeks): `acceptance-test-retain.sh` in CI.
5. ... full list in [`PROJECT_CONTEXT.md`](../PROJECT_CONTEXT.md) §Upcoming Milestones.

## Key Decisions

The first nine architectural decisions are pre-recorded in [`DECISIONS.md`](DECISIONS.md):

- D-001 Control determinista primero (the plant survives without LLM)
- D-002 LLM proposes intents, control-core decides (no `propose_*` tool ever emits a command)
- D-003 ESPHome único — no MicroPython
- D-004 NAS-first deployment (Z2M stays on the existing `zigbee` RPi)
- D-005 SQLite + WAL + drizzle + litestream
- D-006 Cabinet prefix in every MQTT topic from V1.0
- D-007 Operator web in V1.0 (no required CLI)
- D-008 CO₂ disabled by default until independent CO₂ alarm is in place
- D-009 Vercel AI SDK from V1.1 for multi-provider agnosticism

## Open Questions

1. ~~**Mosquitto topology**~~: **resolved 2026-05-02 — D-010** (mirrors `home-infra` ADR-0011). Tomatic reuses the existing shared Mosquitto on the `zigbee` RPi (`zigbee.home.arpa:1883`); no NAS broker. The bridge runs a single MQTT client. ACL + `tomatic_bridge` user provisioned at H1 on the existing broker.
2. **Public dashboard hosting**: NAS behind `edge-caddy` with a `tomatic.lamanoriega.com` static DNS record (consistent with current homelab pattern), or Vercel free tier? Default for now: NAS + edge-caddy. Revisit at H18.
3. **Doppler project naming**: `tomatic` (single config `dev`) or `tomatic-dev` + `tomatic-prd`? Decision deferred to H10.
4. ~~**MQTT cross-broker routing for Z2M actuators**~~: **moot after D-010**. With a single broker (the one on `zigbee`), `tomatic/<cabinet>/+` and `zigbee2mqtt/+/set` live on the same broker; no cross-broker bridge is needed.

## Files To Read First

- [`../reference/Tomatic_v3_2.docx`](../reference/Tomatic_v3_2.docx) — source design document, chapter 4 (hard rules) is axiomatic.
- `README.md`, `AGENTS.md`, `LLM_START_HERE.md`.
- [`docs/PROJECT_CONTEXT.md`](../PROJECT_CONTEXT.md), [`docs/ARCHITECTURE.md`](../ARCHITECTURE.md), [`docs/STRUCTURE.md`](../STRUCTURE.md).
- [`docs/llm/DECISIONS.md`](DECISIONS.md).
- `~/src/home-infra/docs/CONVENTIONS.md`, `INVENTORY.md`, `SERVICES.md` (homelab source of truth).
- `.claude/checklists/homelab-project.md` (the gate before deploy).

## Do Not Touch

- [`../reference/Tomatic_v3_2.docx`](../reference/Tomatic_v3_2.docx) — read-only source. Never edit; if a design contradiction arises, write a new ADR in [`DECISIONS.md`](DECISIONS.md) explaining the deviation.
- The `pentagi/` git submodule pattern from PentAGI-Lab does not apply here — Tomatic has no upstream submodule.
