# Contracts Rules

The portable contract-register spine that `rasa.module.contracts` installs
at `.claude/contracts-rules.md`. It covers the record this module owns, the
contract lifecycle, the customer/account reference convention, the boundary
against `rasa.domain.legal` (the authoritative clause-level engine), and the
ledger conventions. **Read this file when adding or updating a contract,
tracking a renewal, checking an obligation, or asking "what do we have on
file with this counterparty?"**

This file is **Element-owned** — it refreshes on upgrade. It deliberately
does **not** decide the things that vary per project:

- the project's contract **TYPE taxonomy**, its **approval thresholds** (who
  signs off on what value), and its **obligation-tracking policy**.

Those live in the project-owned **`.claude/contracts-gate.md`** (the
contracts analogue of the task module's done-gate and the release module's
release-gate). This file references the seam; it hardcodes none of it. Any
step that needs a type list or an approval threshold reads the seam, and
**hard-stops** if the seam is unfilled — guessing what your organization's
approval ladder is is the one inference this module refuses to make.

## What this module owns — and what it deliberately does not

This is a **lightweight in/out contract register** — the operational answer
to "what agreements do we have, with whom, expiring when, and what do they
oblige us to?" It tracks:

- the existence, direction, and status of every agreement (inbound + outbound),
- its dates, value, counterparty, and the account it attaches to,
- its **renewal window** (so nothing auto-renews or lapses unnoticed), and
- its **obligations** (the recurring things the contract commits you to).

It is **NOT a legal drafting or clause-review engine.** It does not read,
draft, redline, or reason about clauses; it does not do jurisdiction logic,
conflict-of-terms checks, or governing-law analysis. For any of that, defer
to **`rasa.domain.legal`** and its **Matter** model, which is authoritative
for legal-heavy contracts. See "The boundary against domain-legal" below —
this is the load-bearing boundary of the module. When in doubt, this module
records *that* a contract exists and *when it matters*; domain-legal reasons
about *what it says*.

## The record — a Contract

One record per agreement, project-owned in the ledger (see below). The
canonical field set:

| Field | Type / form | What it is |
|---|---|---|
| `id` | `CON-NNN` monotonic | Stable identifier; never reused. |
| `account` | `@acct-<slug>` | The customer/account handle this contract attaches to (shared spine — see below). |
| `direction` | `inbound` \| `outbound` | Inbound = they provide to us (a vendor/service we consume); outbound = we provide to them (a customer agreement). |
| `counterparty` | text | The other party's name as it appears on the agreement. |
| `title` | text | Human title ("Acme MSA 2026", "Cloud hosting SOW #3"). |
| `type` | MSA \| SOW \| NDA \| service \| vendor \| employment \| other | The contract type. The project's real taxonomy lives in the gate; this is the default superset. |
| `effective_date` | `YYYY-MM-DD` | When the agreement takes effect. |
| `expiry_date` | `YYYY-MM-DD` \| none | When it ends (absent for perpetual/at-will). Drives renewal-tracking. |
| `value` | number | Contract value (total or annual — the project picks one and states it in the gate). |
| `currency` | ISO 4217 (`USD`, `EUR`, …) | Currency of `value`. |
| `status` | see lifecycle | Current lifecycle state. |
| `renewal` | `auto` \| `manual` \| `none` + `notice_days` | Renewal mode and the notice window before `expiry_date` at which action is due. |
| `obligations` | list | The recurring commitments this contract creates (e.g. "quarterly security review", "monthly report by the 5th"). Tracked, not enforced — reminders, not a workflow engine. |
| `signed_date` | `YYYY-MM-DD` \| none | When it was executed. Absent until signed. |
| `doc_ref` | path / URI | Pointer to the actual document (a file, a DMS link, a Connections e-sign envelope id). This module stores the pointer, never the document. |
| `notes` | text | Freeform. |

Dates are always **absolute** (`2026-07-05`), never relative. Money is
always a `value` + `currency` pair — never a bare number.

## Lifecycle / states

A contract moves through a small, explicit set of states:

```
draft ──▶ negotiation ──▶ active ──┬──▶ renewed
                                   ├──▶ expired
                                   └──▶ terminated
```

