<!-- generated:start digest=sha256:575dcde05758a0d3a530fcb33e5133c52b948f6a71605ed0f4736b11687ee4e2 -->
# RUN — Invoice correction and unpaid cancellation / แก้ไขยอดและยกเลิกใบแจ้งหนี้

> Auto-generated from `playbook.yaml`. Hand notes live **outside**
> `<!-- generated:start/end -->`. Regenerate with `sidekicks playbook runmd .`

## Identity

- **playbook_id:** `database-correction.invoice-receipt-incident`
- **kind:** `agent`
- **risk:** `mutating`
- **purpose:** Prepare, review, dry-run, commit and verify a scoped utility billing correction with recoverable SQL and evidence-backed Jira checklists.
- **when_to_use:** Smart Utilities incidents involving invoice amount changes, unpaid invoice cancellation, or receipt investigation. Applies per verified utility/schema; paid-record refunds and unrelated operational fixes require their own workflow.
- **uses_skills:** `sk-jira-connector`, `sk-database`, `sk-jira`

## Inputs

| Name | Required | Type | Description |
|---|---|---|---|
| `card` | yes | string | Jira incident or subtask key; read its parent and reuse the parent workspace. |
| `jira_env` | no | string | Registered Jira alias; verify it belongs to the incident. |
| `database_env` | no | string | Registered direct database alias; confirm target and tier. This default is not write consent. |
| `stage` | no | string | prepare, dry-run, commit, or verify. Stage selection never grants live-mutation permission. |
| `operation` | no | string | from-card, amount, cancellation, or combined; derive exact approved operations and periods from this incident. |
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

- Read prompts/task.md section 1 and bind {{card}}, {{jira_env}}, {{database_env}}, {{stage}}, {{operation}} and {{work_item}}. Read the parent first, reuse its workspace and register this playbook/version for each applicable card; preserve reporter content. Preparation is the default; load both shared rule files and references/sql-contracts.md. Dispatch by stage under canonical procedure section 1 before any later bullet: prepare authors; dry-run/commit reuse the unchanged reviewed package and original snapshot; verify is read-only, reuses matching existing evidence and never regenerates or rebinds 03/05 or the snapshot.
- Follow prompts/task.md section 2: inspect attachments and directly query scoped live records early. Resolve municipality to real system ID and unique payer entity before querying financial records; establish exact invoice/transaction/QR/mapping IDs, periods, counts, printing, receipts, saved bank references, fiscal years and dependencies. Discover schema rather than reusing incident values.
- For stage=prepare only, use prompts/task.md section 3 and references/sql-contracts.md to author the five-file SQL suite, immutable full-row backup, exact before/after guards, cancellation recovery and matching verification for every variant. Synchronize full QR/barcode amount tokens. Obtain independent review and bind the actual source digests; preparation checks are not runtime evidence.
- Use templates/report.md.template and templates/checklist.md.template to show exact invoice ID/action lists and a short interactive readiness checklist under prompts/task.md section 4. Evaluate every applicable PBR/PAR requirement once, mark inapplicable receipt work N/A and unknown evidence unmet, and separate preparation, live dry-run, commit and recovery states.
- For an explicitly requested dry-run with valid exact-action user approval, follow prompts/task.md section 5 in one direct atomic session ending ROLLBACK. Capture stdout, stderr, runtime counts, exact removed IDs, readbacks and the rollback completion marker, then verify the complete original state from a separate read-only connection. Without that approval record NOT RUN.
- For an explicitly requested commit after the matching successful dry-run and separate exact-action user approval, follow prompts/task.md section 6. Resolve any demonstrated concurrent-payment/QR insertion risk through a confirmed payer quiet window or separately authorized reviewed control. Refresh guards, change only the final ROLLBACK to COMMIT, execute once and verify the exact committed projection from a fresh connection; never retry an uncertain outcome.
- After a verified COMMIT only, follow prompts/task.md section 7 to bind standby recovery to actual committed after-images and restore complete original invoice, QR and mapping rows in dependency order. Keep recovery rollback-default, reject drift/conflicts and obtain fresh authorization for a later live restore. Automatic rollback on a failed uncommitted transaction is immediate and needs no second permission.
- Follow prompts/task.md section 8: publish actual ID lists, per-table counts, rule/readiness state, result and uniquely named receipts through Jira footprints; refresh the parent workspace registry and show the report to the user. Honor an explicitly requested Jira status through offered workflow transitions; status changes do not authorize Git merges, deployments or new database actions.

## Gates

- **human_gate.required:** true
- **before_phase:** `step-5`
- **description:** Every live mutation needs its exact-action authorization; recall an existing valid scoped grant without asking twice. Dry-run, commit and post-commit restore are distinct actions.
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

# RUN — database-correction.invoice-receipt-incident

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
