# Water meter reading volume correction — reusable authoring contract

Playbook `database-correction.water-meter-reading` **1.0.0** specializes the Smart Utilities database correction procedure for **Water Meter Reading Volume Corrections** (`water_fee_transaction`). Read the canonical [invoice incident playbook](../invoice-receipt-correction/README.md), [SQL contracts](../invoice-receipt-correction/references/sql-contracts.md), and both shared rule files: [`playbook-business-rules.md`](../rules/playbook-business-rules.md) and [`playbook-architect-rules.md`](../rules/playbook-architect-rules.md).

The SQL files in `templates/` are **authoring contracts, not executable fixes**. They deliberately contain `AUTHORING_TEMPLATE_ONLY` barriers. Generate a complete fresh five-script suite from the current incident's empirical schema and live data; independent review must reject any unresolved placeholder or worksheet marker. No prior case UUID, count, amount, or authorization is a reusable default.

## Domain Specifics: Meter Reading Chain & Volume Corrections

In Smart Utilities water fee management (`su-sk`), water meter readings follow a chained sequence invariant:

1. **Meter Reading Chain Continuity (Sequential Volume Invariant):**
   - For any round $N$, the starting reading (`previous_volume_usage`) **MUST** match round $N-1$'s ending reading (`current_volume_usage`).
   - Total volume consumption is `total_volume_usage = current_volume_usage - previous_volume_usage`.
   - When correcting meter readings due to faulty meters or reading errors:
     - The target month's `current_volume_usage` and `total_volume_usage` are updated.
     - The subsequent target month's `previous_volume_usage` is updated to match.
     - **In-flight unread rounds** (e.g. subsequent months where `current_volume_usage IS NULL`) **MUST** have their `previous_volume_usage` updated to maintain the unbroken reading chain.
     - Mark `is_meter_broken = true` and describe the failure in `meter_broken_remark`.

2. **Water Fee Recalculation:**
   - `water_usage_fee`: Computed according to volume tiers from `master_water_usage_fee_rate`.
   - `meter_maintenance_fee`: Standard maintenance fee (e.g. 15.00 THB).
   - `before_total_fee` = `water_usage_fee` + `meter_maintenance_fee`.
   - `percent_vat` = 7.0%.
   - `vat_fee` = round(`before_total_fee` * 0.07, 2).
   - `after_total_fee` = `before_total_fee` + `vat_fee`.

3. **Multi-Tenant Payer Disambiguation (PBR-04):**
   - Payer number `water_no` is shared across municipalities. Always resolve `sid` and `water_id`.

4. **Historical Paid Periods Immutability (PBR-01):**
   - Historical rounds with issued receipts (e.g. July/month 7) remain strictly immutable.

## Lifecycle & Stage Dispatch

Follow the canonical stage dispatch:
- **`prepare`**: Authors the fresh 5-script suite, `scope.json`, backup JSON snapshot, and Thai executive report draft.
- **`dry-run`**: Executes the reviewed `03_correct.sql` in a rollback-only transaction (`ROLLBACK;`).
- **`commit`**: With separate, explicit human authorization, executes `03_correct.sql` ending in `COMMIT;`.
- **`verify`**: Read-only verification on a fresh connection proving that meter reading volumes, fees, and sequence continuity match approved figures.

## Quick Start

Use [PROMPT.md](PROMPT.md) or pass inputs:
```text
card: SU-10768
jira_env: su
database_env: smart_utilities_prod
stage: prepare
operation: water-meter-reading
```
Validate with:
```bash
node bin/sidekicks playbook validate database-correction.water-meter-reading --project su-sk
```
