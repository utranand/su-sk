# Invoice amount — reusable authoring contract

Playbook `database-correction.invoice-amount` **1.1.0** specializes the canonical [invoice incident playbook](../invoice-receipt-correction/README.md) with `operation: amount`. Read its [detailed procedure](../invoice-receipt-correction/prompts/task.md) and [SQL contracts](../invoice-receipt-correction/references/sql-contracts.md), plus both shared PBR/PAR rule files.

The SQL files in `templates/` are **authoring contracts, not executable fixes**. They deliberately contain AUTHORING_TEMPLATE_ONLY barriers. Generate a complete new five-script suite from the current incident's schema/data; independent review must reject any unresolved placeholder or worksheet marker. No prior case UUID/count/amount/authorization is a default.

This specialization accepts only `operation: amount`; route combined or mismatched work to the canonical incident playbook before proceeding. Follow the canonical stage dispatch: `prepare` authors, `dry-run`/`commit` reuse unchanged reviewed evidence, and `verify` performs read-only checks without regenerating or rebinding scripts/snapshots.

For correction, change only verified eligible amount fields and synchronize the complete final QR/barcode satang token atomically. Never use four-character truncation. For cancellation, delete verified unpaid/unprinted invoices with their exclusive QR/barcode rows and mappings; do not assign a guessed cancellation status or assume a cascade. Overdue does not imply paid and future due dates alone do not block. Preserve financial transactions, receipts/bank data and unrelated configuration.

Use [PROMPT.md](PROMPT.md) or supply `card`, with defaults `jira_env: su`, `database_env: smart_utilities_prod`, `stage: prepare`. Source templates do not execute automatically. Current exact-action approval, recoverable transaction and actual captured runtime evidence are mandatory for each later live dry-run, COMMIT or restore. Follow the canonical commit concurrency/quiet-window and uncertain-outcome rules. Reuse the parent workspace, register this playbook/version/stage, and publish exact ID/action lists and real results through footprints.

Validate and preview through the Sidekicks CLI; its `playbook run --dry-run` is a non-executing agent-step preview, not a database dry-run. See generated [RUN.md](RUN.md).
