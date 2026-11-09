# Playbook Business Rules: Smart Utilities (`su-sk`)

> **File:** `playbook-business-rules.md`  
> **Scope:** Smart Utilities (`su-sk`) Production Business Logic & Domain Invariants  
> **Contract:** Referencing this rules file mandates that the playbook and execution **MUST strictly follow all rule-ids defined below**. No business rule may be bypassed.

---

## Business Rule Inventory (Consolidated)

The business rules below consolidate all domain invariants, payment lifecycle checks, billing model constraints, and fiscal year boundaries into 6 clear, non-redundant rule definitions:

| Rule ID | Rule Name & Short Thai Name | Severity | Business Invariant | Former Rules Merged |
|---|---|---|---|---|
| **PBR-01** | **Invoice Lifecycle & Mutation Scope (Unpaid, Unprinted & Demographic Mutability)**<br>*(สถานะบิลและขอบเขตการแก้ไข: แก้ไขเฉพาะบิลที่ยังไม่จ่ายและยังไม่พิมพ์ / ห้ามแตะต้องบิลที่จ่ายแล้ว / แก้ชื่อ-ที่อยู่ได้)* | ⛔ CRITICAL | Financial amount fields (`garbage_fee`, `payment_amount`, QR satang) are **strictly immutable** once paid (`receipt_id IS NOT NULL`) or printed (`printed_date IS NOT NULL`). Rate adjustments apply **ONLY to unprinted, unpaid bills** (`status = invoice_status_1`, `receipt_id IS NULL`, `printed_date IS NULL`). Historical paid periods with receipts must remain completely untouched. However, non-financial demographic metadata (name, address, tax ID, deed no) **CAN be updated** on printed/paid records. | `PBR-01`, `PBR-05`, `PBR-07`, `PBR-09` |
| **PBR-02** | **Billing Model Invariants: Accumulated vs Non-Accumulated (Period Editability & Due Date Extension)**<br>*(รูปแบบการคิดเงิน: แบบทบยอด vs แบบไม่ทบยอด — การแก้ไขรอบเดือนและการห้ามต่ออายุใบแจ้งยอด)* | ⛔ CRITICAL | **Accumulated billing (ค่าน้ำประปา):** rolling arrears; updates restricted strictly to **current active month only** (cannot modify past months); **NEVER extend due date (`payment_due_date`)** because QR code is stateless and bank checks only expiry + paid status, causing double/overlapped payments.<br>**Non-accumulated billing (ค่าขยะ):** discrete month-by-month; any unpaid unprinted month can be updated; due date extension is permitted. | `PBR-02`, `PBR-10`, `PBR-12` |
| **PBR-03** | **Fiscal Year Cycle & Cross-Year Boundary Lock**<br>*(รอบปีงบประมาณ ต.ค.–ก.ย., การคำนวณ tax_year, และการห้ามแก้ไข/โยกจ่ายข้ามปีงบประมาณ)* | ⛔ CRITICAL | Fiscal year ends **September 30**. Rounds in Oct–Dec have `tax_year = round_year + 1`; rounds in Jan–Sep have `tax_year = round_year`. Public revenues and budgets cannot cross fiscal years. Amounts within a fiscal year (e.g. September) can be modified, but **CANNOT be modified, canceled, or shifted to be paid across into October (the next fiscal year)**. In multi-round adjustments, each round must be pinned strictly to its own `tax_year`. | `PBR-06`, `PBR-11` |
| **PBR-04** | **Multi-Tenant Municipality Payer Disambiguation & Strict Scope Confinement**<br>*(ล็อก Entity UUID และจำกัดขอบเขตตามเทศบาลที่ระบุอย่างเคร่งครัด: ห้ามทำข้ามเทศบาล)* | ⛔ CRITICAL | Customer payer numbers (e.g. `68000977`) are shared across municipalities. All queries and updates must resolve and pin the unique entity UUID (`garbage_id` / `water_id`) and row primary keys. **Municipality Scope Confinement:** When the Jira card assigns a municipality (e.g. `[เกาะสมุย]`), the entire scope of work, investigation, verification (PBR-01..PBR-06), and correction is strictly confined within that municipality. Never evaluate, inspect, or mutate data across other municipalities. | `PBR-03` |
| **PBR-05** | **Multi-Channel Satang Synchronization**<br>*(ซิงค์ยอดสตางค์ QR/Barcode ให้ตรงกับค่าธรรมเนียม)* | ⚠️ HIGH | Changes to invoice or transaction fees require atomic synchronization of PromptPay QR and barcode payloads in satang (`garbage_fee_qrcode.qr_code_text`, `barcode_text`) in the same transaction, ensuring payment gateways charge the exact business rate. | `PBR-04` |
| **PBR-06** | **Unpaid Invoice Cancellation by Guarded Deletion**<br>*(ยกเลิกบิลที่ยังไม่ชำระด้วยการลบ / ห้ามลบบิลที่รับชำระแล้ว)* | ⛔ CRITICAL | Cancel verified unpaid target invoices by **DELETE of exact invoice rows and their exclusive QR/barcode rows and mapping links**; future due dates alone do not block cancellation, with live payment/printing/dependency checks, scoped row-count guards, offline backup, recoverable transaction and explicit per-action approval. Overdue does not mean paid. Never delete paid invoices, receipts or bank records, and never use `invoice_status_2` as a guessed cancellation state. | `PBR-08` |

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

