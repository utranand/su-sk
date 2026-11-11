# Incident execution procedure

Use this guide with the current manifest inputs. Read both shared PBR/PAR rule files and [SQL contracts](../references/sql-contracts.md). These are instructions for the current incident, not authority to reuse prior case data or production consent.

## 1. Bind intent, target and workspace

Read the Jira card, attachments, relevant comments, existing playbook registration and parent. Extract municipality, payer, utility, requested periods/amounts and exact operations. Clarify conflicting reporter messages using the final evidenced instruction. Preserve manual receipt context; do not treat it as proof that a particular system invoice was paid.

Confirm `stage` and `operation` are supported; otherwise stop before effects. Default to preparation. Select `su-sk`, verify the registered direct database alias and forbid Teleport/tunnels under the current project floor. Keep credentials in the configuration resolver/process environment, never logs.

Dispatch by stage before authoring or execution:

- `prepare`: ground the current request and author/review the candidate, immutable original snapshot and matching verifiers/recovery. Finish at the preparation report unless a separately requested and authorized later action is in scope.
- `dry-run`: load the unchanged reviewed candidate, original snapshot, matching verifiers and exact-action grant. Refresh live guard evidence without replacing original files or rebasing the snapshot. Execute only the approved rollback-only attempt; stop for renewed preparation/review if the candidate or baseline has drifted.
- `commit`: load the unchanged reviewed candidate and original snapshot, matching passed dry-run, verifiers and separate commit grant. Refresh guards without overwriting the original evidence. Create only the reviewed final-terminator execution copy; after verified COMMIT bind a new standby recovery copy as section 7 specifies.
- `verify`: load the existing exact scope, original snapshot, appropriate original-state or committed-state verifier, candidate hashes and attempt receipts. Perform authorized read-only checks and reporting only. Never author, regenerate, rebase or rebind 03/05, replace snapshots, request mutation approval or perform a repair. Missing evidence is an unmet check, not permission to reconstruct a new baseline.

For stages other than `prepare`, sections 2–4 describe evidence to inspect/refresh, not instructions to regenerate the package. Preserve immutable prior candidates and receipts. If preparation changes are needed, stop the current execution/verification stage and report what must be prepared and reviewed.

Use `jira workspace preflight`. For a standalone parent incident without a workspace, invoke the Jira workspace workflow to initialize, register and bind its parent feature workspace before generating artifacts. If a subtask has no resolvable parent workspace/information, stop and report that missing parent binding; never create an independent child workspace or fall back to a playbook run. A subtask inherits its parent's workspace, branch and registry; never create a second feature workspace. Reuse supplied `work_item`, run and document anchors. Register the chosen playbook ID, version, sources and applicable stage on the parent and relevant subtasks. Preserve reporter text and existing intake answers. A playbook version/rule/input change invalidates affected readiness evidence; do not transfer a verdict without revalidation.

For the normal Jira case, use the parent `jira-workspace` override with `artifacts write sk-jira-workspace <feature-work-item> outputs/...` or `evidence/...`. Only for an explicitly non-Jira invocation without the Jira workspace contract, resolve `playbook run-base` and use `playbook write-output`. Verify supported writer arguments and returned paths. No direct managed-path writes; do not change framework code just to obtain a preferred extension.

## 2. Ground in live records early

After scope resolution, query actual incident records, not just connectivity/schema. Confine every financial lookup to the municipality/system/entity or its already pinned PK/reference set. Resolve municipality code through the actual system mapping: municipality code is not automatically `sid`. Verify unique payer/entity within that municipality; do not inspect similarly numbered payers elsewhere.

Capture exact invoice UUID, invoice number, period, tax year, status, printing, receipt ID, corresponding charge-transaction/QR/mapping IDs and expected per-table counts. Invoice removal must also remove its exclusively linked unpaid charge transactions under PBR06; zero invoice rows alone is not a NO-OP. Trace owner collection/remaining balance and cron/backfill/next-generation pointers; never run a job or alter settings merely to repair a charge. Discover real enum meanings, columns, inbound dependencies, triggers, rewrite rules, RLS, privileges and available audit fields. Include named/application reference columns without FKs, integrated-water invoice/receipt links and overdue-notice links; freeze the inventory and fail on drift. Do not guess water tables, cancellation statuses or nonexistent bank fields.

Check actual receipt-transaction relationships and collected/in-flight payment evidence by saved payment references plus tenant. In the previously verified garbage schema KTB uses quoted `"SID"` and `"Ref_1"`; reverify the current schema and never invent a status column. Save reference values so later checks survive invoice/QR deletion. Zero system receipts means existing receipt updates are N/A, not a missing task. Zero database matches are scoped observations, not proof about external saved QR images.

For amount changes enforce PBR01's unpaid/unprinted eligible state. For cancellation verify the utility's status meanings; verified garbage status1 and overdue status4 can qualify under PBR06 when unpaid/unprinted checks pass. Overdue does not imply paid. Future due dates alone do not block cancellation. Shared QR, duplicate reference, paid/printed or contradictory accounting evidence is a real stop.

