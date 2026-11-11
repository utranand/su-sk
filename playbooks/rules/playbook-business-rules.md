# Playbook Business Rules: Smart Utilities (`su-sk`)

> **File:** `playbook-business-rules.md`  
> **Scope:** Smart Utilities (`su-sk`) Production Business Logic & Domain Invariants  
> **Contract:** Referencing this rules file mandates that the playbook and execution **MUST strictly follow all rule-ids defined below**. No business rule may be bypassed.

---

## Business Rule Inventory (Consolidated)

The business rules below consolidate all domain invariants, payment lifecycle checks, billing model constraints, and fiscal year boundaries into 7 clear, non-redundant rule definitions:

| Rule ID | Rule Name & Short Thai Name | Severity | Business Invariant | Former Rules Merged |
|---|---|---|---|---|
| **PBR-01** | **Invoice Lifecycle & Mutation Scope (Unpaid, Unprinted & Demographic Mutability)**<br>*(สถานะบิลและขอบเขตการแก้ไข: แก้ไขเฉพาะบิลที่ยังไม่จ่ายและยังไม่พิมพ์ / ห้ามแตะต้องบิลที่จ่ายแล้ว / แก้ชื่อ-ที่อยู่ได้)* | ⛔ CRITICAL | Financial amount fields (`garbage_fee`, `payment_amount`, QR satang) are **strictly immutable** once paid (`receipt_id IS NOT NULL`) or printed (`printed_date IS NOT NULL`). Rate adjustments apply **ONLY to unprinted, unpaid bills** (`status = invoice_status_1`, `receipt_id IS NULL`, `printed_date IS NULL`). Historical paid periods with receipts must remain completely untouched. However, non-financial demographic metadata (name, address, tax ID, deed no) **CAN be updated** on printed/paid records. | `PBR-01`, `PBR-05`, `PBR-07`, `PBR-09` |
| **PBR-02** | **Billing Model Invariants: Accumulated vs Non-Accumulated (Period Editability & Due Date Extension)**<br>*(รูปแบบการคิดเงิน: แบบทบยอด vs แบบไม่ทบยอด — การแก้ไขรอบเดือนและการห้ามต่ออายุใบแจ้งยอด)* | ⛔ CRITICAL | **Accumulated billing (ค่าน้ำประปา):** rolling arrears; updates restricted strictly to **current active month only** (cannot modify past months); **NEVER extend due date (`payment_due_date`)** because QR code is stateless and bank checks only expiry + paid status, causing double/overlapped payments.<br>**Non-accumulated billing (ค่าขยะ):** discrete month-by-month; any unpaid unprinted month can be updated; due date extension is permitted. | `PBR-02`, `PBR-10`, `PBR-12` |
| **PBR-03** | **Fiscal Year Cycle & Cross-Year Boundary Lock**<br>*(รอบปีงบประมาณ ต.ค.–ก.ย., การคำนวณ tax_year, และการห้ามแก้ไข/โยกจ่ายข้ามปีงบประมาณ)* | ⛔ CRITICAL | Fiscal year ends **September 30**. Rounds in Oct–Dec have `tax_year = round_year + 1`; rounds in Jan–Sep have `tax_year = round_year`. Public revenues and budgets cannot cross fiscal years. Amounts within a fiscal year (e.g. September) can be modified, but **CANNOT be modified, canceled, or shifted to be paid across into October (the next fiscal year)**. In multi-round adjustments, each round must be pinned strictly to its own `tax_year`. | `PBR-06`, `PBR-11` |
| **PBR-04** | **Multi-Tenant Municipality Payer Disambiguation & Strict Scope Confinement**<br>*(ล็อก Entity UUID และจำกัดขอบเขตตามเทศบาลที่ระบุอย่างเคร่งครัด: ห้ามทำข้ามเทศบาล)* | ⛔ CRITICAL | Customer payer numbers (e.g. `68000977`) are shared across municipalities. All queries and updates must resolve and pin the unique entity UUID (`garbage_id` / `water_id`) and row primary keys. **Municipality Scope Confinement:** When the Jira card assigns a municipality (e.g. `[เกาะสมุย]`), the entire scope of work, investigation, verification (PBR-01..PBR-06), and correction is strictly confined within that municipality. Never evaluate, inspect, or mutate data across other municipalities. | `PBR-03` |
| **PBR-05** | **Multi-Channel Satang Synchronization**<br>*(ซิงค์ยอดสตางค์ QR/Barcode ให้ตรงกับค่าธรรมเนียม)* | ⚠️ HIGH | Changes to invoice or transaction fees require atomic synchronization of PromptPay QR and barcode payloads in satang (`garbage_fee_qrcode.qr_code_text`, `barcode_text`) in the same transaction, ensuring payment gateways charge the exact business rate. | `PBR-04` |
| **PBR-06** | **Unpaid Invoice Cancellation by Guarded Deletion**<br>*(ยกเลิกบิลที่ยังไม่ชำระด้วยการลบ / ห้ามลบบิลที่รับชำระแล้ว)* | ⛔ CRITICAL | Cancel verified unpaid target invoices by **DELETE of exact invoice rows and their exclusive QR/barcode rows and mapping links**; future due dates alone do not block cancellation, with live payment/printing/dependency checks, scoped row-count guards, offline backup, recoverable transaction and explicit per-action approval. Overdue does not mean paid. Never delete paid invoices, receipts or bank records, and never use `invoice_status_2` as a guessed cancellation state. | `PBR-08` |
| **PBR-07** | **Demographic Data Integrity & Prohibition of Synthetic Records (Import-Only Invariant)**<br>*(ห้ามสร้างข้อมูลบุคคลเทียมเข้าสู่ฐานข้อมูล: ข้อมูลประชากรต้องมาจากการ Import เท่านั้น / ป้องกันบุคคลที่ไม่มีตัวตนจริง)* | ⛔ CRITICAL | Operational playbooks, incident handlers, and ad-hoc database corrections are **strictly prohibited from executing `INSERT INTO public.demographic`** to create synthetic, dummy, or placeholder citizen entities into production databases. Demographic entities represent verified physical/legal persons in civil registration and **MUST ONLY enter the system through the authorized civil/demographic import pipeline**. If a utility entity references a missing or orphaned `demo_id`, resolution MUST NOT insert a fake person to bypass missing-relation crashes; it must proceed via: (1) importing the verified official citizen record via official municipality demographic import, (2) re-linking to an existing verified citizen record if matched, or (3) hardening application code (frontend optional chaining, backend null handling) to handle missing demographic relations safely. | `PBR-07` |