### PBR-06: Unpaid Invoice Cancellation by Guarded Deletion (Never Delete Paid Invoices)
- **ชื่อภาษาไทยสั้น (Short Thai Name):** ยกเลิกใบแจ้งหนี้ที่ยังไม่ชำระด้วยการลบข้อมูลอย่างมีเงื่อนไข / ห้ามลบบิลที่รับชำระแล้ว
- **Human clarification — 9 November 2026:** The project owner clarified: "since it was not paid yet, you can cancel it. To cancellation the invoice is to delete it from the database." The user then requested that this method be added to the business rules. This defines the cancellation method; it does not remove the guards below or constitute standing approval for future production mutations.

For amount updates, status1 is the verified eligible state; other statuses are not automatically proof of payment. Cancellation eligibility is evaluated separately below, including the verified overdue state.

#### Cancellation method
1. **Cancel a verified unpaid invoice by deleting its exact invoice row (`DELETE`).** Do not simulate cancellation by changing an invoice to a guessed status. In particular, `invoice_status_2` is the printed/awaiting-payment state in the verified garbage workflow, not a canceled state. Do not require a nonexistent canceled enum value or change the schema to manufacture one.
2. **An overdue invoice is not automatically paid.** For the verified garbage workflow, `invoice_status_1` (awaiting printing) and `invoice_status_4` (overdue) can be considered for deletion only after all unpaid/unprinted and relationship checks pass. Verify status meanings against the actual utility/schema; do not reuse these meanings blindly for another workflow. All other PBR-01..PBR-05 requirements, including printing, accumulated billing and fiscal-year limits, continue to apply.
3. **Remove the target invoice and its exclusive QR/barcode data in the same transaction.** Human clarification on 9 November 2026: "Add to rules not to blocked by future due dates, by removing invoice all related QR and barcode will be removed too." For the verified garbage schema, both payloads are columns of `garbage_fee_qrcode`; include the exact associated `garbage_fee_qrcode_transaction` mapping rows. Verify ownership and exclusivity before deleting: a QR shared with another invoice, transaction or paid accounting record is a hard stop. This method does not authorize deleting financial transactions, receipt links, receipts, bank items, payer accounts or master/configuration records.
4. **A future payment due date alone is not a cancellation blocker.** Do not require the date to expire or introduce a separate readiness hold solely because it is in the future. Apply the unpaid/unprinted/payment and exclusivity checks, then remove the invoice, its exclusive mapping links and the QR row containing both QR and barcode payloads. Do not extend or rewrite due dates as part of cancellation.
5. **Do not assume a database cascade.** Inspect the actual schema, triggers and references. When no cascade exists, explicitly delete the pinned invoice, mapping and QR rows in a safe order, assert each expected count and restore them together if the transaction fails. Do not delete unrelated data or leave a target QR/barcode row behind.

