# Utility invoice correction and cancellation

Reusable playbook `database-correction.invoice-receipt-incident` **1.1.0** for `su-sk`.

Start with [RUN.md](RUN.md), use the [invocation prompt](PROMPT.md), and follow the detailed [procedure](prompts/task.md). The [SQL contracts](references/sql-contracts.md) define the incident-specific scripts. Both shared rule files in [the rules catalog](../PLAYBOOK_RULES.md) are binding.

## Outcome

Prepare or execute exactly the operation requested for the current municipality and payer: correct unpaid/unprinted invoice amounts; cancel verified unpaid invoices with their exclusive QR/barcode rows and mappings; investigate receipts without inventing nonexistent receipt work. Preserve paid accounting, financial transactions and unrelated configuration.

Preparation is the default. A folder invocation, stage parameter, checkbox, old incident approval or CLI `--yes` never grants a fresh production mutation. Recall a valid current scoped user grant; ask only for missing authority, a material scope change or a genuine operational fact.

## Inputs and stages

| Input | Default | Meaning |
|---|---|---|
| `card` | required | Incident or subtask; inherit the parent workspace |
| `jira_env` | `su` | Registered Jira alias |
| `database_env` | `smart_utilities_prod` | Direct target; reverify before connecting |
| `stage` | `prepare` | `prepare`, `dry-run`, `commit`, `verify` |
| `operation` | `from-card` | `from-card`, `amount`, `cancellation`, `combined` |
| `work_item` | derive/reuse | Preserve supplied parent/run anchors |
| `user_guide` | empty | Bounded run adjustments, never authority |

`prepare` produces the five scripts, complete offline snapshot, independent review and short readiness checklist. `dry-run` requires exact authorization, performs real DML and proves ROLLBACK plus original-state restoration. `commit` requires a matching passed dry-run and separate authorization; verify afterward and bind recovery to committed hashes. `verify` reuses existing scope, original snapshots, matching verifiers and receipts; it is read-only and never regenerates/rebinds correction or recovery scripts or repairs data. Execution stages reuse unchanged reviewed candidates; drift returns to preparation/review.

## Operational contract

- Resolve municipality code to actual system ID and unique payer UUID; invoice numbers and payer numbers are not globally unique.
- Amount correction: retain status/receipts/fiscal years, synchronize invoice label, transaction fee and complete final QR/barcode satang token atomically. `150` baht is `15000` satang.
- Cancellation: delete only verified unpaid/unprinted invoices and exclusive QR/barcode rows/mappings. Overdue is not automatically paid. Future due dates alone do not block. No guessed cancellation enum, implicit cascade or financial-transaction deletion.
- Use saved original references for bank checks even after QR deletion. Reject shared QR rows or duplicate references rather than expanding deletion.
- Never split a transaction across connector calls. Use a reviewed direct atomic route with all result sets and NOTICE/error capture; fail/close immediately on error. Reconcile read-only after a lost COMMIT response.
- For demonstrated commit races, establish a real payer quiet window or reviewed compatible control. A rollback-only dry-run can proceed without one only when independent review verifies rollback safety and the absence of nontransactional side effects; disclose temporary locking.
- Post exact invoice UUIDs, numbers, periods and actions before execution, then actual outcomes, counts and unique attempt receipts. Mark NOT RUN truthfully until the actual run exists.

## Files and evidence

Generate `01_investigate_readonly.sql`, `02_backup.sql`, `03_correct.sql`, `04_verify_readonly.sql`, `05_rollback.sql`, a complete immutable JSON snapshot, `scope.json`, review/digest records, a Thai executive report and an incident tracker. Use raw SQL `.sql.txt` if the supported artifact writer requires `.txt`; clients accept the file directly. Provide matching verifier/restore for each alternative operation and do not mix alternatives from the same original baseline sequentially.

Incident execution preserves reusable source. Generated files belong to the parent Jira workspace through a supported CLI writer. Non-Jira/default runs use `playbook run-base`; an override needs its own permitted writer. Private snapshots may include personal data; attach concise reports and receipts rather than bulk snapshots. See [report](templates/report.md.template) and [checklist](templates/checklist.md.template) templates.

## Validation without production execution

```bash
node bin/sidekicks playbook validate database-correction.invoice-receipt-incident --project su-sk
node bin/sidekicks playbook run database-correction.invoice-receipt-incident --project su-sk --input card=EXAMPLE-1 --dry-run
```

The CLI preview plans agent steps; it is not a database dry-run. Never infer mutation success from preview, EXPLAIN without ANALYZE, a typed SELECT projection or an offline model.
