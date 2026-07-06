# `rasa.module.contracts` — content

What this module ships and where it installs. This file is author-time
documentation (not installed into consumer projects).

## The one-liner

The **contract-register** module: a lightweight inbound + outbound register
of agreements — counterparty, dates, value, status, renewals, and
obligations — attached to a customer account by a stable `@acct-<slug>`
handle. A tracker of *what you have on file and when it needs attention*.
**Not** a legal drafting or clause-review engine — that is
`rasa.domain.legal`'s Matter model (authoritative for legal-heavy contracts).

## What installs where

| Source | Installs to | Policy | What it is |
|---|---|---|---|
| `content/contracts-rules.md` | `.claude/contracts-rules.md` | file-replace | The spine — the Contract record, lifecycle, renewal tracking, the account handle, the domain-legal boundary. Element-owned; refreshed on upgrade. |
| `seed/contracts-gate.md.template` | `.claude/contracts-gate.md` | skip-if-exists | **The seam** — the project's contract TYPE taxonomy, approval thresholds, and obligation-tracking policy. Project-owned; filled by the project. |
| `seed/contracts/CONTRACTS.md.template` | `contracts/CONTRACTS.md` | skip-if-exists | The live contract ledger. Project-owned; append-and-amend, never rewritten. |
| `seed/rasa.lock.json.template` | `.claude/rasa.lock.json` | init-only-with-sha | Connection-Contract lockfile, SHA-stamped at init. |

## What is NOT here yet

The `/contract`, `/contracts`, and `/renewals` **skills** — they are the
build phase. See [`BUILD_PLAN.md`](BUILD_PLAN.md) for their contracts and the
extract-after-proof gate that holds them until a real consumer declares the
module in its `requires.elements[]`.

## The shape

Toolkit module, `requires.parent_kind: [domain, orchestrator]`. Pure
Element-layer convention — no kernel engine. Cross-refs to `domain.legal`
(clause-level review), Connections/SA-031 (e-signature + notice delivery),
`module.leads` (a won lead creates a contract), and `module.invoices`
(invoices bill against a contract) are all **soft** (present-if-mounted).

## See also

- [`BUILD_PLAN.md`](BUILD_PLAN.md) — the full spec + build plan.
- `content/contracts-rules.md` — the installed spine.
- `elements/module-notes/` — the shell precedent this module follows
  (spine + spec + seam + ledger, skills deferred).
- `elements/domain-legal/` — the authoritative clause-level engine; the
  boundary between it and this register is load-bearing.
- Canon Spec §6 — the `module` kind + Connection Contract.
- Canon `ELEMENT_CONTRACT.md` §7 — install policies.
