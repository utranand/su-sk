# Prepare water invoice removal and recreation

```text
Use playbook database-correction.water-invoice-recreation from project su-sk.
card: <JIRA_KEY>
jira_env: su
database_env: smart_utilities_prod
stage: prepare
operation: water-invoice-recreation
user_guide: Follow the water invoice removal and recreation procedure, resolve the municipality SID and unique water_id, verify target invoices in water_fee_invoice are unpaid and unprinted with no electronic receipts, confirm underlying water_fee_transaction rows already carry corrected volumes and fees, delete old invoice-transaction mapping links and unpaid invoice rows, insert fresh water_fee_invoice rows with synchronized PromptPay QR payloads and references, recreate mapping links to water transactions and integrated garbage transactions, and prepare complete reviewed backup/SQL/recovery.
```

For meter reading volume adjustments on transactions use `database-correction.water-meter-reading`. Live stages require separate explicit human authorization.
