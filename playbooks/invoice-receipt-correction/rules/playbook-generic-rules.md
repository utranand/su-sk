# Generic playbook rules

These apply to every scaffolded playbook unless replaced by domain rules.

## GPR-01 — Follow the folder
An agent or skill executing this playbook MUST follow `RUN.md` and `playbook.yaml` phases in order. Inventing alternate steps is a violation.

## GPR-02 — Rules files are binding
Every file listed in `playbook_rules_file_ids` is binding in full. Individual rule-ids inside those files are not optional.

## GPR-03 — Human gate before mutation
When `human_gate.required` is true, no mutating phase may run until a human explicitly approves.

## GPR-04 — Evidence over claims
Self-reported success is not enough. Capture real outputs, exit codes, and artifact paths declared under `evidence`.

## GPR-05 — Portable paths only
Templates and declared files use relative portable paths. Absolute host paths and symlinks inside the playbook folder are refused.
