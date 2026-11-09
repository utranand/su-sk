# Authoring Guide: Creating a Production Database Correction Playbook

> **Target Scope:** Smart Utilities (`su-sk`) and PostgreSQL Production Services  
> **Audience:** Operational Engineers and AI Coding Agents  
> **Standard:** Production-grade, zero-DDL safe, human-gated, and fully auditable on Jira.  
> **Governing Rules:** All playbooks **MUST strictly follow every rule declared in the governing rule files cataloged in [`PLAYBOOK_RULES.md`](./PLAYBOOK_RULES.md)**. Referencing a rules file mandates following all rule-ids within that file. No agent or engineer may bypass these rules.

This guide provides the complete, step-by-step standard operating procedure for creating a new **Incident Playbook Template** whenever a new class of production database incident arises. Every playbook must embed and enforce the mandatory rules from `rules/playbook-business-rules.md` and `rules/playbook-architect-rules.md`, including the critical **PBR-01 invariant: UPDATE only unpaid bills; if any bill was paid, halt execution and raise back to the user for decision**.

---

## 1. When to Create a New Playbook Template

Create a new playbook template when:
1. **The incident type is new:** The problem cannot be resolved by merely changing parameters in existing templates (e.g. `duplicate-invoice-cleanup` vs `invoice-amount-correction`).
2. **The affected entity tables differ:** Different root aggregates are involved (e.g. water meter readings vs garbage billing vs KTB bank reconciliation).
3. **The safety guards or verification rules differ:** Different conditions define a valid state (e.g. checking meter rollover arithmetic vs checking QR satang suffixes).

If the incident is merely the same operation in a different municipality or for a different customer, **reuse the existing template**; do not scaffold a new one.

---

## 2. Anatomy of a Completed Playbook

A completed playbook template must be self-contained in its own directory:

```
projects/su-sk/playbooks/database-correction-<issue-type>-template/
├── playbook.yaml                  # Template Registration Manifest (playbook-id, playbook-rules-file-ids)
├── README.md                      # Comprehensive Playbook SOP & Guide (with YAML frontmatter)
└── templates/                     # Reusable SQL Script Suite (0 DDL) & Reporting Templates
    ├── 01_investigate_readonly.sql.template
    ├── 02_backup.sql.template
    ├── 03_correct.sql.template
    ├── 04_verify_readonly.sql.template
    ├── 05_rollback.sql.template
    └── incident_report_th.md.template
```

### Required Files & Responsibilities

| File | Purpose | Application User Constraint |
|---|---|---|
| `playbook.yaml` | Machine-readable registration manifest declaring `playbook_id`, `issue_type_id`, name, domain, scope, and enforced `playbook_rules_file_ids`. | Registration & catalog metadata |
| `README.md` | Incident classification, keyword match rules, workspace setup, live investigation guide, Jira footprint instructions, executive incident report guide, and human sign-off gate. | Documentation & workflow specification |
| `01_investigate_readonly.sql.template` | Queries to verify entity identity, multi-municipality collision, payment state, and data consistency. | **Pure read-only** (`SELECT`) |
| `02_backup.sql.template` | Read snapshot queries and JSON export payload. | **Pure read-only** (0 DDL, no `CREATE TABLE`) |
| `03_correct.sql.template` | Guarded PL/pgSQL block asserting exact row counts, followed by primary-key pinned `UPDATE` statements and in-transaction verification. Ends in `ROLLBACK`. | **Pure DML** (`UPDATE` inside transaction) |
| `04_verify_readonly.sql.template` | Post-commit verification queries checking target values, historical un-mutated months, and blast-radius row counts. | **Pure read-only** (`SELECT`) |
| `05_rollback.sql.template` | Pure DML rollback script using hardcoded original values from Step 2. | **Pure DML** (`UPDATE` inside transaction, no backup tables) |
| `incident_report_th.md.template` | Non-technical executive summary in Thai with prominent target UUID topic and clear TL;DR of database changes without SQL scripts. | Reporting & stakeholder communication |

---

## 3. Step-by-Step Authoring Walkthrough

### Step 1: Directory Naming & Scaffolding
Name the folder using kebab-case with the specific `issue-type`:
```bash
ISSUE_TYPE="<your-issue-type>" # e.g. duplicate-invoice-cleanup
DIR="projects/su-sk/playbooks/database-correction-${ISSUE_TYPE}-template"

mkdir -p "$DIR/templates"
```