Verify discrete garbage versus accumulated water billing, per-row fiscal formula and no cross-year shifting. If future fee settings already match the intended rate, preserve them. Inspect integrations and side effects rather than broadening scope to linked services automatically.

## 3. Prepare exact SQL and recovery

Follow the five [SQL contracts](../references/sql-contracts.md) and the amount/cancellation authoring contracts in the catalog. These source templates are non-executable worksheets; generate complete incident-specific SQL and reject any remaining AUTHORING_TEMPLATE_ONLY marker or unresolved placeholder.

Export one consistent full-row before-state with timestamp/checksum: invoices, transactions, QR rows, mappings, detail/fee settings and relevant payment relationships. Preserve nulls, enums, JSON and timestamps. Capture metadata/source evidence and exact per-table ID sets in `scope.json`. Keep snapshots private and immutable; recheck live rows match them before each authorized action.

Author a zero-DDL atomic transaction with exact PK + tenant/entity + period predicates, deterministic row locking, full before-state drift hashes, eligibility/payment/dependency/fiscal/configuration guards and exact per-table DML row-count assertions. Missing/extra/changed records abort the whole action. Use existing audit fields; QR rows or some transaction tables may lack remarks/updated_date.

For amount changes synchronize invoice amount label, linked transaction fee and the complete delimited final QR/barcode satang token in the same transaction. Preserve references and every other payload field; calculate satang from the verified monetary representation. Never compare/truncate to the last four characters.

For cancellation remove approved invoices, corresponding exclusively linked unpaid charge transactions, their exclusive mappings and QR rows containing QR/barcode payloads in the observed dependency order. Preserve paid/shared/unrelated transactions, receipts, bank and configuration records; any non-target dependent charge reference is a hard stop, not permission to remove accounting links. For a prior invoice-only cancellation, use immutable original invoice/transaction/reference evidence and re-ground current dependencies/payments before preparing a separately authorized transaction-only follow-up. Do not recreate earlier canceled rows in that follow-up recovery. Check both QR UUID ownership and saved reference uniqueness. Do not assume a cascade. Capture deleted IDs and mapping keys using RETURNING/NOTICE or equivalent evidence.

Prepare complete DML recovery: restore all original charge-transaction, QR, mapping and invoice rows deleted by this action in the actual dependency order, plus original amount values; reject PK, invoice number/period/reference conflicts, changed relationships and intervening payments. Do not reconstruct history, clear original remarks/timestamps or use conflict-overwrite. Keep recovery rollback-default. Require exact committed after-images for later amount recovery; placeholders must fail closed.

Independently review the concrete SQL, backup recoverability, exact operation sets, record predicates, execution wrapper and checksums before live execution. Reuse the passing review only for unchanged guards/DML; review any material change. EXPLAIN without ANALYZE and typed backup SELECTs are preparation checks, not DML execution or live recovery tests.

If authoring alternatives from one baseline, give each its own after-state verifier and matching restore. Do not execute them sequentially; regenerate against the actual new baseline. A combined operation must have one coherent five-script suite rather than an amount-only package presented as full incident coverage.

## 4. Keep readiness and footprints useful

The registered playbook is the primary readiness basis. Evaluate every applicable PBR/PAR requirement and map it into roughly 4–8 distinct card-specific checks; add only real incident blockers. Distinguish ready-to-prepare, reviewed preparation, authorized dry-run and authorized commit. A child validation/SQL/review stage does not need production approval to be complete.

Use [checklist](../templates/checklist.md.template) and [report](../templates/report.md.template). Preserve machine readiness/playbook binding metadata through its existing helper and preserve reporter content. Represent unknown evidence as unmet and irrelevant work as N/A. Do not add duplicate generic intake or repeat already settled scope questions.

Before execution post a readable list containing period, invoice number or original cancellation evidence, invoice UUID, charge-transaction UUID, exact proposed action and original→target amount where applicable, grouped into correction and cancellation. Include linked QR/mapping effects and preserved financial records. Use interactive Jira task lists in the description; technical prose belongs in an attached concise report. Approval checkboxes must not be ticked by the agent without verifiable scoped user authorization.

Always use `sk-jira-footprint` for incident progress and lifecycle reporting. Post reviewed correction/recovery SQL as labeled technical attachments or syntax-highlighted material. Keep executive Thai reports free of raw SQL. Use unique dry-run/commit receipt filenames so users can distinguish attempts. Keep prior comments/attachments as historical evidence; add a new corrective footprint rather than rewriting someone else's evidence.

## 5. Run an authorized live dry-run

If `stage=prepare`, finish preparation and report; do not execute DML. For `stage=dry-run`, obtain or recall explicit approval for the exact current target, operation sets, temporary updates/deletions and ROLLBACK recovery. A request to show a result, format a checklist or select a stage is not such approval. Never ask twice for an existing valid current scoped grant; disclose any newly discovered material risk or scope change.

Freeze source, wrapper and snapshot digests. Use one reviewed direct atomic connection, such as psql with `-X --set=ON_ERROR_STOP=on`, bounded lock/statement/idle timeouts, and credentials resolved without printing them. Preserve the reviewed mutation bytes; wrapper additions must be reviewed read-only session guards/output markers/verification. Do not split BEGIN/DML/ROLLBACK across `pg_query.py` calls. A runner's generic OK line is insufficient if it discards result sets or notices.

