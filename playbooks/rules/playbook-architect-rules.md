# Playbook Architect Rules — Smart Utilities

Every playbook referencing this file follows all PAR01..07 below. Framework consent, direct-production access and managed-write floors remain binding. These rules distinguish preparation, live rollback-only testing, committed changes and recovery.

## PAR-01 — Zero DDL and immutable full backup

Use DML only for correction/recovery; never create production backup tables or alter enums/schema to invent a cancellation state. Export a consistent full-row offline JSON snapshot of every target, including canceled charge transactions, and relevant preserved relationship/configuration, including nulls, enums, JSON and timestamps. Save timestamp/checksum and prove typed recovery against the actual schema. Keep snapshots private and immutable; refresh live comparison evidence before each action without replacing the original snapshot.

## PAR-02 — Exact guarded atomic transaction

Use one direct rollback-capable transaction for each approved operation, with bounded timeouts, deterministic locks and upfront database/tenant/entity/PK/period/count/before-hash/eligibility/payment/dependency/fiscal/configuration assertions. Assert every UPDATE/DELETE/INSERT count and capture affected IDs/keys and readbacks. On any failed guard/error immediately roll back or close the same session. Never split a transaction across connector calls. Use saved payment references so checks remain valid after invoice/transaction/QR deletion; freeze discovered reference-column inventory and reject schema drift, including relationships without FKs.

## PAR-03 — Actual rollback-default dry-run evidence

All generated correction and standby recovery SQL defaults to ROLLBACK. A live dry-run itself is a gated production mutation. A pass requires actual guarded DML execution, observed per-table counts/IDs/values, literal ROLLBACK completion and a fresh full original-state comparison. Typed SELECT projections, offline models and EXPLAIN without ANALYZE are preparation checks, never runtime success. Independently verify raw receipts and exact source/wrapper hashes. Record NOT RUN honestly until the approved attempt exists.

## PAR-04 — Scoped approval, concurrency and one-attempt execution

Obtain explicit user permission for each exact live dry-run, production COMMIT or later DML restore. Recall an existing valid current scoped grant without asking again. A method rule, registration, checkbox, stage parameter, CLI `--yes`, old case approval or dry-run approval does not authorize a fresh/different commit or restore. Disclose material target/scope/risk changes.

Before COMMIT, assess actual concurrent collection/payment/QR writer paths. Where non-cooperating READ COMMITTED inserts can bypass existing-row locks/FKs, require a real operator-confirmed payer quiet window or a reviewed compatible control. Do not infer quiescence from a momentary lock scan. Wider table locks or worker suspension need their applicable scope/impact authorization. A rollback-only dry-run can omit the quiet window only after independent review verifies the exact transaction has no nontransactional side effects; temporary lock contention still matters.

Freeze the successful dry-run source; make a separate reviewed commit copy changing only its final ROLLBACK to COMMIT. Use a durable attempt marker, one atomic connection, ON_ERROR_STOP and capture every result set/NOTICE/error plus transaction completion. After a timeout/lost response or possible COMMIT, reconcile read-only before any retry; never assume rollback or auto-restore committed data. Bind standby recovery to exact actual committed after-state and guard conflicts/drift: use removed-key absence plus retained-row/settings hashes for deletion, and distinct updated-row after-images where applicable. Immediate rollback on an uncommitted error is part of its approved action, not a new consent gate.

## PAR-05 — Primary-key scope and real audit fields

Every DML statement pins approved PKs and verified tenant/entity ownership; link/QR tables without tenant columns use pinned keys plus scoped ownership evidence. Preserve paid accounting and every non-target financial transaction/configuration record. Complete cancellation includes only the exact exclusively linked unpaid charge transactions proven eligible under PBR06; paid/shared dependencies remain hard stops. Reject duplicate/shared references rather than expanding deletion. Use existing remarks/timestamp columns when present; do not invent audit fields absent from the schema. Append rather than erase original remarks. Capture deletion audit through immutable originals, exact removed IDs/keys, receipts and Jira; deleted rows cannot retain a new remark. Verify a full before/after projection shows only authorized field/row changes.

## PAR-06 — Honest Thai executive report

Produce a concise Thai business report with prominent target UUID, municipality/payer, periods, exact operations/counts and actual preparation/dry-run/commit/recovery state. Keep raw SQL out of this executive summary; technical SQL and runtime receipts belong in clearly labeled separate attachments. Do not prefill a success claim or show a planned count as an observed result.

## PAR-07 — Parent workspace, reviewable ID lists and stage checklists

Reuse the Jira parent feature workspace and its branch/registry; never create independent child workspaces. Resolve artifact anchors and use a supported CLI writer. The standard package contains 01 investigation, 02 backup plus full snapshot, 03 correction, 04 verification, 05 recovery, scope/digests/review, Thai report and incident tracker. Use raw `.sql.txt` when the writer requires `.txt`; do not bypass managed-write boundaries to force an extension.

Register the current playbook ID/version/sources/stage in parent and relevant child descriptions. Preserve reporter/intake content. Use the existing readiness helper and bound playbook identity; evaluate every applicable PBR/PAR requirement through a short nonduplicate checklist. A preparation child can complete without production approval. Unknown evidence stays unmet; nonexistent receipt work can be N/A with reason.

Before execution, publish exact invoice and charge-transaction UUIDs, invoice numbers (or evidenced prior cancellation), periods, proposed field/amount changes and associated QR/mapping effects, with expected remaining transaction IDs/balance. Afterward publish actual per-table counts, affected/removed IDs, ROLLBACK/COMMIT evidence, independent review and fresh verification through sk-jira-footprint. Use unique attempt-specific receipt filenames. Keep private snapshots local. Preserve historical candidate/attempt evidence and update only current trackers/verdicts with later results.

Discover the requested Jira status and offered workflow steps; follow authorized valid transitions instead of assuming there is no route from a missing direct transition. Record actual status changes without claiming a Git merge, deployment or database action. Stop for known wider side effects or unavailable permissions; never rewrite workflow configuration to bypass them.
