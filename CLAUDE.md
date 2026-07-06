# CLAUDE.md — `rasa.module.contracts`

Per-repo working contract for Claude sessions opened inside this folder.
Extends `~/.claude/CLAUDE.md` and the workspace `~/rAI/rasa-os/CLAUDE.md`
(the `rasa.tenant.rasaos` tenant's contract); does not override them.

## What you are when you're in this folder

You are working on **`rasa.module.contracts`** — a `module`-kind, business-ops
Element. It is the lightweight **contract register**: an inbound + outbound
register of agreements (counterparty, dates, value, status, renewals,
obligations) attached to a customer account. One of the six business-ops
modules (`leads`, `schedule`, `contracts`, `invoices`, `field-log`) that share
an account spine.

It ships a portable spine (`content/contracts-rules.md`) + a per-project
adapter seam (`.claude/contracts-gate.md` — the contract TYPE taxonomy,
approval thresholds, obligation policy) + a live ledger (`contracts/
CONTRACTS.md`). Toolkit shape.

**Critical boundary:** this is **not** a legal drafting or clause-review
engine. Clause-level review, jurisdiction logic, and conflict checks defer
(soft ref) to `rasa.domain.legal`'s **Matter** model, authoritative for
legal-heavy contracts. Stay a tracker — do not reinvent domain-legal here. If
a session starts drifting into clause reasoning, that is the wrong module.

### The account spine

Every Contract references its customer by a stable `@acct-<slug>` handle — an
opaque join key shared across the six business-ops modules. This module never
invents or mutates the account's own record; the account hub is a future canon
primitive (**SA-032 — engagement-hub**). Until it lands, treat the handle as
the join key and do not seed a competing accounts registry here.

### Skills are deferred (extract-after-proof)

v0.1.0 is a **shell**: spine + spec + seam + ledger template, **no skills**.
The `/contract`, `/contracts`, and `/renewals` skills (see
`content/BUILD_PLAN.md` M-1..M-3) are held until a real consumer declares this
module in its `requires.elements[]`, per the `module.tasks` / `module.notes`
precedent. Don't author them ahead of that gate.

## Status

**v0.1.0 — initial shell.** Spine + spec + adapter seam + ledger template;
skills deferred. See CHANGELOG.md.

## Source of truth

- **`~/rAI/rasa-os/canon/`** — authoritative for every architectural
  decision (Spec §6 defines the `module` kind). Canon wins.
- **`elements/module-notes/`** — the module-shell precedent (spine + spec +
  seam + ledger, skills deferred). Shape questions go there, not here.
- **`rasa.json`** — this Element's formal declaration.
- **`~/rAI/rasa-os/elements/REGISTRY.md`** — the live workspace snapshot.

## Don'ts

- **You are NOT the template.** If this contract ever describes
  `domain-core` (or any other Element), the template-CLAUDE.md
  drift class is back — flag it.
- **Don't author content ahead of the authoring phase** without the user
  driving.
- **Don't `bin/init` this Element into itself.** `content/` is the
  source (workspace rule).
- **Don't push to GitHub from the Cowork sandbox.** Local commit + tag;
  the user pushes (workspace rule).

## How a version bump works

Each bump: edit `VERSION` + `rasa.json#version`, write a CHANGELOG entry,
run `bin/check-manifest`, commit + tag `v<version>`. Update
`~/rAI/rasa-os/elements/REGISTRY.md` +
`~/rAI/rasa-os/elements/CHANGELOG.md` (track #2).
