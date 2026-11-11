# Prepare water meter reading and invoice correction

```text
Use playbook database-correction.water-meter-invoice from project su-sk.
card: <JIRA_KEY>
jira_env: su
database_env: smart_utilities_prod
stage: prepare
operation: water-meter-invoice
user_guide: Follow the water meter reading and invoice correction procedure, resolve the municipality SID and unique water_id, verify sequential meter reading chain across preceding (paid), target, and subsequent (unread) rounds, verify target invoices in water_fee_invoice are unpaid and unprinted with no electronic receipts, delete target unpaid invoices and invoice-transaction mapping links, update meter reading volumes and recalculate usage/VAT fees on water_fee_transaction, realign the subsequent round's starting reading, preserve integrated garbage transactions, and prepare complete reviewed backup/SQL/recovery.
```

For garbage fee amount adjustments use `database-correction.invoice-amount`. For garbage invoice cancellations use `database-correction.invoice-cancellation` or `database-correction.annual-invoice-cancellation`. Live stages require separate explicit human authorization.
