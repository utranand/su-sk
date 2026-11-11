# Water invoice removal and recreation — reusable authoring contract

Playbook `database-correction.water-invoice-recreation` **1.0.0** specializes the Smart Utilities database correction procedure for **Removing and Recreating Water Invoices** (`water_fee_invoice` & `water_fee_invoice_transaction`). Read the canonical [invoice incident playbook](../invoice-receipt-correction/README.md), [SQL contracts](../invoice-receipt-correction/references/sql-contracts.md), and both shared rule files: [`playbook-business-rules.md`](../rules/playbook-business-rules.md) and [`playbook-architect-rules.md`](../rules/playbook-architect-rules.md).

The SQL files in `templates/` are **authoring contracts, not executable fixes**. They deliberately contain `AUTHORING_TEMPLATE_ONLY` barriers. Generate a complete fresh five-script suite from the current incident's empirical schema and live data; independent review must reject any unresolved placeholder or worksheet marker. No prior case UUID, count, amount, or authorization is a reusable default.

## Domain Specifics: Invoice Removal & Recreation Invariants

In Smart Utilities water fee management (`su-sk`), removing and recreating invoices requires strict multi-table coordination:

1. **Pre-condition: Volume Corrections Complete:**
   - Before removing and recreating invoices, the underlying water consumption volumes and fees in `water_fee_transaction` **MUST** already be corrected and verified using `database-correction.water-meter-reading`.

2. **Unpaid & Unprinted Invoice Pre-condition (PBR-01, PBR-06):**
   - The candidate invoices for removal in `water_fee_invoice` **MUST** be:
     - Unpaid (`status = 'invoice_status_1'` or `'invoice_status_4'`).
     - Unprinted (`printed_date IS NULL`).
     - Unaccompanied by electronic receipts (`receipt_id IS NULL`, `payment_date IS NULL`).
     - Free of online bank settlement in `ktb_transaction_id` or `water_fee_ktb_transaction`.
   - If any candidate invoice was paid or printed, financial records are immutable (PBR-01) and execution must **HALT**.

3. **Guarded Removal of Old Invoices:**
   - In accordance with PBR-06, deletion must occur in exact dependency order:
     1. Delete links in `water_fee_invoice_transaction` matching target `invoice_id`.
     2. Delete invoice records in `water_fee_invoice` matching target `invoice_id` and `water_id`.

4. **Recreation of Fresh Water Invoices:**
   - Generate new `water_fee_invoice` rows with:
     - New `invoice_id` (`gen_random_uuid()`).
     - Formatted `invoice_no` conforming to municipality round sequencing.
     - Synchronized references (`ref_1`, `ref_2`) derived from `sid`, fiscal year, round month, and invoice number.
     - Synchronized PromptPay QR (`qr_code_text`) and barcode (`barcode_text`) with amount in satang (PBR-05) matching total fee (including integrated garbage fees).
     - Standard payment due date and status `'invoice_status_1'` (unpaid).
   - Insert new mapping records into `water_fee_invoice_transaction` linking new `invoice_id` to:
     - Target `water_fee_transaction` row.
     - Target integrated `garbage_fee_transaction` row (`garbage_transaction_id`) to preserve integrated municipality billing.

5. **Historical Paid Invoices Immutability (PBR-01):**
   - Preceding rounds (e.g. July/month 7) that are already paid with issued receipts remain 100% untouched.

## Lifecycle & Stage Dispatch

Follow the canonical stage dispatch:
- **`prepare`**: Authors the fresh 5-script suite, `scope.json`, backup JSON snapshot, and Thai executive report draft.
- **`dry-run`**: Executes the reviewed `03_correct.sql` in a rollback-only transaction (`ROLLBACK;`).
- **`commit`**: With separate, explicit human authorization, executes `03_correct.sql` ending in `COMMIT;`.
- **`verify`**: Read-only verification on a fresh connection proving that old invoices are deleted, new invoices are created with valid amounts, QR payloads are synchronized, and transaction links are intact.

## Quick Start

Use [PROMPT.md](PROMPT.md) or pass inputs:
```text
card: SU-10768
jira_env: su
database_env: smart_utilities_prod
stage: prepare
operation: water-invoice-recreation
```
Validate with:
```bash
node bin/sidekicks playbook validate database-correction.water-invoice-recreation --project su-sk
```