---

### Step 2: Register Playbook Metadata & Enforced Rules (in `playbook.yaml` and `README.md`)
Every playbook template must register its unique machine-readable identity and declare the standing rules it enforces:

1. **Playbook ID (`playbook_id`):** Unique dot-separated or kebab-case identifier (e.g. `database-correction.invoice-amount` or `database-correction.duplicate-invoice-cleanup`).
2. **Playbook Rules File IDs (`playbook_rules_file_ids`):** List of governing rule files under [`rules/`](./rules/) that this playbook references and enforces (e.g. `[playbook-business-rules.md, playbook-architect-rules.md]`). Rules must be referenced **by files only** using relative file names, **never by individual rule-ids**. **Contract:** Referencing a rules file mandates that the playbook strictly satisfies and enforces **all rule-ids** defined within that file (e.g. referencing `playbook-business-rules.md` enforces all `PBR-01`–`PBR-06` business logic checks, and referencing `playbook-architect-rules.md` enforces all `PAR-01`–`PAR-07` architectural and safety controls).
3. **Issue Type ID (`issue_type_id`):** Jira/incident pattern key (e.g. `invoice-amount-correction`).
4. **Issue Pattern Matching:**
   - Thai: e.g. `ใบแจ้งหนี้ซ้ำ`, `ยกเลิกใบแจ้งหนี้`, `สร้างเบิ้ล`
   - English: e.g. `duplicate invoice`, `void duplicate bill`, `cleanup batch`
5. **Core Intent:** What business outcome this fix achieves.
6. **Negative Triggers (What this is NOT):** Clearly state which related issues must land on a *different* playbook to prevent misrouting.

### Step 2.1: Stamp `playbook_id` and `playbook_rules_file_ids` into Templates
All SQL templates and markdown report templates in `templates/` must declare these identifiers in their top comment headers so generated artifacts remain traceable back to their governing rulebook.

---

### Step 3: Design the Live Read-Only Grounding (Phase 3 of SOP)
Every playbook must document the domain traps specific to this incident class:

1. **Utility Type Disambiguation:**
   - Are tables under `garbage_fee_*` or `water_fee_*`?
   - Never assume water fee by default.
2. **Multi-Tenant / Municipality ID Collision & Scope Confinement:**
   - In smart utilities, customer/payer numbers (`water_no`, `garbage_no`) are **not globally unique**. Multiple municipalities share identical numbers.
   - Always resolve to the unique primary key UUID (`garbage_id` / `water_id`) using district/province filters.
   - **Municipality Scope Confinement:** When the Jira card assigns a municipality (e.g. `[เกาะสมุย]`), the entire scope of work, live grounding, rule verification (PBR-01..PBR-06), and correction is strictly confined within that municipality. Never evaluate, inspect, or mutate data across other municipalities. All rule pass/fail criteria apply strictly to records within the assigned municipality.
3. **Payment State Verification:**
   - Are any candidate rows already paid (`status = invoice_status_3` or `receipt_id IS NOT NULL`)?
   - Paid rows must **never** be silently overwritten.
4. **Payment Channel Verification (KTB / QR):**
   - Check `ktb_transaction_item` for active or completed PromptPay payments.
   - Check if payload amounts (in satang) need synchronization.

---

### Step 4: Author the 5 SQL Templates

#### 1. `01_investigate_readonly.sql.template`
- Pinpoint the exact entity UUID.
- Query status, receipt presence, fee amounts, and transaction links.
- Check payment gateway transaction logs (`ktb_transaction_item`) for in-flight payments.

#### 2. `02_backup.sql.template`
- Provide structured `SELECT` queries across all affected tables.
- Include a `json_build_object` query that aggregates all rows into an exportable JSON payload.
- Document saving this payload locally to `<workspace>/playbook/backup_data_<date>.json`.

#### 3. `03_correct.sql.template`
- Must begin with `BEGIN;` and end with `ROLLBACK;`.
- Must contain an **Upfront Guard Assertion** using PL/pgSQL:
  ```sql
  do $$
  declare
    n_target int;
    expected_count int := <EXPECTED_COUNT>;
  begin
    select count(*) into n_target from ...;
    if n_target <> expected_count then
      raise exception 'Safety guard failed: expected %, got % — rolling back', expected_count, n_target;
    end if;
  end $$;
  ```
