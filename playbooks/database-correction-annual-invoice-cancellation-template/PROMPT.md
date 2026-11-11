# Prepare annual invoice cancellation

```text
Use playbook database-correction.annual-invoice-cancellation from project su-sk.
card: <JIRA_KEY>
jira_env: su
database_env: smart_utilities_prod
stage: prepare
operation: annual-cancellation
user_guide: Follow the annual payment cancellation procedure, verify recurring_type_4 and the full fiscal year (tax_year) scope, verify the 1 annual invoice and all linked fiscal-year charge transactions (up to 12 monthly rows), verify no electronic receipt or bank payment exists for the manual outside payment, verify canceled charges leave collection and the retained balance matches 0 (or verified prior arrears), and prepare complete reviewed backup/SQL/recovery. Preserve paid/unrelated records.
```

For standard monthly cancellations use `database-correction.invoice-cancellation`. For combined amount corrections and cancellations use `database-correction.invoice-receipt-incident`. Live stages require separate explicit human authorization.
