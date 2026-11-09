# Prepare invoice cancellation

```text
Use playbook database-correction.invoice-cancellation from project su-sk.
card: <JIRA_KEY>
jira_env: su
database_env: smart_utilities_prod
stage: prepare
operation: cancellation
user_guide: Follow the canonical incident procedure, derive current IDs/counts/amounts from scoped live evidence, and show a concise ID/action checklist and complete reviewed SQL/recovery package.
```

For combined corrections/cancellations use database-correction.invoice-receipt-incident. A later live stage requires its own current exact-action authorization; no source template is a runnable fix or consent.