Record actual per-table UPDATE/DELETE counts, exact IDs/mapping keys, in-transaction values, full guards and the literal ROLLBACK completion marker. Counts inside a DO block are supported by successfully completed exact row-count assertions plus readback/NOTICE; distinguish this from merely seeing a plan or an expected-count constant.

Immediately verify complete original-state restoration in the same session and a separate fresh read-only connection; compare full snapshot sections except observation time. Independently check raw receipts and candidate hashes. On an error, roll back/close immediately and stop; do not automatically retry a mutation.

A quiet window is not required for rollback correctness only when independent review verifies the exact rollback-only transaction has no nontransactional side effects, external function calls, triggers, rewrite rules, sequence effects or similar risks. Temporary locks can still affect competing writers and must be disclosed. Do not extend that assessment to COMMIT.

Until an actual approved run exists, show **NOT RUN**, with preparation checks separately labeled. A passing dry-run proves rollback behavior; it does not commit the requested correction.

## 6. Commit the separately approved operation

Require the matching passed live dry-run, unchanged scope/guards and current explicit approval to COMMIT. Record user provenance and exact target/operation/digests. A valid approval persists for its named action; no duplicate consent prompt. Dry-run approval, a method rule or approval from a different incident is not commit approval.

Assess concurrent collectors, payment ingestion and QR generation using actual application write paths and current schema. SERIALIZABLE plus row locks alone cannot prevent a non-cooperating READ COMMITTED writer from inserting accounting/mappings without FKs. When this race applies, establish a real payer activity quiet window confirmed by the operator, or prepare a compatible reviewed control. A momentarily empty pg_locks result is not a quiet-window guarantee. Do not silently suspend production workers or acquire broader write-blocking table locks; disclose/authorize any material wider impact.

Refresh the full before-state immediately before execution. Create a checksum-recorded execution copy changing only the final ROLLBACK to COMMIT; do not edit the reviewed source in place. Review wrapper/transaction differences. Use a durable one-attempt marker; never rerun after a success, timeout or lost connection without read-only reconciliation.

Capture actual COMMIT completion and all runtime counts/readbacks/removed IDs. Run the matching 04 verifier in the same session and a separate fresh connection. Compare complete before/after rows to the exact approved projection, including unchanged detail/future settings, retained transactions, references and fiscal years. Verify approved canceled transaction IDs are absent from owner collection and the remaining balance matches the projection; invoice/QR absence alone is not completion. Missing/extra changes are a failure, not an excuse to execute an unapproved repair.

If the response is lost or verification fails after COMMIT may have occurred, record outcome unknown or committed-but-unverified. Reconcile read-only; never assume ROLLBACK and never automatically retry or execute DML recovery. Once verified, tell the operator the commit window is complete.

## 7. Bind standby recovery

Bind recovery to the actual committed state for the exact operation. Amount recovery needs hashes for the expected distinct updated invoice IDs and applicable transaction/QR rows. Cancellation recovery needs verified absence of the exact removed invoice/charge/QR/mapping keys plus full retained-row/settings state; deleted rows have no after-image. A transaction-only follow-up uses removed-charge absence and retained-state hashes and does not require a nonexistent canceled invoice after-image. Save the complete actual scoped after-state/digest and operation-specific keys/hashes in a separately checksum-recorded standby05 binding. Never use dry-run hashes, guessed timestamps, incomplete/duplicate ID sets or null placeholders as committed recovery evidence.

Independently verify only the after-image binding changed and that all original-value/PK/conflict/payment guards remain. Default to ROLLBACK. Preserve full original charge-transaction, QR, mapping and invoice rows removed by this action, nulls, enum/JSON values and timestamps; restore amount fields/remarks exactly. Keep recovery prepared but unexecuted unless a fresh exact-action live restore is authorized. Automatic rollback of a failed uncommitted transaction is already part of its approval.

## 8. Close the evidence loop

Post actual invoice and charge-transaction UUID/number/period/action lists, per-table counts and retained collection/balance results; distinguish planned, dry-run-rolled-back, committed and restored outcomes. Attach concise reports and uniquely named stdout/stderr receipts, retaining private snapshots locally. Update the primary readiness state, visible checklist, child stage outcomes, incident tracker, target database registry/status and current report through supported CLI writers. Keep immutable candidate/attempt evidence and source hashes.

On an explicit request for a Jira destination such as Ready to Merge, inspect project statuses and offered transitions. Follow valid workflow steps to the requested destination when authorized; do not conclude a status is unreachable just because there is no direct transition from the current state. If a transition has known deployment/other effects beyond the grant, obtain the applicable authorization. Never force unavailable transitions or alter workflow configuration to bypass permissions. A Jira status change is not an instruction to merge Git or deploy code.

Finally show the report and exact result to the user. Report partial/failed/unknown results and genuine remaining work. Do not call the incident complete until the requested database outcome is committed and independently verified, or the user explicitly requested only a preparation/verification stage.
