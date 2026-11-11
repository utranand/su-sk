# Prepare water meter reading volume correction

```text
Use playbook database-correction.water-meter-reading from project su-sk.
card: <JIRA_KEY>
jira_env: su
database_env: smart_utilities_prod
stage: prepare
operation: water-meter-reading
user_guide: Follow the water meter reading volume correction procedure, resolve the municipality SID and unique water_id, verify sequential meter reading chain across preceding (paid), target, and subsequent (unread) rounds, update previous_volume_usage, current_volume_usage, and total_volume_usage on water_fee_transaction, recalculate water usage fees and VAT, realign the subsequent round's starting reading, record meter broken flags/remarks, and prepare complete reviewed backup/SQL/recovery.
```

For invoice deletion and recreation use `database-correction.water-invoice-recreation`. Live stages require separate explicit human authorization.
