<!-- generated:start digest=sha256:16d837f9df30642921216d6e2c091fad9cdd20f6c7dfc349eb49d8a2b27a7fe7 -->
# RUN — Invoice cancellation authoring contract

> Auto-generated from `playbook.yaml`. Hand notes live **outside**
> `<!-- generated:start/end -->`. Regenerate with `sidekicks playbook runmd .`

## Identity

- **playbook_id:** `database-correction.invoice-cancellation`
- **kind:** `agent`
- **risk:** `mutating`
- **purpose:** Prepare current-schema complete unpaid invoice, charge-transaction and exclusive QR/mapping removal using the shared incident execution procedure.
- **when_to_use:** Use for a verified cancellation request; combined requests use database-correction.invoice-receipt-incident.
- **uses_skills:** `sk-jira-connector`, `sk-database`, `sk-jira`

## Inputs

| Name | Required | Type | Description |
|---|---|---|---|
| `card` | yes | string | Jira incident or subtask key; read its parent and reuse the parent workspace. |
| `jira_env` | no | string | Registered Jira alias; verify it belongs to the incident. |
| `database_env` | no | string | Registered direct database alias; confirm target and tier. This default is not write consent. |
| `stage` | no | string | prepare, dry-run, commit, or verify. Stage selection never grants live-mutation permission. |
| `operation` | no | string | This specialized authoring contract covers cancellation only; use the combined incident playbook when both are requested. |
| `work_item` | no | string | Reuse supplied work item/run anchors; otherwise derive from the parent card. Never create a separate child workspace. |
| `user_guide` | no | string | Task-specific adjustments within the current rules, exact scope and approval boundaries. |

Pass as `--input name=value` (exported as `PLAYBOOK_INPUT_<NAME>`).

## Prerequisites

1. `sidekicks playbook validate <this-folder>` must be OK.
2. Runtime requirements: (none declared).
3. Load and obey **all** rule-ids in: `playbook-business-rules.md`, `playbook-architect-rules.md`.
4. Collect every required parameter before mutating steps.
5. Optional `user_guide` (this-run free text): merge preferences within rules — playbook procedure/safety always win; never rewrite the folder.

## Procedure

- Read README.md and the linked canonical incident procedure. Bind {{card}}, {{jira_env}}, {{database_env}}, {{stage}} and {{work_item}}; reuse the parent workspace. Default to preparation and derive this incident scope, never prior IDs/counts. Dispatch by stage under canonical procedure section 1 before any later bullet: prepare authors; dry-run/commit reuse the unchanged reviewed package and original snapshot; verify is read-only, reuses matching existing evidence and never regenerates or rebinds 03/05 or the snapshot. Bind {{operation}} to cancellation only; redirect combined or mismatched requests to database-correction.invoice-receipt-incident before proceeding, never widen this specialized scope.
- Read all shared business/architect rules. Follow templates/01_investigate_readonly.sql.template as an authoring contract to inspect scoped live records early, resolve the real tenant/entity and prove eligibility and dependencies for exact invoice/charge-transaction/QR/mapping sets; trace owner collection/balance and cron/backfill behavior, including evidenced transaction-only follow-ups.
- For stage=prepare only, use templates/02_backup.sql.template to author a consistent complete private backup export and typed recovery checks. Use templates/03_correct.sql.template and templates/05_rollback.sql.template to author full incident-specific guards/DML/recovery, with independent review and source digests. Source worksheets must never be executed as generated SQL.
- For stage=prepare only, use templates/04_verify_readonly.sql.template to author matching after-state and original-state verification including canceled charge absence and exact retained collection IDs/balance. Preparation checks, actual dry-run, COMMIT and recovery are separate states; no plan or source template is runtime proof.
- Execute the specialized operation only for the current explicitly requested stage with valid exact-action approval and the canonical atomic receipt/rollback/commit concurrency controls. After a possible COMMIT failure reconcile read-only; never automatically retry or recover.
- Use templates/incident_report_th.md.template and templates/rules_verification_footprint.md.template to post exact invoice and charge-transaction IDs/actions, short interactive stage checklists, real per-table results and unique receipts. Refresh the parent registry and show the user the report.

## Gates

- **human_gate.required:** true
- **before_phase:** `step-5`
- **description:** Exact-action authorization is required for each live dry-run, commit and later restore; source contracts and stage values are not consent.
- CLI stops with exit `10` at a gated step unless `--yes` is passed after human approval.
- **Gated phases:** `step-5`

## Outputs (mission folder)

- Default deliverables: `artifacts/runs/<work-item>/outputs/` inside this playbook; helpers receive `PLAYBOOK_OUTPUT_DIR`.
- **After the run:** show the user that folder and summarize / open the report.

## Evidence

- Runs write evidence under this playbook’s `artifacts/runs/<work-item>/evidence/` (or explicit `--evidence-dir`).
- Each run produces `receipt.json`, `RESULT.md` (points at the shared output folder), and per-step stdout/stderr.

## Stop / failure handling

| Exit | Meaning |
|---|---|
| 0 | OK |
| 1 | Usage |
| 2 | Validation / missing inputs |
| 10 | Human gate — approve then re-run with `--yes` |
| 11 | Agent hold (pure-agent) — follow RUN.md / template |
| 12 | Script step failed |
| 13 | Timeout |
| 14 | Runner error |

- Do not skip ahead when a prior phase failed a gate or script.
- Prefer `playbook run --dry-run` to plan before any mutating step.
- Never invent alternate steps; change `playbook.yaml` and regenerate this block instead.
<!-- generated:end -->

## Author notes

<!-- Hand-written judgment notes go below. `playbook runmd` preserves this section. -->