---

## Detailed Business Logic Rules

### PBR-01: Invoice Lifecycle & Mutation Scope (Unpaid, Unprinted & Demographic Mutability)
- **ชื่อภาษาไทยสั้น (Short Thai Name):** สถานะบิลและขอบเขตการแก้ไข: แก้ไขเฉพาะบิลที่ยังไม่จ่ายและยังไม่พิมพ์ / ห้ามแตะต้องบิลที่จ่ายแล้ว / แก้ชื่อ-ที่อยู่ได้
- **Consolidation Note:** รวมกฎเดิม `PBR-01` (Unpaid Bill Invariant), `PBR-05` (Historical Paid Period Integrity), `PBR-07` (Unprinted Invoice Invariant), และ `PBR-09` (Field-Level Mutation Scope Distinction) เข้าด้วยกันเป็นกฎเดียวที่ครอบคลุม Lifecycle ของใบแจ้งหนี้และการแก้ไขข้อมูล
- **Business Rationale & Lifecycle Constraints (หลักการทางธุรกิจและวงจรชีวิตของใบแจ้งหนี้):**
  1. **ฟิลด์ทางการเงิน (Financial / Amount Fields) — ห้ามแก้ไขเมื่อพิมพ์หรือเก็บเงินแล้ว (Strictly Immutable):**
     - ประกอบด้วย: ยอดเงินค่าธรรมเนียม (`garbage_fee`, `water_fee`), ยอดชำระบนใบเสร็จ (`payment_amount`), ชื่อประเภทค่าธรรมเนียมที่ระบุจำนวนเงิน (`fee_type_name`), และข้อความสตางค์ใน QR Code/Barcode (`qr_code_text`, `barcode_text`)
     - **เงื่อนไขที่อนุญาตให้แก้ไขยอดเงิน:** จะต้องเป็นใบแจ้งหนี้ที่เข้าข่ายครบทั้ง 2 เงื่อนไข:
       - **ยังไม่ชำระเงิน (Unpaid):** `status::text = 'invoice_status_1'`, `receipt_id IS NULL`, และไม่มีรายการรับชำระในระบบธนาคาร (`ktb_transaction_item`)
       - **ยังไม่เคยสั่งพิมพ์ (Unprinted):** `printed_date IS NULL`
     - **เงื่อนไขหยุดการทำงาน (Halt Conditions):**
       - หากบิลใด **ไม่อยู่ในเงื่อนไขแก้ไขยอด (`receipt_id IS NOT NULL` หรือ `status <> invoice_status_1`)** ➡️ **สคริปต์ต้อง HALT ทันที (Raise Exception)** เพื่อป้องกันความผิดพลาดทางบัญชีและการกระทบยอดเงิน
       - หากบิลใด **พิมพ์ไปแล้ว (`printed_date IS NOT NULL`)** ➡️ **สคริปต์ต้อง HALT ทันที (Raise Exception)** เพื่อป้องกันความขัดแย้งกับยอดเงินในใบแจ้งหนี้กระดาษที่ส่งถึงมือประชาชนแล้ว
     - **รอบบิลในอดีตที่ชำระเงินแล้ว (Historical Paid Periods):** รอบบิลใดๆ ที่มีใบเสร็จรับเงินแล้ว ถือเป็นเอกสารทางการเงินที่ปิดรอบบัญชีแล้วและมีผลผูกพันทางกฎหมาย **ห้ามแก้ไข ย้อนคืน หรือเปลี่ยนแปลงข้อมูลโดยเด็ดขาด**
  2. **ฟิลด์ข้อมูลทั่วไปและข้อมูลบุคคล (Non-Financial / Demographic Fields) — สามารถแก้ไขได้ (Permitted):**
     - ประกอบด้วย:
       - **ชื่อ-นามสกุล:** `title`, `firstname`, `lastname`, `main_user_fullname`
       - **ที่อยู่:** `address_no`, `moo`, `alley`, `road`, `village_name`, `subdistrict_name`, `district_name`, `province_name`, `postcode`, `invoice_address`
       - **ข้อมูลทางทะเบียนและภาษี:** เลขประจำตัวผู้เสียภาษี / เลขบัตรประชาชน, เลขที่โฉนดที่ดิน (`deed_no`), หมายเหตุการชำระ (`payment_remarks`)
     - **ข้อกำหนดทางธุรกิจ:** **อนุญาตให้แก้ไขได้** แม้ว่าใบแจ้งหนี้หรือใบเสร็จนั้นจะถูกพิมพ์ไปแล้วหรือเก็บเงินแล้วก็ตาม เพื่อให้ประชาชนนำเอกสารไปใช้เบิกจ่ายหรือยื่นภาษีได้อย่างถูกต้อง โดยไม่มีผลกระทบต่อยอดเงินรวม ไม่กระทบต่อรายการเดินบัญชีธนาคาร และไม่กระทบต่อการสรุปยอดกระทบยอด (Reconciliation)
- **Enforcement & PL/pgSQL Guards (ข้อบังคับในคำสั่งแก้ไข):**
  - ใน Safety Guard Assertion ของ `03_correct.sql` สำหรับการแก้ไขยอดเงิน จะต้องตรวจสอบทั้งบิลที่จ่ายแล้วและบิลที่พิมพ์แล้ว:
    ```sql
    -- 1. ตรวจสอบบิลที่จ่ายแล้ว (Halt if paid)
    select count(*) into n_paid
    from garbage_fee_invoice
    where garbage_id = '<ENTITY_ID>'
      and invoice_id in (<INVOICE_IDS>)
      and (receipt_id is not null or status::text <> 'invoice_status_1');

    if n_paid > 0 then
      raise exception 'HALT (PBR-01): % bill(s) fail amount-change eligibility. Reconcile status/payment evidence before continuing.', n_paid;
    end if;

    -- 2. ตรวจสอบบิลที่พิมพ์แล้ว (Halt if printed)
    select count(*) into n_printed
    from garbage_fee_invoice
    where garbage_id = '<ENTITY_ID>'
      and invoice_id in (<INVOICE_IDS>)
      and printed_date is not null;

    if n_printed > 0 then
      raise exception 'HALT (PBR-01): % bill(s) in target period have ALREADY BEEN PRINTED (printed_date is not null). Aborting execution. User decision required.', n_printed;
    end if;
    ```
  - คำสั่ง `UPDATE` ยอดเงินต้องมีเงื่อนไข `and status::text = 'invoice_status_1' and receipt_id is null and printed_date is null` กำกับเสมอ
  - หากเป็นการแก้ไขข้อมูลทั่วไป (Demographic) คำสั่ง `UPDATE` จะต้องแยกออกจากฟิลด์ตัวเลขทางการเงิน และมี `remarks` บันทึกเหตุผลการแก้ไข