| State | Meaning |
|---|---|
| **draft** | Being prepared; not yet shared with the counterparty as a firm offer. |
| **negotiation** | Terms in flight with the counterparty; not yet executed. |
| **active** | Executed and in force. `signed_date` is set. |
| **renewed** | An active contract carried past its `expiry_date` into a new term (a new `CON-NNN` may supersede, or the term extends — the project's gate says which). |
| **expired** | Reached `expiry_date` without renewal or termination; no longer in force. |
| **terminated** | Ended early by one party per the agreement's terms. |

Hard rules for the lifecycle:

- **Only forward, except renewal.** A contract does not go back from
  `active` to `negotiation`; a fresh `draft` records a renegotiation.
- **`active` requires a `signed_date`.** A contract with no signature is at
  most `negotiation`.
- **Terminal states are terminal.** `expired` and `terminated` are not
  edited back to `active` — a new contract is filed instead.
- **The ledger is append-and-amend, not rewrite-history.** Status changes
  update the record's `status` in place; the fact that it changed is
  captured by the change itself + the git history of the ledger. Never
  delete a contract row — mark it `expired`/`terminated`.

## Renewal tracking

This is the module's most operational job. For every `active` contract with
an `expiry_date`:

- **`renewal: auto`** — the contract renews itself unless cancelled. The
  live signal is the **cancellation deadline** = `expiry_date − notice_days`.
  A report surfaces these *before* that deadline so a human can decide to
  let it renew or cancel in time.
- **`renewal: manual`** — the contract ends at `expiry_date` unless actively
  renewed. The action window opens at `expiry_date − notice_days`.
- **`renewal: none`** — one-shot; ends at `expiry_date`, no window.

The module surfaces "expiring / action-due soon" — it **never auto-renews,
auto-cancels, or sends anything on its own.** It is a reminder surface, not
an actor. Sending an actual renewal or termination notice is a soft
reference to **Connections (SA-031)**, not something this module does.

## Customer / Account reference (shared spine)

> **Customer / Account reference (shared spine).** Every record this module
> owns references its customer by a stable **account handle**:
> `account: @acct-<slug>` in the record's frontmatter (for example
> `@acct-acme`). The handle is an opaque, stable join key. This module **never
> invents or mutates the account's own record** — the account's name,
> contacts, address, and billing details live on the **account hub**, which
> RasaOS does not yet have as a first-class primitive (tracked as canon task
> **SA-032 — engagement-hub primitive**). Until the hub lands, treat the
> handle as the join key and, if a project-owned account ledger is mounted,
> resolve display names from it; **do not seed a competing accounts registry
> here.** Every sibling business-ops module (`leads`, `schedule`, `contracts`,
> `invoices`, `field-log`) uses this same `@acct-<slug>` handle, so records
> join across modules today by shared key and refactor to the canonical hub
> FK — with no change to the records themselves — when SA-032 resolves.

## The boundary against domain-legal — read this before you reach for the wrong tool

This module and `rasa.domain.legal` are **different primitives**, and
conflating them is the known failure mode:

- **This module (`contracts`)** = the lightweight operational register.
  Its unit is *a contract as a row*: it exists, it's inbound or outbound,
  it's worth X, it expires on date Y, it renews in mode Z, it obliges us to
  these recurring things. It answers "**what do we have on file, and when
  does it need attention?**" It never reads the clauses.
- **`rasa.domain.legal` / the Matter model** = the authoritative legal
  engine. Its unit is *a matter*: clause-level review, redlines,
  jurisdiction and governing-law logic, conflict checks, negotiation
  strategy. It answers "**what does this agreement actually say, and is it
  acceptable?**"

Rule of thumb:

- Tracking that a vendor MSA renews in 30 days → **this module**.
- Reviewing whether the MSA's indemnity clause is acceptable → **defer to
  `domain.legal`'s Matter model** (it is authoritative for legal-heavy
  contracts).
- A `doc_ref` on a contract row may point at a `domain.legal` Matter when
  one exists — that is the seam between the register and the legal engine.

Do not reimplement clause review here. If a project needs it and
`domain.legal` is not mounted, the honest answer is "this module does not do
that" — not a half-built clause checker.

## Soft cross-references (degrade gracefully)

This module stands alone, but plays well with siblings when mounted. All of
these are **soft** — present-if-mounted, graceful-if-absent. Never harden
any into a `requires.elements[]` dependency.

- **`rasa.domain.legal`** — authoritative for clause-level review /
  jurisdiction / conflicts (see the boundary above). A `doc_ref` may link a
  contract row to a legal Matter.
- **Connections (SA-031)** — e-signature and notice delivery. Executing a
  signature or sending a renewal/termination notice is a Connections
  concern; this module records the resulting `signed_date` / `doc_ref`, it
  does not send.
- **`rasa.module.leads`** — a won lead may *create* a contract (the
  handoff from pipeline to agreement). The lead id can be noted in the
  contract's `notes`; the modules join on the shared `@acct-<slug>` handle.
- **`rasa.module.invoices`** — invoices *bill against* a contract. An
  invoice references the `CON-NNN` it draws on; the two join on both the
  contract id and the shared account handle.

## Ledger conventions

- One `contracts/CONTRACTS.md`, project-owned (seeded skip-if-exists),
  git-versioned. One row/record per contract.
- Markdown, human-first. The ledger must read cleanly top-to-bottom to a
  person — an operator scanning "what's expiring this quarter" is the
  primary reader, not a query engine.
- `CON-NNN` monotonic, never reused; dates absolute; money as value +
  currency.
- Never delete a contract — mark it `expired`/`terminated`. The register's
  worth is that the history of what you agreed to survives.
- The **document itself never lives here** — only its `doc_ref` pointer.
  This module tracks agreements; it does not store them.

## The seam — `.claude/contracts-gate.md`

The one per-project thing. It holds what genuinely varies per organization:

1. **Contract TYPE taxonomy** — the project's real list of contract types
   (which may be narrower or wider than the default MSA/SOW/NDA/service/
   vendor/employment/other).
2. **Approval thresholds** — who approves a contract at what value (e.g.
   "< $10k: any manager; $10k–$100k: department head; > $100k: legal +
   exec"). The register records the state; the gate says who may move it to
   `active`.
3. **Obligation-tracking policy** — how obligations are surfaced and to
   whom (a cadence, an owner, an escalation), and how far ahead renewal
   windows are surfaced.

Any operation that needs a type, an approval, or an obligation cadence reads
the gate. If the gate is unfilled, the honest default applies (the default
type superset, a single "a human approves" bar, and a plain "surface
expiring-within-notice_days" renewal report) — but running on the default is
a smell, not a destination. Fill the gate.
