# Smart Utilities project workspace

Sidekicks project metadata for `su-sk`. Service source repositories are independent and acquired on demand; this repository records their locations and current source revisions.

## Registered services

- `smart-utilities-api`
- `smart-utilities-citizen-api`
- `smart-utilities-frontend`
- [`smart-utilities-local-admin`](https://github.com/utranand/smart-utilities-local-admin)

The local Admin service is a personal project with its own UX/design handoff and no Jira-card assignment. Its application implementation has not started.

## Repository contents

The initial project commits contain the project manifest/index, service registration metadata and existing project-behavior documentation. Existing operational playbooks and non-secret configuration can be versioned separately after their own content review.

Credentials, legacy local configuration, environment files, local knowledge/memory, database backups, incident execution artifacts and service source checkouts stay local. Public configuration structure may be committed in `config/`; credentials belong in ignored `*.secret.yaml` files. Never publish raw incident snapshots or production data.

No database operation, production deployment or application implementation is performed by initializing these repositories.