- **Verification Checklist (วิธีการตรวจสอบ):**
  1. **สคริปต์ `01_investigate_readonly.sql`:** ตรวจสอบ `status`, `receipt_id`, `printed_date` ยืนยันว่าค่าเป็นไปตามข้อกำหนดก่อนดำเนินการ
  2. **สคริปต์ `03_correct.sql`:** มี Assertion ดักจับทั้งบิลที่จ่ายแล้วและบิลที่พิมพ์แล้ว และคำสั่ง UPDATE มีเงื่อนไขป้องกันครบถ้วน
  3. **สคริปต์ `04_verify_readonly.sql`:** ตรวจสอบว่ารอบบิลในอดีตที่จ่ายแล้วยังคงค่าเดิมครบถ้วน 100%

---

### PBR-02: Billing Model Invariants: Accumulated vs Non-Accumulated (Period Editability & Due Date Extension)
- **ชื่อภาษาไทยสั้น (Short Thai Name):** รูปแบบการคิดเงิน: แบบทบยอด vs แบบไม่ทบยอด — การแก้ไขรอบเดือนและการห้ามต่ออายุใบแจ้งยอด
- **Consolidation Note:** รวมกฎเดิม `PBR-02` (Garbage Billing Non-Accumulation), `PBR-10` (Billing Period Editability), และ `PBR-12` (Invoice Due Date Extension Invariant) เข้าด้วยกันเป็นกฎเดียวที่ควบคุมกลไกความแตกต่างระหว่างระบบทบยอดและไม่ทบยอด
- **Billing Mechanics Comparison (เปรียบเทียบรูปแบบการคิดเงิน):**
  1. **ค่าบริการแบบทบยอด (Accumulated / Rolling Arrears Billing — เช่น ค่าน้ำประปา `water_fee_*`):**
     - **กลไกหนี้สะสม:** ยอดหนี้ค้างชำระของเดือน $M$ จะถูกคำนวณยกยอดไปบวกทบเป็นยอดหนี้ในใบแจ้งหนี้ของเดือนถัดไป ($M+1$) ทันที
     - **การแก้ไขรอบเดือน (Period Editability):** **สามารถแก้ไขได้เฉพาะ "รอบเดือนปัจจุบัน" เท่านั้น** ห้ามแก้ไขย้อนหลังข้ามเดือนในอดีตโดยเด็ดขาด เพราะจะทำให้ยอดหนี้สะสมยกยอดในใบแจ้งหนี้ของเดือนถัดๆ ไปขัดแย้งกับยอดหนี้จริง เกิดความเสียหายต่อสายการคำนวณยอดหนี้ต่อเนื่อง (Cascading Arrears Inconsistency)
     - **การห้ามต่ออายุใบแจ้งยอด (Due Date Extension Prohibition):** **จะไม่มีการต่ออายุหรือเพิ่มอายุใบแจ้งยอด (`payment_due_date`) โดยเด็ดขาด**
       - **Stateless QR & Direct Bank Gateway Integration:** QR Code (PromptPay / Bill Payment) มีคุณลักษณะเป็น **Stateless** อย่างสมบูรณ์ เมื่อประชาชนสแกนผ่าน Mobile Banking ระบบจะยิงตรงเข้าไปยังธนาคาร โดยธนาคารจะตรวจสอบเพียง 2 เงื่อนไขเท่านั้น คือ:
         1) วันหมดอายุ (`payment_due_date`) ว่ายังไม่หมดอายุใช่ไหม
         2) สถานะการชำระเงิน (Paid Status) ว่า Reference นี้จ่ายไปแล้วหรือยัง
       - ธนาคารไม่รู้ว่ายอดหนี้ของบิลเก่าได้ถูกยกยอดไปใส่ในบิลเดือนถัดไปแล้ว หากมีการต่ออายุบิลเก่า ธนาคารจะเห็นว่าบิลเก่ายังไม่หมดอายุและยังไม่จ่าย จึงอนุญาตให้ตัดเงินได้ทั้งบิลเก่าและบิลใหม่ ทำให้เกิด **"การชำระเงินซ้ำซ้อน (Double Payment)"** หรือ **"ยอดชำระเหลื่อมทับกัน (Overlapped Payment)"** ซึ่งทำให้ระบบกระทบยอด (Reconciliation) ผิดพลาดอย่างรุนแรง
  2. **ค่าบริการแบบไม่ทบยอด (Non-Accumulated / Discrete Billing — เช่น ค่าธรรมเนียมขยะ `garbage_fee_*`):**
     - **กลไกยอดเดี่ยว:** แต่ละรอบเดือนเป็นเอกเทศจากกัน (Discrete month-by-month) ไม่มียอดหนี้ทบไปรอบถัดไป
     - **การแก้ไขรอบเดือน (Period Editability):** **สามารถแก้ไขรอบเดือนใดก็ได้** ย้อนหลังหลายรอบเดือนได้อย่างอิสระ (ตราบเท่าที่เป็นไปตามกฎ `PBR-01` คือยังไม่จ่ายและยังไม่พิมพ์)
     - **การต่ออายุใบแจ้งยอด (Due Date Extension):** **สามารถต่ออายุหรือขยายวันครบกำหนดชำระได้** เนื่องจากแต่ละบิลเป็นของเดือนนั้น ๆ โดยเฉพาะ ไม่มีการยกยอดหนี้ค้างไปรวมทบกับบิลเดือนถัดไป การขยายเวลาชำระจึงไม่ทำให้เกิดยอดหนี้ซ้ำซ้อนหรือทับซ้อนกับบิลเดือนอื่น