#### Mandatory payment and accounting guards
- **Never delete a paid invoice or its accounting evidence.** Halt if the target has a paid status, `receipt_id`, a linked receipt/payment transaction, or matching collected/in-flight bank payment evidence. A statement that a bill is unpaid cannot override contradictory live records.
- Resolve the municipality/system mapping, unique payer entity and exact invoice primary keys before inspecting or deleting data. Confine every query, assertion and mutation to that scope (PBR-04); include exact periods and fiscal years.
- Check live invoice status, `receipt_id`, printing state, receipt-transaction relationships and bank references. Use the actual schema and the invoice's real payment references; do not query an invented `ktb_transaction_item.status` or guess reference mappings. For the verified garbage schema, the KTB identifiers include quoted `"SID"` and `"Ref_1"`; no `status` column exists.
- A zero receipt/bank match proves only the scoped database lookup. A mention of manual receipts is not by itself proof that the system invoice is paid. Record the reporter's explanation and verify that the approved deletion concerns unpaid/duplicate system invoice rows, while preserving manual-receipt/payment accounting. If contradictory or unresolved evidence makes the target's unpaid state uncertain, halt for reconciliation.
- Do not require a preliminary `invoice_status_2` update or voided receipt for a target that has no system receipt: `invoice_status_2` is not cancellation, and a nonexistent receipt needs no receipt mutation. Any existing payment/receipt relationship remains a hard stop, not an invitation to delete or void it automatically.

#### Required preparation and execution contract
1. Export a complete immutable offline before-state for each target invoice, exclusive QR/barcode row and mapping link, plus the preserved financial transactions and relevant relationships, with timestamp and checksum. Verify that the saved rows can be restored; do not create backup tables or other DDL (PAR-01).
2. Prepare a guarded transaction with exact primary-key + municipality/entity predicates, approved periods, expected row count, unpaid/unprinted/payment assertions and before-state drift checks. Fail on missing/extra/changed targets. Preserve fiscal years, all non-target records and financial reconciliation.
3. Execute deletion only after explicit approval for that exact action, target and scope, within one rollback-capable transaction. This includes rollback-ending dry-runs and every use of `--confirm-write`; a rule change or playbook selection is not production consent (framework Rule 4 / PAR-02..PAR-04).
4. Capture deleted invoice IDs, QR IDs, mapping keys and each row count, with in-transaction verification. A mismatch or error must roll back. Post-commit read-only checks must show exactly the approved invoices, their exclusive mappings and QR/barcode records absent, and financial transactions, receipts, bank evidence and unrelated records preserved. Check payments by the saved original reference values even after QR deletion; do not join only surviving invoices or QR rows. Future due dates are informational, not a readiness failure. Record the verified database outcome accurately: deleting database payloads does not empirically prove that previously saved external QR images have been erased or rejected by a bank. This external limitation alone does not add a future-due-date gate; contradictory collected/in-flight payment evidence remains a hard stop.
5. Prepare a separately gated DML restore from the offline snapshot, with primary-key, reference and relationship conflict guards. Restore complete original QR rows, mapping links and invoice rows in dependency order in one transaction; preserve original values, nulls and timestamps rather than recreating financial history from assumptions. Any post-commit DML restore requires fresh exact-action approval. Automatic transaction rollback on an error is part of the approved transaction and must proceed immediately; it never waits for a new approval.
6. Record the decision, before-state evidence, reviewed SQL checksum, affected IDs/counts and actual execution/rollback state in the incident workspace and Jira. Keep draft, dry-run and committed outcomes distinct.

#### PBR-06 decision gate
- **PASS for deletion preparation:** the cancellation method and exact invoice plus exclusive QR/barcode/mapping scope are established; current evidence supports unpaid/unprinted eligibility; dependencies, backup/recovery and guarded SQL are verified. Explicit approval and runtime evidence remain separate execution gates.
- **HOLD:** any payment/printing/identity/relationship/accounting uncertainty (excluding a future due date alone), a row-count or before-state mismatch, an unreviewed scope expansion, or a missing recoverable backup. A paid-record finding is a hard block.
- **Cancellation completed:** the separately approved deletion committed and read-only verification proves the exact intended outcome. Neither registration nor an offline review is execution evidence.
