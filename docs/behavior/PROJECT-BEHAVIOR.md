# Project behaviour — su-sk

> **DERIVED** by `sk-forge-project-behavior` from the sources in the appendix. Do not hand-edit
> this file outside a `<!-- human:start -->` … `<!-- human:end -->` block — a refresh preserves
> those verbatim and rewrites everything else. Nothing is ever silently dropped: an entry leaves
> the active sections only through `behavior.py retire <id> --reason …`, which moves it to
> **Retired** below. To file a new one: `behavior.py complain "<what went wrong>" --domain <d>`.


_Forged at commit `a10d82e5` — 1 SSOT statements, 0 TTSR entries, 2 sources._

## Top TTSR

The entries most likely to bite an implementation run. Read these before the first write; the full register is sectioned below.

_None recorded yet._

## 0. Hard floor — inherited, non-negotiable

Inherited from the repo-root `AGENTS.md` and the framework enable map. These are **pointers, not copies** — read the named section for the binding text. No entry below may weaken one.

- **Rule 1** — CLI Write Surface · _AGENTS.md § The Six Boundary Rules_
- **Rule 2** — Agent Free-Write Surface · _AGENTS.md § The Six Boundary Rules_
- **Rule 3** — Skills Canonical Location · _AGENTS.md § The Six Boundary Rules_
- **Rule 4** — Database Write Safety (Mandatory Permission & Transaction) · _AGENTS.md § The Six Boundary Rules_
- **Rule 5** — Empirical Verification (No Assumptions) · _AGENTS.md § The Six Boundary Rules_
- **Rule 6** — Multi-CLI Parity (instructions canonical at AGENTS.md) · _AGENTS.md § The Six Boundary Rules_
- **hard rule** — Irreversible / outward-facing actions · _AGENTS.md § Operational Rules_
- **hard rule** — Protected branches — explicit waiver required · _AGENTS.md § Operational Rules_
- **hard rule** — Production access — direct only, never Teleport · _AGENTS.md § Operational Rules_

## 1. Architecture & code

### SSOT

_None recorded._

### TTSR — things to stop repeating

_None recorded._

## 2. Implementation & delivery

### SSOT

_None recorded._

### TTSR — things to stop repeating

_None recorded._

## 3. Database & data

### SSOT

_None recorded._

### TTSR — things to stop repeating

_None recorded._

## 4. Jira & cards

### SSOT

- **SSOT-jira-6e066c36** — KTB feature flow: same feature branch into ktb-development for verification, then MR into ktb-main
  - **blocking** · _memory:su-ktb-feature-merge-process_

### TTSR — things to stop repeating

_None recorded._

## 5. Deploy & cluster

### SSOT

_None recorded._

### TTSR — things to stop repeating

_None recorded._

## 6. Framework & safety

### SSOT

_None recorded._

### TTSR — things to stop repeating

_None recorded._

## Retired

_Nothing retired._

## Appendix — source ledger

Every statement above traces to one of these. A hash change is what makes this document stale.

| source | kind | sha256 |
|---|---|---|
| `.sidekicks/memory/store/projects/su-sk/su-ktb-feature-merge-process.md` | memory | `6d49402605e6` |
| `AGENTS.md` | floor | `7060de45a61f` |
