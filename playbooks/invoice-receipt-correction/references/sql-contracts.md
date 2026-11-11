# SQL authoring contracts

Generate fresh incident-specific SQL from empirical schema/data. No UUID, payer, amount, count, enum meaning or approval from a previous incident is a reusable default. Source `.sql.template` files are authoring worksheets with a deliberate fail-closed marker; they are never production execution files.

## Required package

| File | Contract |
|---|---|
| `01_investigate_readonly.sql` | Scoped actual row identities, tenant mapping, eligibility, payments, printing, billing/fiscal model, QR references, dependencies and configuration; pure reads |
| `02_backup.sql` | One consistent full-row JSON snapshot for affected and preserved relationships/settings; offline timestamp/checksum, no backup-table DDL |
| `03_correct.sql` | Exact guarded DML in one transaction ending ROLLBACK, with in-transaction readback and affected-ID/count evidence |
| `04_verify_readonly.sql` | Exact target after-state plus unaffected full-row/configuration checks; saved bank references remain checkable after QR deletion |
| `05_rollback.sql` | Complete original-value DML recovery, conflict/drift/payment guards, exact committed after-image binding where required; defaults ROLLBACK |

Add `scope.json`, backup JSON, checksum manifest, independent review, Thai report and incident tracker. When the writer allows only text extensions, preserve raw SQL in `.sql.txt` and record the map. Every alternative operation needs a matching 04/05; do not mix alternatives based on one original snapshot.

## Before-state and metadata

Resolve named municipality → actual system ID → unique payer entity, then pin invoice PKs and the corresponding charge-transaction/QR/mapping keys. Cancellation must remove the approved collectible charge rows too; an already-absent invoice is not a NO-OP while charges remain. For a follow-up, prove the original invoice/transaction/reference association from immutable evidence and re-ground current rows; do not guess it. Preserve periods, tax years and bill references. Inspect enum labels/actual application meanings, inbound FKs/cascades, user triggers, rewrite rules, RLS, privileges and existing audit columns. Assess external/sequence/function side effects and concurrent write paths before selecting execution safeguards. Discover named/FK/application reference columns, including integrated-water and overdue relationships without FKs; freeze the inventory and reject changes. Trace collection/balance and cron/backfill/next-round paths for retention, repricing or recreation risk; do not infer a job run from created_by or change jobs/settings without authorization.

Verify receipts and receipt-transaction links as well as bank evidence. In the previously verified garbage schema KTB columns include quoted `"SID"` and `"Ref_1"`; there is no bank status field to filter on. Adapt only from the current schema. Save original reference values in the scope so lookup is independent of live invoice/QR joins after deletion.

Export complete original rows, including nulls, JSON, enums and timestamps. Capture row hashes in a consistent timezone/representation; use full before hashes and stable-field after hashes where a timestamp is deliberately changed. Confirm the snapshot still matches before execution. Keep full snapshots out of public attachments.

## 03 guard and DML order

1. BEGIN with an isolation/locking design appropriate to the real writer paths; bounded lock/statement/idle timeouts and deterministic row locks. SERIALIZABLE alone is not proof that other writers cannot insert related rows.
2. Assert database, real tenant/entity, exact ID sets and counts, configuration, schema/dependency assumptions, original full row state, fiscal years and unpaid/unprinted/payment conditions. Fail before any DML on discrepancy. Validate QR exclusivity by mapping keys and all invoice associations; require unique saved references, since PK uniqueness does not imply reference uniqueness.
3. Amount operation: update only pinned eligible invoice label/audit fields, transaction fee/audit fields and full final QR/barcode amount tokens atomically. Append existing remarks; do not invent absent audit columns. Use exact old values and expected row counts.
4. Cancellation operation: delete approved invoice rows, corresponding exclusively linked unpaid charge transactions, exclusive mapping rows and QR/barcode rows in the actual dependency order. No guessed void status. Halt on paid/printed or non-target/shared dependencies; preserve all paid/unrelated transactions, receipts/bank items and payer/settings. If invoices/QRs/mappings were already canceled, require original linkage/reference evidence and new current guards/approval for a transaction-only cleanup; assert zero expected changes to already-absent tables. Use full before-row comparisons, exact period/tax-year predicates and explicit PK ownership checks, including scoped transaction joins for QR/link tables lacking tenant columns.
5. GET DIAGNOSTICS/RETURNING or equivalent must assert each observed table count, not just a final total. Capture actual removed invoice IDs, charge transaction IDs, QR IDs and mapping composite keys, with exact separate counts (including evidenced zero-count tables). Mark deleted-row audit evidence externally through immutable backup, receipt and Jira rather than adding nonexistent remarks to a removed row.
6. Assert the complete target after-state and preserved rows/settings inside the transaction, then emit readbacks. End with ROLLBACK; independent source review and actual dry-run approval precede execution.

