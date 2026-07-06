# Rasa · Module · Contracts

**Canonical name:** `rasa.module.contracts`
**Repo / folder:** `module-contracts`
**Kind:** `module` (canon Spec §6)
**Contract:** Element Contract v1.3.0
**Version:** 0.1.0 (initial shell)
**Status:** Live — spine + spec + adapter seam + ledger template. Skills deferred.

## What this is

The **contract-register** module: a lightweight inbound + outbound register of
agreements — counterparty, dates, value, status, renewals, and obligations —
each attached to a customer account by a stable `@acct-<slug>` handle. It
answers "**what agreements do we have, with whom, expiring when, and what do
they oblige us to?**" — an operational tracker, not a legal engine.

**It is NOT a legal drafting or clause-review engine.** Clause-level review,
jurisdiction logic, and conflict checks defer to `rasa.domain.legal`'s
**Matter** model, which is authoritative for legal-heavy contracts. A
contract row's `doc_ref` may link to a legal Matter, but the reasoning lives
there. This boundary is load-bearing — see `content/contracts-rules.md`.

## The record it owns — a Contract

`id` · `account` (`@acct-<slug>`) · `direction` (inbound|outbound) ·
`counterparty` · `title` · `type` (MSA/SOW/NDA/service/vendor/employment/
other) · `effective_date` · `expiry_date` · `value` · `currency` · `status` ·
`renewal` (auto/manual/none + notice_days) · `obligations` · `signed_date` ·
`doc_ref` · `notes`.

Lifecycle: `draft → negotiation → active → (renewed | expired | terminated)`.

## Element- vs project-owned

| File | Owner | Installs to | Policy |
|---|---|---|---|
| `content/contracts-rules.md` | Element | `.claude/contracts-rules.md` | file-replace (refreshed on upgrade) |
| `content/README.md`, `content/BUILD_PLAN.md` | Element | author-time docs | opt-in (not installed) |
| `seed/contracts-gate.md.template` | **Project** | `.claude/contracts-gate.md` | skip-if-exists |
| `seed/contracts/CONTRACTS.md.template` | **Project** | `contracts/CONTRACTS.md` | skip-if-exists |
| `seed/rasa.lock.json.template` | Project | `.claude/rasa.lock.json` | init-only-with-sha |

The **account spine:** every record joins to its customer by an `@acct-<slug>`
handle — the same handle used by the sibling business-ops modules (`leads`,
`schedule`, `invoices`, `field-log`), so records join across modules today and
refactor to the canonical hub FK when canon task **SA-032 (engagement-hub
primitive)** lands. This module never seeds a competing accounts registry.

The **seam** (`.claude/contracts-gate.md`) is the one per-project thing: the
contract TYPE taxonomy, the value-tiered approval thresholds, and the
obligation-tracking policy.

## Skills are deferred to a build phase

The `/contract`, `/contracts`, and `/renewals` skills are **not shipped in
v0.1.0** — they are held until a real consumer declares this module in its
`requires.elements[]`, per the RasaOS extract-after-proof precedent
(`module.tasks`, `module.notes`). See `content/BUILD_PLAN.md` (M-1..M-3).

## See also

- `~/rAI/rasa-os/elements/module-notes/` — the shell precedent this module
  follows (spine + spec + seam + ledger, skills deferred).
- `~/rAI/rasa-os/elements/domain-legal/` — the authoritative clause-level
  engine; the boundary between it and this register is load-bearing.
- Canon Spec §6 — the `module` kind + Connection Contract.
- `~/rAI/rasa-os/elements/REGISTRY.md` — the live workspace snapshot.
