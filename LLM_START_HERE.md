<!-- doc-version: 0.1.9 -->
# LLM Start Guide - tomatic

- Last Updated: 2026-09-23 - Codex (DocKit fleet update).
- Tooling update: see `docs/llm/DOCKIT_ADOPTION.md`; historical project status below is preserved.

## Read This First (Mandatory)

Welcome to Tomatic. This is an agentic indoor tomato grow system: a deterministic TypeScript control-core owns physical control; an LLM agent observes the cabinet through MCP and proposes intents that the control-core validates and turns into commands. The plant survives without the LLM.

Recommended reading order:
1. This file (rules, workflows, and current expectations).
2. **[`docs/reference/Tomatic_v3_2.docx`](docs/reference/Tomatic_v3_2.docx)** — source design document. Chapter 4 (hard rules R1-R12) is axiomatic; if any instruction contradicts a rule, pause and ask.
3. `AGENTS.md` — homelab integration rules and required reading from `~/src/home-infra/`.
4. `docs/PROJECT_CONTEXT.md` — vision, objectives, stakeholders, current state.
5. `docs/ARCHITECTURE.md` — 8 layers, hard rules R1–R12, key flows, contracts.
6. `docs/STRUCTURE.md` — planned monorepo layout.
7. `docs/VERSIONING_RULES.md` — version management policy.
8. `docs/llm/HANDOFF.md` — current work state and priorities.
9. `docs/llm/DECISIONS.md` — D-001..D-009 already recorded.

## Critical Rules (Non-Negotiable)

### Language Policy
- All code and documentation: English (update if your project needs a different language)
- Conversation with the user: Spanish
- Comments in code: English
- File names: English