## Amount token correctness

Determine payload structure from live rows/source. For a delimited garbage BillPayment payload, replace the entire final `%0A<satang>` / `+<satang>` token and preserve every preceding token/reference. Verify integer satang equals the approved monetary amount multiplied by 100 using the actual currency representation. A 150-baht target is 15000 satang, not its last four characters. Mixed payload layouts must halt or have a separately reviewed transformation.

## Cancellation verification

Verify exact absence of cancelled invoice IDs, corresponding charge transaction UUIDs, QR UUIDs, mapping keys and all saved cancellation payment references inside the approved mutation transaction. For a dry-run this is a temporary preview: after literalROLLBACK require the complete original IDs/rows/balance restored on a fresh connection. Only after COMMIT require target absence and the intended remaining balance to persist on a fresh connection. Exercise or faithfully reproduce the owner-collection/balance query: no canceled charge may remain collectible, and retained IDs/amounts/total balance must match the approved projection. A second QR row reusing a reference must halt before deletion; never delete an unexpected duplicate automatically. Check bank evidence using immutable saved references plus tenant, even when QR rows are gone. Preserve all unrelated and paid financial records; only the exact eligible cancellation charge rows are removed. Do not declare completion from invoice/QR absence alone. Future due dates alone do not block; database removal does not prove an external bank rejected a previously saved image.

## 05 recovery and execution receipts

Restore complete original charge-transaction rows, QR rows, mappings and invoices removed by this action in the captured dependency order using explicit column lists or an empirically checked composite-record representation. Assert types/columns still match. Reject existing PKs, same-period/invoice-number conflicts, reused references, changed relationships and new receipts/bank items. No ON CONFLICT overwrite and no fabricated original values. A transaction-only follow-up restores only its own deleted transactions; never recreate invoices/QRs removed by an earlier committed action. Verify the exact full original transaction rows and preserved state after restore.

For deletion recovery bind the full actual committed after-state, exact removed-key absence and retained-row/settings hashes; deleted rows have no after-image. A transaction-only follow-up must not require invoice after-images or resurrect earlier canceled rows. For amount recovery bind exactly the expected distinct actual committed invoice IDs and full after hashes, including the actual changed timestamp. Compare transaction/QR stable fields and approved changed values as well. Require complete, non-null, duplicate-free bindings; use no dry-run hashes. Standby generation is not live restore authorization.

Use one direct atomic psql session with ON_ERROR_STOP (or an equally reviewed route capturing every result/notice). Do not split a multi-statement transaction across single-call connectors. Freeze source/assembly/wrapper hashes, action approval, target and one-attempt marker. Capture exit status, actual row-count assertions, IDs, values and literal ROLLBACK/COMMIT marker. After a possible COMMIT, failure/timeout is an uncertain outcome until reconciled read-only; never infer rollback or retry automatically.

A dry-run passes only when runtime guards/readbacks/ROLLBACK and a fresh original-state comparison pass. A commit passes only when actual COMMIT and both same-session and fresh after-state checks pass, with a full-row projection showing only authorized changes. Independent raw-receipt verification is required before claiming success.
