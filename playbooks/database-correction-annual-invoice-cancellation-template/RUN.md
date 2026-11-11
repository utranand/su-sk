<!-- generated:start digest=sha256:aac612ea0c0c783c6dd484ff6f72d7c01efa1cee593962e08f57028e40fb43d2 -->
# RUN — Annual invoice cancellation authoring contract

> Auto-generated from `playbook.yaml`. Hand notes live **outside**
> `<!-- generated:start/end -->`. Regenerate with `sidekicks playbook runmd .`

## Identity

- **playbook_id:** `database-correction.annual-invoice-cancellation`
- **kind:** `agent`
- **risk:** `mutating`
- **purpose:** Prepare current-schema complete unpaid annual invoice, all associated fiscal-year charge transactions (up to 12 months) and exclusive QR/mapping removal for annual payment (recurring_type_4) using the shared incident execution procedure.
- **when_to_use:** Use for a verified cancellation request for an annual payment invoice (recurring_type_4 / รายปี) where one annual bill covers the entire fiscal year; standard monthly cancellations use database-correction.invoice-cancellation.
- **uses_skills:** `sk-jira-connector`, `sk-database`, `sk-jira`

## Inputs

| Name | Required | Type | Description |
|---|---|---|---|
| `card` | yes | string | Jira incident or subtask key; read its parent and reuse the parent workspace. |
| `jira_env` | no | string | Registered Jira alias; verify it belongs to the incident. |
| `database_env` | no | string | Registered direct database alias; confirm target and tier. This default is not write consent. |
| `stage` | no | string | prepare, dry-run, commit, or verify. Stage selection never grants live-mutation permission. |
| `operation` | no | string | Specialized authoring contract for annual payment cancellation (recurring_type_4); monthly cancellations use database-correction.invoice-cancellation. |
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

- Read README.md and the linked canonical incident procedure. Bind {{card}}, {{jira_env}}, {{database_env}}, {{stage}} and {{work_item}}; reuse the parent workspace. Default to preparation and derive this incident scope, never prior IDs/counts. Dispatch by stage under canonical procedure section 1 before any later bullet: prepare authors; dry-run/commit reuse the unchanged reviewed package and original snapshot; verify is read-only, reuses matching existing evidence and never regenerates or rebinds 03/05 or the snapshot. Bind {{operation}} to annual-cancellation; redirect monthly or combined requests to appropriate playbooks before proceeding.
- Read all shared business/architect rules (PBR-01..06, PAR-01..07). Follow templates/01_investigate_readonly.sql.template as an authoring contract to inspect scoped live records for annual payment: verify recurring_type = 'recurring_type_4', resolve municipality -> SID -> entity; pin finite fiscal year (tax_year) and inspect the 1 annual invoice and all linked fiscal-year charge transactions (up to 12 months in garbage_fee_transaction); verify unpaid/unprinted state, no electronic system receipts (garbage_fee_receipt / receipt_id), no bank payments in ktb_transaction_item, and account for manual/outside payment claims.
- For stage=prepare only, use templates/02_backup.sql.template to author a consistent complete private backup export of the 1 annual invoice, all linked fiscal-year charge transactions, exclusive QR/barcode, mapping records, and detail settings with typed recovery checks. Use templates/03_correct.sql.template and templates/05_rollback.sql.template to author full incident-specific guards/DML/recovery in actual dependency order (mapping -> invoice -> QR -> transactions), ensuring all linked charge transactions are deleted so no collectible arrears remain. Source worksheets must never be executed as generated SQL.
- For stage=prepare only, use templates/04_verify_readonly.sql.template to author matching after-state and original-state verification: verify the 1 annual invoice AND all linked fiscal-year charge transactions are absent from database and collection/arrears; verify projected retained balance is 0 (or matches verified prior-year arrears) with zero orphaned transactions remaining. Preparation checks, actual dry-run, COMMIT and recovery are separate states; no plan or source template is runtime proof.
- Execute the specialized annual cancellation only for the current explicitly requested stage with valid exact-action approval and the canonical atomic receipt/rollback/commit concurrency controls. After a possible COMMIT failure reconcile read-only; never automatically retry or recover.
- Use templates/incident_report_th.md.template and templates/rules_verification_footprint.md.template to post exact invoice, QR, and all linked charge-transaction IDs/actions (highlighting the annual scope across the fiscal year), short interactive stage checklists, real per-table results and unique receipts. Refresh the parent registry and show the user the report.

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

# RUN — database-correction.annual-invoice-cancellation

Placeholder. `sidekicks playbook new` regenerates this file from `playbook.yaml`
inside `<!-- generated:start/end -->` markers. Put hand notes outside that block.


## Author notes — playbook run storage

Before executing phases, locate the framework checkout and set PLAYBOOK_DIR to this
playbook's resolved folder (`playbook working-folder <target>`). The active project's
catalog is the default even when a service is selected. Add `--project <name>` to
resolver/helper commands when targeting another project; this does not change ambient
project/service selection. Use `--project sidekicks` for the optional shared library.

Reuse incoming run/artifact anchors when resuming. For a new run, use the caller's
portable work_item or derive a fresh item from the playbook id and a run id, then resolve:

```bash
RUNBASE="$(node "$FRAMEWORK_ROOT/bin/sidekicks" playbook run-base "$WORK_ITEM" \
  --playbook "$PLAYBOOK_DIR")" || exit 1
ARTIFACTSDIR="$RUNBASE"  # apply explicit artifacts_dir / docs_dir overrides when supplied
export PLAYBOOK_OUTPUT_DIR="$ARTIFACTSDIR/outputs"
export PLAYBOOK_EVIDENCE_DIR="$ARTIFACTSDIR/evidence"
```

The default is `<physical-playbook>/artifacts/runs/<work-item>/{outputs,evidence}`.
Stop on resolution failure; do not substitute cwd or another project's artifact tree.
Honor explicit artifacts_dir; otherwise explicit docs_dir uses its sibling artifacts/;
otherwise ARTIFACTSDIR=RUNBASE. Resolve relative overrides against their owning workspace.

Pass the resolved anchors to helpers and require helpers to consume them. The playbook
CLI accepts `--evidence-dir "$PLAYBOOK_EVIDENCE_DIR"` and, when overriding deliverables,
`--output-dir "$PLAYBOOK_OUTPUT_DIR"`. Agent-produced text or binary artifacts in the
default run tree use `playbook write-output <work-item> outputs/<file> --source <file-or-stdin>
--playbook <target>` (or `evidence/<file>`); replacing an existing artifact requires
`--replace`. An explicit override needs its own supported write route. Preserve mandatory
provider-managed output locations and report the actual paths.

Omit manifest output-directory fields for these workspace defaults. Existing explicit
source-relative output declarations and historical runs retain their contracts. Record
actual report/gallery links and show the user the result. Generated files belong in the
resolved run directories; execution leaves reusable source unchanged. PROMPT.md remains
optional run adjustments only.
