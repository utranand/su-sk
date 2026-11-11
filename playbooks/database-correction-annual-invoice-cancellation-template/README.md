# Annual invoice cancellation — reusable authoring contract

Playbook `database-correction.annual-invoice-cancellation` **1.0.0** specializes the Smart Utilities cancellation procedure for **Annual Payment invoices** (`recurring_type_4` / ชำระแบบรายปี). Read the canonical [invoice incident playbook](../invoice-receipt-correction/README.md), [SQL contracts](../invoice-receipt-correction/references/sql-contracts.md), and both shared rule files: [`playbook-business-rules.md`](../rules/playbook-business-rules.md) and [`playbook-architect-rules.md`](../rules/playbook-architect-rules.md).

The SQL files in `templates/` are **authoring contracts, not executable fixes**. They deliberately contain `AUTHORING_TEMPLATE_ONLY` barriers. Generate a complete fresh five-script suite from the current incident's empirical schema and live data; independent review must reject any unresolved placeholder or worksheet marker. No prior case UUID, count, amount, or authorization is a reusable default.

## Domain Specifics: Annual Payment vs Monthly Payment

In Smart Utilities garbage fee management (`su-sk`), annual payment introduces distinct domain relationships:

1. **Recurring Type & Frequency (`recurring_type_4`):**
   - Annual billing is configured as `recurring_type = 'recurring_type_4'` in `garbage_fee_detail`.
   - The citizen receives **one annual bill** (`garbage_fee_invoice`) covering the **entire fiscal year** (`tax_year`, Oct 1 – Sep 30).

2. **Multiple Linked Transactions per Single Annual Invoice:**
   - In monthly billing (`recurring_type_1`), an invoice typically has 1 transaction (or a few overdue months).
   - In annual billing, cron generates up to **12 monthly transaction rows** in `garbage_fee_transaction` for the fiscal year (months 10, 11, 12 of prior calendar year, and months 1 to 9 of current calendar year).
   - All 12 monthly fee transactions are bundled under **one PromptPay QR / barcode** (`garbage_fee_qrcode`) through `garbage_fee_qrcode_transaction` and billed in **one annual invoice** (`garbage_fee_invoice`).
   - **Critical Cancellation Boundary:** Deleting only the single invoice is an incomplete, dangerous failure — the 12 transactions in `garbage_fee_transaction` would remain collectible in the database as phantom arrears! Cancellation **MUST** delete the mapping (`garbage_fee_qrcode_transaction`), the invoice (`garbage_fee_invoice`), the exclusive QR (`garbage_fee_qrcode`), **AND all linked fiscal-year charge transactions** (`garbage_fee_transaction`).

3. **Manual / Outside Payment Verification ("ประชาชนมาจ่ายมาแล้ว"):**
   - When an incident states the citizen has already paid (e.g. at the municipal counter with a manual paper receipt):
     - **Electronic Receipt Check:** Verify `garbage_fee_receipt`. If an electronic receipt exists (`receipt_id IS NOT NULL`), financial records are strictly immutable (PBR-01) and execution must **HALT**. If paid outside the system via manual paper receipt (ใบเสร็จมือ), no electronic system receipt exists.
     - **Receipt-Transaction Link Check:** Verify that none of the 12 transactions are linked in `garbage_fee_receipt_transaction` (`gfr.status <> 'receipt_status_3'`) or in `water_fee_receipt_transaction` (for integrated municipalities).
     - **Bank Gateway Check:** Query `ktb_transaction_item` by municipality `SID` and QR reference `Ref_1` to confirm no online payment arrived via Krungthai/PromptPay.

4. **Fiscal Year (`tax_year`) Scope:**
   - Annual bills cover one complete fiscal year (`tax_year = 2027` for Thai fiscal year 70 / ปีงบ 70).
   - In multi-tenant systems, payer IDs (e.g. `69006050`) are shared across municipalities; all queries must resolve the specific municipality `SID` and unique entity UUID (`garbage_id`).

5. **Next Generation Pointer (`next_invoice_generation`):**
   - For `recurring_type_4`, `garbage_fee_detail.next_invoice_generation` tracks annual intervals. Verify its state; do not infer cron execution, reset pointers, or mutate settings without explicit authority.

## Lifecycle & Stage Dispatch

Follow the canonical stage dispatch:
- **`prepare`**: Authors the fresh 5-script suite, `scope.json`, backup JSON, and Thai executive report draft.
- **`dry-run`**: Executes the reviewed `03_correct.sql` in a rollback-only transaction (`ROLLBACK;`) to preview in-transaction assertions and readbacks without changing data.
- **`commit`**: With separate, explicit human authorization and operator quiet-window confirmation, executes `03_correct.sql` ending in `COMMIT;`, capturing raw receipts.
- **`verify`**: Read-only verification from a fresh connection proving that the invoice and all linked fiscal-year charge transactions are absent, and the remaining owner balance matches the projected after-state.

## Quick Start

Use [PROMPT.md](PROMPT.md) or pass inputs:
```text
card: SU-10766
jira_env: su
database_env: smart_utilities_prod
stage: prepare
operation: annual-cancellation
```
Validate with:
```bash
node bin/sidekicks playbook validate database-correction.annual-invoice-cancellation --project su-sk
```