- **Enforcement & Operational Guardrails (ข้อบังคับในระบบ):**
  - หากเป็นระบบทบยอด (ค่าน้ำประปา):
    - ใน Safety Guard Assertion ต้องมีคำสั่งตรวจสอบว่ารอบเดือนเป้าหมายเป็นเดือนปัจจุบันเท่านั้น (`round_month = current_month`)
    - ห้ามมีคำสั่ง UPDATE `payment_due_date` บนตารางค่าน้ำประปา (`water_fee_*`) โดยเด็ดขาด
  - หากเป็นระบบไม่ทบยอด (ค่าขยะ):
    - อนุญาตให้ระบุแก้ไขหลายเดือนพร้อมกันได้ (เช่น `round_month in (8, 9, 10)` สำหรับค่าขยะ)
    - แต่ละรอบเดือนต้องมี PromptPay QR Code และตัวเลขสตางค์แยกของตนเอง (เช่น 60 บาท = `6000` สตางค์ต่อเดือน) ห้ามนำมารวมเป็นยอดเดียว
- **Verification Checklist (วิธีการตรวจสอบ):**
  1. **สคริปต์ `01_investigate_readonly.sql`:** ตรวจสอบประเภทบริการว่าเป็นแบบทบยอดหรือแบบไม่ทบยอด
  2. **สคริปต์ `03_correct.sql`:** ยืนยันว่าหากเป็นแบบทบยอด มีเงื่อนไขเฉพาะเดือนปัจจุบันและไม่มีการแก้ไข `payment_due_date`
  3. **เกณฑ์การผ่าน (Expected Result):** บริการแบบทบยอดไม่มีการแก้ไขย้อนหลังข้ามเดือนและไม่มีการต่ออายุบิล; บริการแบบไม่ทบยอดมี QR Code แยกรายเดือนสอดคล้อง 100%

---

### PBR-03: Fiscal Year Cycle & Cross-Year Boundary Lock
- **ชื่อภาษาไทยสั้น (Short Thai Name):** รอบปีงบประมาณ ต.ค.–ก.ย., การคำนวณ tax_year, และการห้ามแก้ไข/โยกจ่ายข้ามปีงบประมาณ
- **Consolidation Note:** รวมกฎเดิม `PBR-06` (Fiscal Year Boundary & Accounting Year Cycle) และ `PBR-11` (Cross-Fiscal-Year Mutation & Cancellation Prohibition) เข้าด้วยกันเป็นกฎการบริหารจัดการปีงบประมาณที่สมบูรณ์
- **Fiscal Calendar Invariant & tax_year Formula (ปฏิทินปีงบประมาณและสูตรการคำนวณ):**
  - **วันสิ้นสุดปีงบประมาณ:** ปีงบประมาณขององค์กรปกครองส่วนท้องถิ่นและทางราชการ สิ้นสุดทุกวันที่ **30 กันยายน** ของทุกปี
  - **การกำหนดปีงบประมาณ (`tax_year`):**
    - ค่าใช้จ่าย/รายได้ที่เกิดขึ้นระหว่าง **1 ตุลาคม ถึง 30 กันยายนของปีถัดไป** ถือเป็นปีงบประมาณของปีถัดไป
    - **สูตรคำนวณ `tax_year`:**
      $$\text{tax\_year} = \begin{cases} \text{round\_year} + 1 & \text{เมื่อ } \text{round\_month} \in \{10, 11, 12\} \\ \text{round\_year} & \text{เมื่อ } \text{round\_month} \in \{1, 2, \dots, 9\} \end{cases}$$
    - **ตัวอย่าง:**
      - รอบเดือน สิงหาคม 2026 (`round_month = 8`): ปีงบประมาณ 2026 (`tax_year = 2026`, พ.ศ. 2569)
      - รอบเดือน กันยายน 2026 (`round_month = 9`): ปีงบประมาณ 2026 (`tax_year = 2026`, พ.ศ. 2569)
      - รอบเดือน ตุลาคม 2026 (`round_month = 10`): จัดเป็นปีงบประมาณ 2027 (`tax_year = 2027`, พ.ศ. 2570)
- **Closed Fiscal Year Lock & Cross-Year Prohibitions (การล็อกรอยต่อสิ้นปีงบประมาณและข้อห้ามเด็ดขาด):**
  1. **งบประมาณไม่สามารถเบิกจ่ายหรือชำระข้ามปีได้:** ตามกฎหมายวินัยการเงินการคลังและระเบียบกระทรวงมหาดไทย งบประมาณ รายได้ และค่าใช้จ่าย **ไม่สามารถเบิกจ่ายหรือนำเงินมาชำระข้ามปีงบประมาณได้**
  2. **การแก้ไขยอดเงินในรอบบิลของปีงบประมาณเดิม (Permitted within the same fiscal year):**
     - หากเป็นการแก้ไขยอดเงินของรอบบิลที่อยู่ในปีงบประมาณเดิม เช่น **ยอดเงินของรอบเดือนกันยายน** ซึ่งอยู่ในปีงบประมาณเดิม:
     - **สามารถแก้ไขได้** (ตราบเท่าที่เป็นไปตามกฎ `PBR-01` บิลยังไม่จ่าย, ยังไม่พิมพ์, และ `PBR-02` ค่าบริการแบบไม่ทบยอด) เพื่อให้ยอดเรียกเก็บของรอบเดือนกันยายนถูกต้องตามระเบียบ
  3. **ห้ามแก้ไขให้มาจ่ายข้ามมาเดือนตุลาคมโดยเด็ดขาด (Strict Prohibition on Cross-Year Payment to October):**
     - **ไม่สามารถแก้ไขให้นำยอดเงินของเดือนกันยายนมาจ่ายข้ามมาในเดือนตุลาคมได้** เนื่องจากเดือนตุลาคมถือเป็นปีงบประมาณใหม่ (`tax_year` ถัดไป)
     - **ห้าม** รวมยอดเงิน ห้ามยกยอดหนี้ และห้ามโอนย้ายเงินของเดือนกันยายนมาจ่ายในใบแจ้งหนี้เดือนตุลาคม
     - **ห้าม** นำเงินที่ชำระผ่าน QR/บิลของเดือนตุลาคมไปตัดชำระหนี้ของเดือนกันยายน
     - เมื่อพ้นวันที่ 30 กันยายนมาแล้ว การเงินและรายได้ของปีงบประมาณเดิมจะต้องถูกรับรู้และชำระภายใต้ `tax_year` เดิมเท่านั้น
  4. **การจัดการกรณีครอบคลุมหลายปีงบประมาณ (เช่น SU-10716 ส.ค.–ต.ค. 2569):**
     - ในกรณีที่มีคำร้องขอปรับอัตราค่าธรรมเนียมหลายรอบเดือนที่คาบเกี่ยวเส้นแบ่งปีงบประมาณ (ส.ค.–ก.ย. อยู่ใน `tax_year = 2026`, ต.ค. อยู่ใน `tax_year = 2027`):
     - **คำสั่งแก้ไขจะต้องแยกอัปเดตแต่ละแถวตาม `tax_year` ของตนเองอย่างเด็ดขาด** (เช่น ส.ค. 60 บาทใน FY2026, ก.ย. 60 บาทใน FY2026, ต.ค. 60 บาทใน FY2027)
     - แต่ละเดือนมี QR Code และ Payload สตางค์แยกกัน ประชาชนชำระแยกบิลของตนเอง เงินของเดือนกันยายนเข้าบัญชี FY2026 และเงินของเดือนตุลาคมเข้าบัญชี FY2027 อย่างเป็นเอกเทศ
     - **ห้าม** รวมยอดเงินของ ส.ค./ก.ย. มาไว้ในใบแจ้งหนี้ ต.ค. หรือนำเงินรับชำระของ ต.ค. ไปหักล้างหนี้ของ ส.ค./ก.ย. ข้ามปีงบประมาณ
  5. **Halt on Cross-Year Shift Attempts:**
     - หากพบคำสั่งที่พยายามโยกย้ายเงินข้ามปีงบประมาณ หรือขอยกเลิกบิลของปีงบประมาณเดิมเพื่อโอนเงินมาจ่ายปีใหม่ (เช่น เอาเงินเดือน ก.ย. มาจ่ายในเดือน ต.ค.) ➡️ **สคริปต์จะต้องหยุดการทำงานทันที (HALT / Raise Exception)**
