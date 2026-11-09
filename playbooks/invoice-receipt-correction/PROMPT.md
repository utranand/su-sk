# Run the invoice correction and cancellation playbook

Copy and replace the incident key and task-specific inputs:

```text
Use playbook database-correction.invoice-receipt-incident from project su-sk.
card: <JIRA_KEY>
jira_env: su
database_env: smart_utilities_prod
stage: prepare
operation: from-card
work_item: <reuse supplied parent feature work item, otherwise derive it>
user_guide: Ground directly in the scoped live database. Prepare exact invoice ID/action lists, full backup/recovery and a short interactive readiness checklist. Keep results concise.
```

Read the current card, parent, attachments and rules. Derive the municipality, payer, operation, periods, amounts, UUIDs and counts from this incident; no identifier/count/approval from another incident is a default. Reuse the parent's workspace. Produce the complete reviewed package and show the report.

For a later authorized run, set `stage: dry-run`, `stage: commit` or `stage: verify` and supply the current user instruction. Stage selection alone is not permission. `verify` remains read-only. Recall an already valid exact-action grant rather than asking the same question again.
