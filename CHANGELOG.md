# CHANGELOG — `rasa.module.contracts`

Reverse-chronological. Each entry is a version bump.

---

## 0.1.1 — 2026-07-09

### `parent_kind` → `[domain, tenant]` (canon SA-023)

- The `orchestrator` kind was folded into `tenant`; this module now mounts into a tenant or a domain (`requires.parent_kind: ["domain", "tenant"]`, was `["domain", "orchestrator"]`).

## 0.1.0 — 2026-07-05

Initial shell — the **contract-register** business-ops module.

Ships:

- **The spine** (`content/contracts-rules.md`) — the Contract record and its
  canonical field set, the `draft → negotiation → active → (renewed | expired
  | terminated)` lifecycle, the renewal-tracking discipline (surface the
  action window; never auto-renew, auto-cancel, or send), the `@acct-<slug>`
  account handle (shared across the six business-ops modules), the
  load-bearing boundary against `rasa.domain.legal`'s Matter model, and the
  ledger conventions.
- **The specification** (`content/BUILD_PLAN.md`) — entity model, lifecycle,
  the adapter seam, the deferred skills as milestones, soft cross-refs, and an
  explicit honest-scope/deferred section.
- **The adapter seam** (`seed/contracts-gate.md.template` → `.claude/
  contracts-gate.md`, skip-if-exists) — the one per-project thing: the
  contract TYPE taxonomy, the value-tiered approval thresholds, and the
  obligation-tracking policy. Placeholder-free honest defaults.
- **The live ledger template** (`seed/contracts/CONTRACTS.md.template` →
  `contracts/CONTRACTS.md`, skip-if-exists) — project-owned, append-and-amend,
  one entry per contract.

Deliberately deferred per the RasaOS extract-after-proof precedent
(`module.tasks`, `module.notes`): the `/contract` (add/update), `/contracts`
(register view), and `/renewals` (expiring-soon report) skills. Held until a
real consumer declares the module in its `requires.elements[]`. See
`content/BUILD_PLAN.md` M-1..M-3.

`rasa.json` set: `shape_pattern: toolkit`, `role: L1-mountable-capability`,
`permissions: [fs:read, fs:write]`, capabilities `contracts.registry` /
`contracts.lifecycle` / `contracts.renewal-tracking` /
`contracts.account-linkage`. Stripped the domain-core fork scaffold
(`.claude/`, `content/SHAPE.md`, skills/rules/agents, output-style-enforcement,
the CLAUDE.md + output-style seed templates). `bin/check-manifest` GREEN.

Account spine threaded throughout; SA-032 (engagement-hub) noted as the future
canonical FK; e-signature + notice delivery noted as Connections/SA-031 soft
refs; `module.leads` (a won lead may create a contract) and `module.invoices`
(invoices bill against one) noted as soft refs.