- **Verification Checklist (วิธีการตรวจสอบ):**
  1. **สคริปต์ `01_investigate_readonly.sql`:** ตรวจสอบคอลัมน์ `tax_year` ของทุกใบแจ้งหนี้เป้าหมาย ยืนยันว่าระบุ `tax_year` ตรงตามสูตรปฏิทินงบประมาณ ไม่มีการจับคู่ผิดปี
  2. **สคริปต์ `03_correct.sql`:** ตรวจสอบการ JOIN ตารางว่าผูก `t.tax_year = i.tax_year` เสมอ และไม่มีคำสั่งใดที่ถ่ายโอนตัวเลขเงินข้าม `tax_year`
  3. **เกณฑ์การผ่าน (Expected Result):** ยอดเงินและธุรกรรมของแต่ละปีงบประมาณคงอยู่ภายในขอบเขต `tax_year` ของตนเอง 100% ไม่มีการเบิกจ่ายหรือชำระข้ามปีงบประมาณ

---

### PBR-04: Multi-Tenant Municipality Payer Disambiguation & Strict Scope Confinement
- **ชื่อภาษาไทยสั้น (Short Thai Name):** ล็อก Entity UUID และจำกัดขอบเขตตามเทศบาลที่ระบุอย่างเคร่งครัด (ห้ามทำข้ามเทศบาล)
- **Consolidation Note:** คงเดิมจาก `PBR-03` เดิม พร้อมกำหนดข้อบังคับ Municipality Scope Confinement เพิ่มเติม
- **Disambiguation & Strict Scope Confinement Invariants (การจำกัดขอบเขตและไม่ซ้ำซ้อนของข้อมูลหลายเทศบาล):**
  1. **Multi-Tenant Payer Collision:** ในระบบ Smart Utilities ฐานข้อมูลถูกออกแบบเป็น Multi-tenant รองรับหลายองค์กรปกครองส่วนท้องถิ่นในฐานข้อมูลเดียวกัน รหัสผู้ชำระเงินที่ระบุมา (`garbage_no` / `water_no`) **ไม่ใช่ค่าที่ไม่ซ้ำกันในระดับโกลบอล (Not Globally Unique)** ชุดตัวเลขเดียวกันสามารถถูกสร้างซ้ำกันได้ในหลายเทศบาลหรือหลายเขตพื้นที่
  2. **Municipality Assignment & Strict Scope Confinement (ข้อบังคับการจำกัดขอบเขตตามเทศบาลที่ระบุในการ์ด Jira):**
     - หากในการ์ด Jira มีการระบุชื่อเทศบาลมาอย่างชัดเจน เช่น ในหัวข้อการ์ดระบุ `[เกาะสมุย]`, `[เกาะพะงัน]`, `[ดอนสัก]` หรือระบุในเนื้อหา:
     - **ขอบเขตการทำงาน (Scope of Work), การสืบค้นตรวจสอบ (Investigation & Grounding), การประเมินกฎ (Rules Verification), และการแก้ไขข้อมูล (Correction): จะต้องจำกัดและกระทำอยู่ "ภายในเทศบาลที่กำหนดเท่านั้น"**
     - **ห้ามทำการตรวจสอบ ประเมินกฎ หรือแก้ไขข้อมูลข้ามเทศบาลโดยเด็ดขาด (Strict Municipality Isolation):**
       - แม้ว่าเลขผู้ชำระ (`garbage_no`) จะพบซ้ำในเทศบาลอื่น ข้อมูลของเทศบาลอื่นถือเป็น **Out of Scope โดยสมบูรณ์** ไม่ใช่เป้าหมายของการ์ด
       - **การประเมินกฎทั้งหมด (PBR-01 ถึง PBR-06):** ต้องประเมินเงื่อนไขและผลกระทบเฉพาะข้อมูลที่อยู่ภายใต้ Entity UUID ของเทศบาลที่ระบุเท่านั้น ห้ามนำข้อมูล สถานะบิล ยอดชำระ หรือประวัติของเทศบาลอื่นมาพิจารณา หรือนำมาสรุปว่าเกิดผลกระทบขัดแย้งกับคำขอ
       - **ตัวอย่างกรณี PBR-06 (การยกเลิกบิล):** หากการ์ดระบุ `[เกาะสมุย]` และขอยกเลิกบิลก่อน พ.ค. 69 การตรวจสอบต้องทำเฉพาะในเทศบาลนครเกาะสมุย เมื่อพบว่าในเกาะสมุยมี 0 แถว ขอบเขตในเทศบาลนั้นจึงเป็น 0 แถว (NO-OP ในเทศบาลที่ระบุ) จะต้องไม่นำบิลของเทศบาลอื่น (เช่น อบต.บางปลา, เทศบาลเมืองดอนสัก) ที่มีการชำระเงินแล้วมานับรวมเพื่อบอกว่าผิด PBR-06 เพราะเทศบาลอื่นอยู่นอกขอบเขตการทำงานอย่างสิ้นเชิง