- Updates must be filtered by **primary key UUIDs** and entity UUID.
- Must add a traceable audit stamp in `remarks` (e.g. `remarks = '<CARD_KEY> <action>'`).
- Must run an in-transaction `SELECT` readback showing new values before the `ROLLBACK;`.

#### 4. `04_verify_readonly.sql.template`
- Read-back query asserting target rows carry the new state.
- Blast-radius assertion query counting all rows in the database with the incident's `remarks` tag.

#### 5. `05_rollback.sql.template`
- Must be **pure DML** (`UPDATE`).
- Uses hardcoded original values from Step 2 backup snapshot.
- Must NOT depend on `data_fix_*` tables.
- Pinned by exact primary key UUIDs.

---

### Step 5: Document Jira Footprint Procedures (in `README.md`)
Specify how agents or engineers must record evidence on the Jira ticket:
1. **Comment 1 (Rules Verification Footprint):** Post the Playbook Rules verification result using `rules_verification_footprint.md.template`, with short Thai rule names, status indicators, and the Decision Gate transition status:
   - **`✅` (green checkmark)** for passed rules (`[PASS]`).
   - **`❌` (red cross)** for unmet or failed rules (`[FAIL]`).
   - **Decision Gate Transition Status:** If ALL business rules pass (6/6 `✅ [PASS]`), mark as `🟢 [ELIGIBLE]` to move on to the final human decision gate; if ANY rule fails, mark as `🔴 [BLOCKED]` and halt immediately.
2. **Comment 2 (Correction SQL):** Post `03_correct.sql` in a Jira ADF syntax-highlighted code block, marked:  
   `DRAFT / PENDING HUMAN SIGN-OFF — NOT APPLIED TO DATABASE`.
3. **Comment 3 (Rollback SQL):** Post `05_rollback.sql` in an ADF code block, marked:  
   `STANDBY / CONTINGENCY PLAN — NOT APPLIED`.
4. Include the standard footprint trailer:  
   `su-sk · projects/su-sk/artifacts/runs/<slug>/jira-workspace/playbook/<file>`

---

### Step 6: Define the Human Sign-Off Gate & Execution SOP
Document the strict transition rules:
1. **The Business Rules Gate & Stop Condition:** The agent completes all read-only steps and SQL authoring, verifies all business rules, posts footprints including the verification results and gate status. **If and only if all business rules pass**, the card moves to the final human decision gate; the agent sets Jira DoR to `NOT READY` awaiting human sign-off, and stops. If any business rule fails, the agent halts immediately without proceeding to the human decision gate.
2. **The Execution Flow:**
   - Execute dry-run with `03_correct.sql` (ends in `ROLLBACK`). Verify output shows expected `UPDATE <N>` and clean readback.
   - Flip `ROLLBACK` to `COMMIT`.
   - Run with `--confirm-write`.
   - Run `04_verify_readonly.sql` and attach the result to the Jira card.

---

### Step 7: Register in Central Catalog
Add the new playbook to [`README.md`](./README.md) with:
- Issue Type ID
- Issue Type Name (Thai and English)
- Relative path to the template directory
- Brief description of the incident scope

---

## 4. Completed Playbook Checklist (Definition of Done)

Before marking a new playbook template complete, verify:

- [ ] Directory is named `database-correction-<issue-type>-template/`.
- [ ] `README.md` defines clear keyword pattern matching and negative routing rules.
- [ ] Domain-specific grounding checks (utility separation, ID collisions, payment states) are documented.
- [ ] All 5 SQL templates are created under `templates/`.
- [ ] `02_backup.sql.template` creates **zero** database tables (pure read snapshot + offline JSON).
- [ ] `03_correct.sql.template` includes an upfront count guard (`do $$ ... raise exception $$;`) and ends in `ROLLBACK`.
- [ ] `05_rollback.sql.template` is pure DML with hardcoded original values (no backup table dependency).
- [ ] Jira footprint posting instructions are documented.
- [ ] The template is registered in [`README.md`](./README.md).