<!-- DOCKIT-TEMPLATE:START doc-update-rules -->
### Documentation Update Rules
- Update docs/llm/HANDOFF.md every time you make a change.
- Append an entry to docs/llm/HISTORY.md in every session.
- HISTORY format: YYYY-MM-DD - <LLM_NAME> - <Brief summary> - Files: [list] - Version impact: [yes/no + details]
- Put long-form rationale in docs/llm/DECISIONS.md and link to it from HANDOFF.
- Prefer ASCII-only in docs/llm/* to avoid Windows encoding issues.
<!-- DOCKIT-TEMPLATE:END doc-update-rules -->

<!-- DOCKIT-TEMPLATE:START doc-sync-rules -->
### Documentation Sync Rules
- Keep this file's "Current Focus" section synchronized with docs/llm/HANDOFF.md "Current Status".
- Keep docs/STRUCTURE.md synchronized with the actual repository file tree.
- Keep docs/PROJECT_CONTEXT.md synchronized with architectural reality.
- Version markers (`<!-- doc-version: X.Y.Z -->`) in documentation files are managed by `scripts/bump-version.sh`. See `docs/version-sync-manifest.yml` for the full list of tracked files.
<!-- DOCKIT-TEMPLATE:END doc-sync-rules -->

<!-- DOCKIT-TEMPLATE:START commit-policy -->
### Commit Message Policy
- Every response that includes code or documentation changes must end with suggested commit information:
  - **Title:** under 72 characters
  - **Description:** under 200 characters, focused on user impact and why the change matters
- Format:
  `
  ## Commit Info
  **Title:** <concise title>
  **Description:** <short explanation of what changed and why>
  `
<!-- DOCKIT-TEMPLATE:END commit-policy -->

<!-- DOCKIT-TEMPLATE:START version-management -->
### Version Management
- Every commit that changes code/config files MUST include a version bump. The pre-commit hook enforces this.
- For version bumps, run `scripts/bump-version.sh <new_version>`; do not edit version strings manually.
- The bump script reads `docs/version-sync-manifest.yml` to update all tracked files atomically.
- Validate sync with `scripts/check-version-sync.sh` (also available as pre-commit hook).
- Do not bump versions without consulting docs/VERSIONING_RULES.md for impact level (patch/minor/major).
- Do NOT batch multiple code commits without versioning. No exceptions.
<!-- DOCKIT-TEMPLATE:END version-management -->

<!-- DOCKIT-TEMPLATE:START env-policy -->
### Environment Files (If Applicable)
- Do not edit generated .env.example files directly.
- Never change or remove existing credentials in .env or equivalent secret stores.
- If a new variable is needed, document it in the relevant README and ask the user to add it manually.
<!-- DOCKIT-TEMPLATE:END env-policy -->

## Current Focus (Snapshot)

Source of truth: docs/llm/HANDOFF.md.
- Last Updated: 2026-09-12 - Codex
- Working on: existing DocKit adoption registered and central independent-review
  policy adopted; Pre-H0 remains closed.
- Status: Source documentation is v0.1.8. Ready to start **H0 — Schemas + DB**
  in the next session. No application or runtime behavior changed.

Keep this section synchronized with the "Current Status" block in docs/llm/HANDOFF.md.

<!-- DOCKIT-TEMPLATE:START checklist -->
## Getting Started Checklist
- [ ] Read this entire file and update placeholders
- [ ] Review docs/PROJECT_CONTEXT.md
- [ ] Review docs/VERSIONING_RULES.md
- [ ] Read the current docs/llm/HANDOFF.md
- [ ] Install pre-commit hook: `cp scripts/pre-commit-hook.sh .git/hooks/pre-commit && chmod +x .git/hooks/pre-commit`
- [ ] Run `scripts/check-version-sync.sh` to verify version markers
- [ ] Confirm scope with the user
- [ ] Complete the work
- [ ] Update docs/llm/HANDOFF.md
- [ ] Add an entry to docs/llm/HISTORY.md
<!-- DOCKIT-TEMPLATE:END checklist -->

## Project-Specific Rules (Tomatic)

- **R1–R12 of the source design document are axioms.** See `docs/ARCHITECTURE.md` §Non-negotiables. Memorize them before writing code that touches MQTT, SQLite, the planner, the control loops, or the firmware.
- **Never edit `docs/reference/Tomatic_v3_2.docx`.** It is read-only context. If implementation diverges, write a new ADR in `docs/llm/DECISIONS.md` (D-010+) explaining the deviation.
- **Mandatory homelab updates** apply per `AGENTS.md` and `~/.claude/CLAUDE.md`: when changes affect host placement, ports, exposed URLs, or runtime version, update `~/src/home-infra/docs/` (INVENTORY/SERVICES/PROJECTS) **in the same session**, before declaring the change done.
- **Acceptance test on every hit (R11).** No advance to the next milestone without closing the current one with the test specified in the source doc §22.

## Quick Navigation
- Project Overview: docs/PROJECT_CONTEXT.md
- Architecture: docs/ARCHITECTURE.md
- Version Rules: docs/VERSIONING_RULES.md
- Version Sync Manifest: docs/version-sync-manifest.yml
- LLM Docs Index: docs/llm/README.md
- Current Work State: docs/llm/HANDOFF.md
- Change History: docs/llm/HISTORY.md
- Decision Rationale: docs/llm/DECISIONS.md
- Reviews (optional): docs/llm/REVIEWS.md
- Runbooks: docs/operations/

<!-- DOCKIT-TEMPLATE:START llm-communication -->
## LLM-to-LLM Communication
When handing off to another LLM:
1. Update docs/llm/HANDOFF.md with the current state and next steps.
2. Append an entry to docs/llm/HISTORY.md following the required format.
3. Ensure the snapshot in this file matches the latest status.
<!-- DOCKIT-TEMPLATE:END llm-communication -->

<!-- DOCKIT-TEMPLATE:START do-not-touch -->
## Do Not Touch Zones
Use the Do Not Touch section in docs/llm/HANDOFF.md to flag any files or areas that must remain unchanged without explicit approval from the user.
<!-- DOCKIT-TEMPLATE:END do-not-touch -->

<!-- DOCKIT-EXTERNAL-CONTEXT:START -->
<!-- DOCKIT-EXTERNAL-CONTEXT:END -->

<!-- DOCKIT-TEMPLATE:START independent-review-policy -->
### Independent Review Policy

The operator-wide Claude default is Opus 5.5, exact model
`claude-opus-5-5`, with high effort. This applies to Claude advisory/coauthor
work and independent source review unless the operator explicitly selects a
different model for the task. The operator's 2026-09-23 decision supersedes the
previous inherited Fable-first / quota-only Opus fallback preference.

1. Select the exact model explicitly for non-interactive review; do not use a
   floating `opus`, `best` or `default` alias as audit provenance.
2. Record effective model, effort, command, candidate revision/tree and validation
   packet in `docs/llm/REVIEWS.md` when present, otherwise the audited revision's
   HISTORY entry. Verify returned model metadata; a self-reported model name in
   the answer is not evidence. Do not silently substitute Fable, older Opus,
   Sonnet, Haiku or another model. If the selected model is unavailable or the
   provider changes it, preserve the packet and keep the required review gate
   open; continue only authorized work that does not depend on that verdict.
3. When durable Trace is enabled, keep non-commit object IDs such as tree hashes
   as plain text in HISTORY and HANDOFF Trace Anchors. Backtick-quoted hashes are
   reserved for commit provenance and must resolve as commits. Never remove
   backticks from a commit to bypass validation; classify a cross-repository
   commit with the Trace `external=repo@hash` field instead.

The auditor is independent and read-only: it reads primary files and evidence,
does not edit the candidate, and returns evidence-backed findings. The executor
must verify each finding, reconcile disagreements with the same auditor session
where practical, and preserve explicit unresolved disagreement for the operator.
A review verdict does not authorize build, deployment, runtime, secrets,
infrastructure, lifecycle or acceptance changes.

This synchronized section changes the model preference, not other project review
requirements. Explicit task-specific operator choices remain authoritative. A documented
operator-approved project model exception remains an exception until explicitly
superseded; exclusion alone is not approval of a different model. Other project
review constraints remain in force. Projects that exclude this section through
`.dockit-config.yml` must keep the exception and its authority visible.
The source policy belongs to LLM-DocKit; Claude Code's user `model` setting is a
separate runtime default and does not prove that project copies were synchronized.
<!-- DOCKIT-TEMPLATE:END independent-review-policy -->

<!-- DOCKIT-TEMPLATE:START footer -->
---

Every change must be documented. If you are unsure about a rule, ask the user before proceeding.
<!-- DOCKIT-TEMPLATE:END footer -->


<!-- DOCKIT-TEMPLATE:START delivery-evidence -->
## Optional Delivery Controls

For repeated delivery/infrastructure attempts, adopt `docs/DELIVERY_CONTRACT.md`
in the actual project mutation command. The project runs the real prerequisite
probe, records its observation, and calls `scripts/dockit-delivery-record.sh begin`
immediately before mutation; it records outcome and recovery afterward.

Copying scripts or a passing session validator is not integration. Require the
project's rerunnable negative test to demonstrate zero mutation calls for a failed
prerequisite, plus a current candidate-bound integration receipt. The journal
preserves attempt/review counts across sessions and versions. Repeating a failed
causal state or exhausting a budget requires bounded reassessment, never automatic
approval. Existing independent review and operational authority still apply.
Recovery must remain available independently of ordinary delivery checks.

Keep source publication, deployment and actual user acceptance separate. A short
HANDOFF names the observed result, current blocker and next concrete step.
<!-- DOCKIT-TEMPLATE:END delivery-evidence -->