- **Enforcement (ข้อบังคับในการเขียนคำสั่ง SQL และการปฏิบัติการ):**
  - **ระบุ Entity UUID ของเทศบาลเป้าหมายทันที:** ในขั้นตอนสืบค้น (`01_investigate_readonly.sql`) เมื่อพบว่าการ์ดระบุเทศบาล ให้ฟิลเตอร์หรือเลือกเฉพาะ `garbage_id` (หรือ `water_id`) ที่สังกัดเทศบาลนั้น
  - **ห้ามใช้เพียงรหัส `garbage_no` หรือ `water_no`:** ห้ามใช้เพียงรหัสผู้ชำระเป็นเงื่อนไขหลักในการ `UPDATE` หรือ `DELETE` โดยไม่มีการระบุตัวตนระดับแถวเด็ดขาด
  - **คิวรีและ Safety Guard ทั้งหมดผูกกับ Entity UUID เดียว:** ทุกคำสั่งใน `01_investigate_readonly.sql`, `02_backup.sql`, `03_correct.sql`, `04_verify_readonly.sql`, `05_rollback.sql` ต้องล็อก Foreign Key `garbage_id = '<SPECIFIED_MUNICIPALITY_ENTITY_UUID>'` (หรือ `water_id`) และ Primary Key UUIDs ของตาราง (`transaction_id in (...)`, `invoice_id in (...)`, `qrcode_id in (...)`) เสมอ
- **Verification Checklist (วิธีการตรวจสอบ):**
  1. **สคริปต์ `01_investigate_readonly.sql`:** ยืนยันว่าเลือก `garbage_id` ที่ตรงกับเทศบาลที่การ์ดระบุ (เช่น `<ENTITY_UUID>` สำหรับเกาะสมุย) และคิวรีตรวจสอบทั้งหมดจำกัดอยู่เฉพาะ Entity นี้
  2. **การประเมินกฎ (PBR Verification):** ผลการตรวจสอบ PBR-01..PBR-06 สะท้อนสถานะเฉพาะภายในเทศบาลที่ระบุ ไม่มีการดึงข้อมูลเทศบาลอื่นมาตัดสิน
  3. **สคริปต์ `03_correct.sql` และ `05_rollback.sql`:** มี WHERE clause กำกับด้วย `garbage_id` ของเทศบาลที่ระบุ และ Primary Key ชัดเจน
  4. **เกณฑ์การผ่าน (Expected Result):** การดำเนินการทั้งหมดถูกกักขอบเขต (Contained) อยู่เฉพาะภายในเทศบาลที่กำหนด 100% ไม่เกิดผลกระทบต่อเทศบาลอื่น

---

### PBR-05: Complete QR/Barcode Satang Synchronization

- Any financial amount correction must update the verified invoice amount label, linked transaction fee and both QR/barcode amount payloads atomically. Preserve references, due dates, fiscal fields and every unrelated payload component.
- Inspect the actual payload structure. For the verified delimited garbage BillPayment format, replace the **complete final amount token**, after `%0A` in QR text and `+` in barcode text. Compute integer satang from the actual currency representation;150baht is15000satang. Never use a last-four-characters comparison or replacement.
- Assert the original full payload before correction and the exact whole expected payload afterward, including preserved references. Mixed or unknown formats require verified transformation evidence rather than truncation assumptions.
- Verify payload values through in-transaction readback and a fresh post-action query. An EXPLAIN plan or offline string projection is preparation evidence only.
- For cancellation under PBR06, remove the approved exclusive QR/barcode row and mappings instead of changing their amount or guessing a cancelled status. Check payments by saved original reference values after deletion. Database row removal alone does not establish what an external bank will do with a saved QR image.

---

### PBR-06: Complete Unpaid Invoice and Charge Cancellation by Guarded Deletion
- **ชื่อภาษาไทยสั้น:** ยกเลิกใบแจ้งหนี้พร้อมรายการค่าธรรมเนียมที่ยังไม่ชำระและข้อมูล QR/Barcode / ห้ามลบหลักฐานการรับชำระ
- **Human method clarification — 9 November 2026:** "since it was not paid yet, you can cancel it. To cancellation the invoice is to delete it from the database." Exclusive QR/barcode rows and mapping links are included; future due dates alone do not block.
- **Human completion clarification — 10 November 2026:** "since user requests for removing the invoice, these old records will need to be removed too"; the owner then requested this cancellation removal be added to the reusable playbook templates. Removing only invoice/QR/mapping rows is incomplete when their old charge transactions remain collectible. This replaces the earlier blanket preserve-financial-transactions instruction for the corresponding verified unpaid cancellation targets. Paid and unrelated financial records remain protected. A method/template change is never live mutation consent.

For amount updates, the verified garbage status1 is the eligible unpaid/unprinted state. Cancellation has its own eligibility checks; neither a status nor an absent invoice is enough to establish that a charge can be deleted.

