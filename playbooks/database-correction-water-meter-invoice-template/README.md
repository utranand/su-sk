# Water meter reading and invoice correction — reusable authoring contract

Playbook `database-correction.water-meter-invoice` **1.0.0** specializes the Smart Utilities database correction procedure for **Water Meter Reading and Invoice Corrections** (`water_fee_transaction` & `water_fee_invoice`). Read the canonical [invoice incident playbook](../invoice-receipt-correction/README.md), [SQL contracts](../invoice-receipt-correction/references/sql-contracts.md), and both shared rule files: [`playbook-business-rules.md`](../rules/playbook-business-rules.md) and [`playbook-architect-rules.md`](../rules/playbook-architect-rules.md).

The SQL files in `templates/` are **authoring contracts, not executable fixes**. They deliberately contain `AUTHORING_TEMPLATE_ONLY` barriers. Generate a complete fresh five-script suite from the current incident's empirical schema and live data; independent review must reject any unresolved placeholder or worksheet marker. No prior case UUID, count, amount, or authorization is a reusable default.

## Domain Specifics: Water Meter Readings & Invoice Correction

In Smart Utilities water fee management (`su-sk`), water services exhibit distinctive chained accounting and reading invariants:

1. **Meter Reading Chain Continuity (Sequential Volume Invariant):**
   - In water billing, meter readings are chained: for any billing round $N$, the starting reading (`previous_volume_usage`) **MUST** equal the preceding round $N-1$'s ending reading (`current_volume_usage`).
   - Total volume consumption is defined as `total_volume_usage = current_volume_usage - previous_volume_usage`.
   - When correcting historical readings due to faulty meters or recording errors:
     - The target month's `current_volume_usage` and `total_volume_usage` are updated.
     - The subsequent target month's `previous_volume_usage` is realigned to match.
     - **In-flight unread subsequent rounds** (e.g. current active month where `current_volume_usage IS NULL`) **MUST** also have their `previous_volume_usage` updated to maintain the unbroken reading chain before the next field reading is logged.
     - Faulty meters are marked with `is_meter_broken = true` and `meter_broken_remark`.

2. **Water Fee Recalculation:**
   - `water_usage_fee`: Calculated according to the customer's water fee schedule / rate tiers from `master_water_usage_fee_rate`.
   - `meter_maintenance_fee`: Standard meter maintenance fee (e.g. 15.00 THB).
   - `before_total_fee` = `water_usage_fee` + `meter_maintenance_fee`.
   - `percent_vat` = 7.0%.
   - `vat_fee` = round(`before_total_fee` * 0.07, 2).
   - `after_total_fee` = `before_total_fee` + `vat_fee`.

3. **Invoice Cancellation & Deletion Scope:**
   - If incorrect invoices have already been issued for the target rounds:
     - Verify invoices in `water_fee_invoice` are **unpaid** (`status = 'invoice_status_4'` / `'invoice_status_1'`), **unprinted** (`printed_date IS NULL`), carry **no electronic receipt** (`receipt_id IS NULL`), and have **no online bank settlement** in `ktb_transaction_id` or `water_fee_ktb_transaction`.
     - In accordance with PBR-06, delete `water_fee_invoice_transaction` links and `water_fee_invoice` rows.
     - **Integrated Garbage Fees Preservation:** In municipalities where water invoices bundle garbage fees (indicated by `water_fee_invoice_transaction.garbage_transaction_id`), the linked garbage transaction must remain intact in `garbage_fee_transaction` so that it can be correctly rebilled.

4. **Multi-Tenant Payer Disambiguation (PBR-04):**
   - Water consumer numbers (`water_no`, e.g. `68002492`) are shared across municipalities.
   - All queries and updates **MUST** resolve the municipality `sid` (e.g. `S0006` for บางหญ้าแพรก) and pin the unique entity UUID (`water_id`).

5. **Historical Paid Periods Immutability (PBR-01):**
   - Preceding rounds with issued receipts (e.g. July/month 7 with `status = 'invoice_status_3'`) must remain strictly untouched and read-only.

## Lifecycle & Stage Dispatch

Follow the canonical stage dispatch:
- **`prepare`**: Authors the fresh 5-script suite, `scope.json`, backup JSON snapshot, and Thai executive report draft.
- **`dry-run`**: Executes the reviewed `03_correct.sql` in a rollback-only transaction (`ROLLBACK;`) to preview in-transaction assertions and readbacks without changing data.
- **`commit`**: With separate, explicit human authorization and operator quiet-window confirmation, executes `03_correct.sql` ending in `COMMIT;`, capturing raw receipts.
- **`verify`**: Read-only verification from a fresh connection proving that target invoices are deleted, meter reading sequences are continuous, and volumes/fees match approved figures.

## Quick Start

Use [PROMPT.md](PROMPT.md) or pass inputs:
```text
card: SU-10768
jira_env: su
database_env: smart_utilities_prod
stage: prepare
operation: water-meter-invoice
```
Validate with:
```bash
node bin/sidekicks playbook validate database-correction.water-meter-invoice --project su-sk
```
