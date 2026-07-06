# `rasa.module.contracts` — Specification & Build Plan

**Status:** v0.1.0 = spine + spec + seam + ledger template. The skills are
the build phase (M-1..M-3 below), deliberately gated.

This is the author-time specification. The installed spine is
`content/contracts-rules.md`; this file is the design record behind it.

---

## Why this module exists

Every business-ops vertical eventually needs to answer, without lawyering:
"**what agreements do we have, with whom, expiring when, and what do they
oblige us to?**" That is an operational register, distinct from legal
review. RasaOS already has the heavyweight legal engine — `rasa.domain.legal`
and its **Matter** model, authoritative for clause-level work — but nothing
that plays the lightweight in/out tracker role: renewals that don't lapse
unnoticed, obligations that get surfaced, a row per agreement joined to a
customer account.

`rasa.module.contracts` is that tracker. It is one of the six business-ops
modules (`leads`, `schedule`, `contracts`, `invoices`, `field-log`) that
share the `@acct-<slug>` account spine so records join across modules.

## Non-goals (hard boundaries)

- **Not a legal drafting or clause-review engine.** No clause reading,
  redlining, jurisdiction logic, conflict-of-terms checks, or governing-law
  analysis. That is `rasa.domain.legal`'s Matter model — authoritative for
  legal-heavy contracts. This module stays a register; a `doc_ref` may point
  at a legal Matter, but the reasoning lives there. This is the load-bearing
  boundary (see `contracts-rules.md` → "The boundary against domain-legal").
- **Not an e-signature or notice-delivery system.** Executing a signature or
  sending a renewal/termination notice is a Connections (SA-031) concern.
  This module records the resulting `signed_date` / `doc_ref`; it never sends.
- **Not a document store.** It holds the `doc_ref` pointer, never the
  document bytes.
- **Not a workflow engine for obligations.** Obligations are *tracked and
  surfaced* (reminders), not *enforced* (assigned/gated). Open work is
  `module.tasks` territory if the project mounts it.

---

## Entity model

One project-owned ledger + one project-owned seam.

### `contracts/CONTRACTS.md` (the ledger) — the load-bearing record

One record per agreement, canonical field set:

| Field | Type / form | Notes |
|---|---|---|
| `id` | `CON-NNN` monotonic | Never reused. |
| `account` | `@acct-<slug>` | The account handle — the shared join key across the six modules. |
| `direction` | `inbound` \| `outbound` | Inbound = vendor/service we consume; outbound = customer agreement. |
| `counterparty` | text | The other party, as on the agreement. |
| `title` | text | Human title. |
| `type` | MSA \| SOW \| NDA \| service \| vendor \| employment \| other | Default superset; real taxonomy in the gate. |
| `effective_date` | `YYYY-MM-DD` | Takes effect. |
| `expiry_date` | `YYYY-MM-DD` \| none | Ends; drives renewal-tracking. |
| `value` | number | Total or annual (project picks; states it in the gate). |
| `currency` | ISO 4217 | Currency of `value`. |
| `status` | lifecycle | See below. |
| `renewal` | `auto`\|`manual`\|`none` + `notice_days` | Renewal mode + the notice window. |
| `obligations` | list | Recurring commitments; tracked, not enforced. |
| `signed_date` | `YYYY-MM-DD` \| none | Absent until executed. |
| `doc_ref` | path / URI | Pointer to the document (or a legal Matter). Never the bytes. |
| `notes` | text | Freeform. |

Dates absolute; money as value + currency; `CON-NNN` monotonic, never reused.

### Lifecycle / states

```
draft ──▶ negotiation ──▶ active ──┬──▶ renewed
                                   ├──▶ expired
                                   └──▶ terminated
```

Forward-only except renewal; `active` requires a `signed_date`; terminal
states (`expired`, `terminated`) are terminal — file a new contract rather
than reviving one. Never delete a row; mark it terminal. Full rules in
`contracts-rules.md`.

### `.claude/contracts-gate.md` (the seam) — the one per-project thing

See "The adapter seam" below.

---

## The adapter seam — `.claude/contracts-gate.md`

The crux of the design, mirroring `module.tasks`' done-gate and
`module.releases`' release-gate. It holds the three things that genuinely
vary per organization:

1. **Contract TYPE taxonomy** — the project's real contract-type list (may
   narrow or widen the default MSA/SOW/NDA/service/vendor/employment/other).
2. **Approval thresholds** — who approves a contract at what value (the
   money-tiered approval ladder). The register records the state; the gate
   says who may move a contract to `active`.
3. **Obligation-tracking policy** — how obligations and renewal windows are
   surfaced, to whom, on what cadence, with what escalation.

Why a seam and not fixed logic: an organization's approval ladder and its
type taxonomy are exactly the part that cannot be guessed — the same reason
`module.releases` pushes "what shippable means + how we deploy" into its
gate. The horizontal core is *the register + lifecycle + renewal discipline*;
the vertical part is *your types, your approvals, your obligation cadence*.