#### Complete cancellation scope
1. **Delete the exact verified unpaid invoice, its exclusive QR/barcode rows and mappings, and its corresponding exclusively linked unpaid charge transactions in one guarded transaction.** For the verified garbage schema the charge rows are `garbage_fee_transaction`; both QR and barcode payloads are in `garbage_fee_qrcode`, with links in `garbage_fee_qrcode_transaction`. Discover the actual utility/schema and dependency order. Pin every invoice/transaction/QR UUID and mapping key by municipality/system/entity and approved periods/tax years. Do not select by payer number alone or broaden a request to all payer transactions.
2. **Owner collection must no longer include the canceled charge transactions.** An invoice/QR-only deletion is not completion while those transactions still appear as unpaid or contribute to the payer's remaining balance. Verify exact transaction absence and the projected remaining transaction set/balance, preserving all non-target periods and their correct fees. Zero invoice rows is not a NO-OP if the corresponding historical charges remain.
3. **Never delete paid, shared or unrelated accounting records.** Preserve receipts, receipt links, bank items, manual-receipt/payment evidence, unrelated transactions and payer/master/future configuration. A target charge referenced by a non-target invoice, QR, receipt, integrated-water billing/payment record, overdue notice or another live dependency is a hard stop until reconciled; never remove those references to manufacture eligibility or silently widen the deletion set. Prove every relation is exclusive to the approved cancellation; any approved dependent deletion has its own exact keys/counts.
4. **No guessed cancellation status or cascade.** `invoice_status_2` is a printed/awaiting-payment transition in the verified garbage workflow, not cancellation. An overdue `invoice_status_4` is not automatically paid; verified status1/4 candidates still need unpaid/unprinted/payment guards. Future due dates alone do not block; do not extend or rewrite them. Existing shared QR or duplicate payment references are stops.
5. **Previously deleted invoice follow-up:** use immutable original invoice/QR/mapping evidence to establish the exact canceled periods, charge UUIDs, tenant/entity and saved payment references. Re-ground current charge rows, all live dependencies and payment state; obtain a new exact transaction-cleanup grant. If invoice/QR/mappings are already absent, record expected counts of zero for those tables and delete only the remaining approved charges. A new cleanup restore contains only rows actually removed by that action; it must not recreate invoice/QR rows canceled by an earlier committed action. Missing ownership/reference evidence is a hold, not a guessed transaction match.

#### Mandatory payment, accounting and job checks
- Halt for a paid status, `receipt_id`, printing evidence, receipt/payment relationship, or collected/in-flight bank match. A user's unpaid statement cannot override contradictory live records. Inspect both garbage and integrated-water relationships when the actual schema supports them; inspect metadata and source before writing predicates.
- Check bank evidence with immutable original references plus tenant even after invoice/QR deletion. In the previously verified garbage schema KTB uses quoted `"SID"` and `"Ref_1"`; there is no bank `status` column. Zero scoped database matches do not establish external/manual payment state. Preserve manual accounting; unresolved contradictions require reconciliation.
- Discover all actual dependent columns/FKs, triggers, rewrite rules, RLS and inheritance/partition behavior; account for dependencies without FKs. Freeze the discovered reference inventory and reject schema/inventory drift before DML. Do not assume that absence of a database FK means absence of an application relationship.
- Trace the owner-collection/balance query and actual cron, billing-generation, backfill and next-generation-pointer paths. Determine whether rows were retained or recreated and whether a later job could recreate/reprice canceled periods. A system `created_by` alone is not proof of a cron run. Do not rerun jobs, reset generation pointers, change rates, or suspend workers as an inferred correction. Any necessary recreation/concurrency control needs its own reviewed scope and applicable authorization. Distinguish verified current absence from future job behavior not exercised.

#### Required preparation, execution and recovery
1. Export an immutable consistent full-row before-state for every target invoice, charge transaction, QR row and mapping plus preserved relationships/settings; retain timestamps/checksum and original payment references. Verify typed full-column recoverability, including nulls, JSON/enums and timestamps. No production backup tables or DDL (PAR01).
2. Author one exact PK + tenant/entity + period/tax-year transaction, with bounded timeouts/locks, expected per-table counts, complete before-state drift, unpaid/unprinted/payment, exclusivity/dependency and configuration guards. Use the observed dependency graph to remove target references before parents; capture every removed invoice/transaction/QR UUID, mapping key and actual count. Fail/rollback on any discrepancy. Deleting a row means its audit trail is the immutable backup, RETURNING/NOTICE receipts and Jira, not a new remark on a vanished row.
3. Each live rollback-only dry-run, COMMIT and later DML restore requires its own explicit exact-action approval under Rule4/PAR04. A successful real dry-run must show actual counts/IDs, in-transaction after-state, literal ROLLBACK and complete original-state restoration on a fresh connection. EXPLAIN without ANALYZE, typed SELECTs, offline models and CLI previews are preparation only.
4. Before COMMIT, address demonstrated concurrent collection/QR/payment-ingestion or recreation races with an actual operator-confirmed payer quiet window or separately authorized reviewed control. Row locks/SERIALIZABLE alone cannot exclude noncooperating inserts without FKs. Do not assume a quiet window from an empty lock scan. Freeze reviewed buffers/hashes, use one atomic ON_ERROR_STOP session and an exclusive one-attempt marker; reconcile possible COMMIT failures read-only before any retry.
5. Inside the mutation transaction verify exactly the approved invoice/transaction/QR/mapping sets are absent, including saved-reference checks independent of surviving joins. After a dry-run ROLLBACK, verify full original rows/IDs/balance restored from a fresh connection; after COMMIT, require the intended removal/balance to persist on a fresh connection. Compare all retained full rows/settings to the exact projected after-state and verify owner-collection transaction IDs and remaining balance. Preserve fiscal boundaries and non-target amounts. Database deletion alone does not prove that external saved QR images disappeared or that a bank will reject them.
6. Prepare rollback-default full-row DML recovery in actual dependency order (charge/QR parents before dependent mapping/invoice rows where required by the captured schema). Reject PK/period/reference conflicts, new payments and intervening row/configuration drift; never use ON CONFLICT overwrite. Bind recovery to actual committed after-state. For transaction-only follow-ups restore only that action's deleted charge rows. A later live restore needs fresh permission; automatic rollback of an uncommitted failure is immediate and already part of its approved action.
7. Record exact before/after IDs, per-table planned/actual counts, balance projection, source/wrapper/snapshot hashes, authority, independent review and actual ROLLBACK/COMMIT receipts in the parent workspace and Jira. Never reuse an old approval or mutate historical incident snapshots/candidates to fit a new action.

#### PBR-06 decision gate
- **PASS for preparation:** complete invoice/charge/QR/mapping scope (or evidenced transaction-only follow-up) is established; all payment, printing, identity and exclusivity checks pass; complete backup/recovery and guarded SQL are reviewed. Runtime approvals remain separate.
- **HOLD:** any paid/printing/shared-reference/dependency/accounting uncertainty, unsafe recreation/concurrency path, scope/schema/before-state mismatch, unreviewed expansion or missing recoverable original. Future due dates alone are not a hold.
- **Cancellation completed:** the separately approved action committed, actual receipts and fresh full-state checks prove the complete intended removal including collectible charges, and retained transactions/balance/accounting/settings match the approved projection. An absent invoice, registration or offline review alone is not completion.

---

### PBR-07: Demographic Data Integrity & Prohibition of Synthetic Citizen Records (Import-Only Invariant)
- **ชื่อภาษาไทยสั้น:** ห้ามสร้างข้อมูลบุคคลเทียมเข้าสู่ฐานข้อมูลเด็ดขาด / ข้อมูลประชากรต้องมาจากการ Import อย่างเป็นทางการเท่านั้น / ป้องกันบุคคลที่ไม่มีตัวตนจริง (No-Actual Person Prevention)
- **Human instruction — 11 November 2026:** "never adding new demographic record to prevent no-actual person into the database, since this record must be imported into the system only."

#### Core Principle & Invariant (หลักการและความถูกต้องของข้อมูล)
1. **ห้ามสร้างหรือ Insert แถวข้อมูลบุคคลใหม่ (`public.demographic`) ขึ้นมาเองโดยเด็ดขาด (Prohibition of Synthetic Demographic Records):**
   - Playbooks, operational scripts, incident handlers, และคำสั่งแก้ไขฐานข้อมูลทุกประเภท **ห้ามรันคำสั่ง `INSERT INTO public.demographic`** เพื่อสร้างข้อมูลบุคคลสังเคราะห์ (Synthetic/Mock/Placeholder Entities เช่น "ผู้ใช้น้ำ 69000782") หรือใส่ข้อมูลที่ไม่มีตัวตนจริง (No-Actual Person) ลงในฐานข้อมูล Production เพื่อแก้ปัญหาหน้าจอ Error, เลี่ยงบั๊กของระบบ, หรือแก้ปัญหาคีย์กำพร้า
2. **ข้อมูลบุคคลต้องมาจากการนำเข้าอย่างเป็นทางการเท่านั้น (Import-Only Invariant):**
   - ข้อมูลในตาราง `public.demographic` เป็นข้อมูลระเบียนราษฎร์และประชากรที่มีผลทางกฎหมายและการคุ้มครองข้อมูลส่วนบุคคล (PDPA)
   - ข้อมูลจะต้องเข้าสู่ระบบผ่านกระบวนการ **Import ข้อมูลอย่างเป็นทางการ (Official Civil/Demographic Import Pipeline)** จากองค์กรปกครองส่วนท้องถิ่น (อปท.) หรือฐานข้อมูลทะเบียนราษฎร์กรมการปกครอง (DOPA) เท่านั้น เพื่อรับประกันว่าบุคคลทุกคนในฐานข้อมูลมีตัวตนอยู่จริงและมีเลขประจำตัวประชาชนที่ถูกต้อง
3. **การป้องกันปัญหาบุคคลจำลอง / ประชากรผี (Ghost Citizen Prevention):**
   - การ Insert ข้อมูลประชากรเทียมลงในฐานข้อมูล Production ส่งผลกระทบต่อ:
     - ความถูกต้องของฐานข้อมูลประชากรภาครัฐ
     - การกระทบยอดทางสถิติและภาษีขององค์กรปกครองส่วนท้องถิ่น
     - การสืบค้นประวัติบุคคลในระบบบูรณาการ (Cross-Service Integration)
     - เสี่ยงต่อการนำข้อมูลเทียมไปผูกกับทรัพย์สินหรือใบเสร็จรับเงินที่ผิดกฎหมาย

#### Resolution Matrix for Missing / Orphaned `demo_id` (แนวทางปฏิบัติเมื่อพบข้อมูลคีย์กำพร้า)
เมื่อเกิดอุบัติการณ์ที่ข้อมูลผู้ใช้น้ำ (`water_fee_detail`) หรือผู้ชำระค่าขยะ (`garbage_payer` / `garbage_fee_detail`) อ้างอิง `demo_id` ที่ไม่มีอยู่จริงในตาราง `demographic`:
- **แนวทางที่ 1 (Official Demographic Import):** นำเข้าข้อมูลประชากรของประชาชนตัวจริงเข้าสู่ระบบผ่านกระบวนการ Import ข้อมูลทะเบียนราษฎร์ตามขั้นตอนมาตรฐานของท้องถิ่น
- **แนวทางที่ 2 (Re-link to Verified Citizen):** หากตรวจสอบพบว่าประชาชนตัวจริงมีข้อมูลอยู่ในตาราง `demographic` อยู่แล้วภายใต้ `demo_id` อื่น (ยืนยันจากเลขบัตรประชาชน `id_no` หรือข้อมูลโฉนด/ทะเบียนบ้าน) ให้อนุมัติทำการ Re-link ปรับปรุงค่า `water_fee_detail.demo_id` หรือ `garbage_fee_detail.demo_id` ให้ชี้ไปยัง `demo_id` ของบุคคลตัวจริง
- **แนวทางที่ 3 (Application Hardening):** ทำการแก้ไขโค้ดโปรแกรมในระดับ Application (Frontend ใส่ Optional Chaining `?.` และ Backend ปรับปรุง Null-safe DTO Mapping) เพื่อให้หน้าจอไม่แครช (Graceful Handling) เมื่อพบข้อมูลประชากรเป็น Null
- **สิ่งที่เป็นข้อห้ามเด็ดขาด:** **ห้ามใช้คำสั่ง DML `INSERT INTO demographic` ด้วยข้อมูลบุคคลจำลอง เพื่อปลดบล็อกหน้าจอโดยเด็ดขาด**

#### PBR-07 Verification Checklist (วิธีการตรวจสอบความปลอดภัย)
1. **ตรวจสอบสคริปต์ Playbook (`*.sql`):** ยืนยันว่าไม่มีคำสั่ง `INSERT INTO public.demographic` หรือคำสั่งสร้าง record จำลองในตาราง `demographic` อยู่ในสคริปต์
2. **ตรวจสอบกรณีพบคีย์กำพร้า:** หากพบ `demo_id` ไม่มีใน `demographic` ให้บันทึกเป็นข้อจำกัด (Blocker) และปฏิบัติตามแนวทาง Official Import, Re-link, หรือ Code Hardening เท่านั้น
3. **PBR-07 Decision Gate:**
   - **PASS:** ไม่มีการแทรกหรือสร้างข้อมูลบุคคลเทียมลงในฐานข้อมูล
   - **FAIL / HALT:** มีความพยายามที่จะ INSERT ข้อมูลบุคคลสังเคราะห์เข้าสู่ `public.demographic` สคริปต์ต้องหยุดการทำงานทันที