Absent a filled gate, the honest default applies: the default type superset,
a single "a human approves" bar, and a plain "surface expiring-within-
notice_days" renewal report. Running on the default is a smell, not a
destination.

---

## Skills (the build phase — M-1..M-3)

Three skills, MVP-scoped. Each is a thin driver over the ledger + seam; the
discipline lives in `contracts-rules.md`, not duplicated per skill. **None of
these ship in v0.1.0** — they are held until a real consumer declares the
module (see the gate below).

### M-1 — `/contract` (add / update)
- **`/contract add`** — file a new Contract row in `contracts/CONTRACTS.md`.
  Collects the record fields; assigns the next `CON-NNN`; validates `type`
  against the gate's taxonomy; requires an `account` handle. A contract with
  no `signed_date` cannot be filed `active` (at most `negotiation`).
- **`/contract update CON-NNN`** — advance status (respecting forward-only +
  the renewal branch), amend fields, attach a `doc_ref` / `signed_date`.
  Enforces the approval threshold from the gate before moving to `active`.

### M-2 — `/contracts` (register view)
- **`/contracts [active|expiring|inbound|outbound|@acct-<slug>|search <term>]`**
  — read/render the ledger. Default: active contracts grouped by account.
  Filters by state, direction, account handle, or free text. Read-only.

### M-3 — `/renewals` (expiring-soon report)
- **`/renewals [<horizon>]`** — the renewal-tracking report. Lists `active`
  contracts whose action window is open — for `auto`, the cancellation
  deadline (`expiry_date − notice_days`); for `manual`, the renewal window.
  Surfaces only; **never auto-renews, auto-cancels, or sends** (delivery is
  Connections/SA-031). Reads the gate's obligation/renewal-surfacing policy.

**Style/quality bar:** match the `module.releases` / `module.notes` SKILL.md
files — a crisp operation list, honest hard-stop behavior (refuse to file
`active` without a signature; refuse to approve above threshold without the
gate's approver), no invented plumbing. Fan-out plan: author `/contract` as
the reference skill, then `/contracts` + `/renewals` in parallel against it,
gate each independently.

---

## The gate — when to build the skills

Per the RasaOS extract-after-proof precedent (`module.tasks`, `module.notes`),
hold the skill build until **one real consumer** wants it — a domain or
orchestrator that adds `rasa.module.contracts` to its `requires.elements[]`.
The natural first consumers:

- an **orchestrator** running a real business's operations (the six
  business-ops modules together are the "run the back office" set).
- **`rasa.domain.legal`** as a *soft* neighbor — not to absorb this module,
  but to prove the boundary: the register tracks, the Matter model reviews,
  and a `doc_ref` links the two. If that seam survives a real legal-heavy
  contract without the register drifting into clause logic, the boundary
  holds.

Building the skills before a real consumer would repeat the premature-
abstraction trap the deferred modules exist to avoid.

---

## Soft cross-references

All soft — present-if-mounted, graceful-if-absent, never hardened into
`requires.elements[]`:

- **`rasa.domain.legal`** — authoritative clause-level engine (the boundary).
- **Connections (SA-031)** — e-signature + notice delivery.
- **`rasa.module.leads`** — a won lead may create a contract.
- **`rasa.module.invoices`** — invoices bill against a contract (`CON-NNN`).

All six business-ops modules join on the shared `@acct-<slug>` account
handle (see `contracts-rules.md` → "Customer / Account reference"), and
refactor to the canonical hub FK — with no change to the records — when canon
task **SA-032 (engagement-hub primitive)** resolves.

---

## HONEST SCOPE / deferred

- **v0.1.0 (this)** ships the spine (`contracts-rules.md`), the spec (this
  file), the seam template, and the ledger template. **No skills.** No
  kernel engine. Local only — not pushed.
- **Deferred to v0.2.0**, gated on a real consumer declaring the module:
  the `/contract`, `/contracts`, `/renewals` skills (M-1..M-3), `bin/init`
  smoke-tested into that consumer.
- **Explicitly out of scope, permanently:** clause review, jurisdiction
  logic, conflict checks (→ `domain.legal`); e-signature + notice delivery
  (→ Connections/SA-031); document storage (this module holds pointers).
- **Depends on nothing built:** the SA-032 engagement-hub is not yet a canon
  primitive; until it lands, the `@acct-<slug>` handle is the join key and
  this module does not seed a competing accounts registry.

---

## Version plan

- **v0.1.0 (this)** — spine + spec + seam template + ledger template. No
  skills. Local.
- **v0.2.0** — M-1..M-3 skills authored, once a consumer declares the
  module. `bin/init` smoke-tested into that consumer.
- **v1.0.0** — the seam format + record shape locked after the register has
  run in at least one real business-ops deployment unchanged.
