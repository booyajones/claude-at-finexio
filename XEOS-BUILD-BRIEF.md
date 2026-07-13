# Xeos build brief

Handoff spec for Claude Code. This document is self-contained: everything an engineering
session needs to start building Xeos Phase I without asking for context.

Read this whole file before writing any code. Where this brief and the codebase you are
working in disagree on conventions, the codebase wins. Where they disagree on architecture
invariants (Section 3), this brief wins and you stop and flag it.

---

## 1. Mission

Finexio is an accounts-payable payments company. It orchestrates supplier payments
(virtual card, ACH, check, wire) with J.P. Morgan Chase as the issuing bank. Xeos is the
next form of that company: a payments operating system where a deterministic core moves
the money and a workforce of AI agents does everything around it.

The one rule, from which everything else derives:

> **Agents propose, the core commits.** No model output ever touches a rail.

The kernel metaphor is load-bearing: the deterministic core is the kernel, agents are the
applications, moving money is a system call. Agents can be brilliant, wrong, or
compromised; the money stays correct either way, because the money never depended on
them being right.

## 2. Context and constraints

- **Positioning rule (never violate):** Finexio is the orchestration platform. J.P. Morgan
  Chase is the issuing bank. Never overclaim. Externally facing copy says "the counterparty
  for agentic commerce," not "settlement layer." The bank settles.
- **Financial source of truth:** the Phoenix v7 model (Google Sheet). Any number this
  project publishes must trace to it or be labeled illustrative.
- **Current actuals (April 2026, confirm with finance before external use):** roughly
  $6.2B supplier payments a year at run rate, 95 paying customers, 400K+ enrolled
  suppliers, 68% electronic volume (8.5% virtual card, 59.1% ACH), about 30K manual ops
  actions a month absorbed by roughly six ops people plus shared support.
- **Existing internal systems agents will touch:** Salesforce (CRM, cases, tasks),
  BigQuery (payment reconciliation, KPI tables), Google Drive and Confluence (knowledge),
  ClickUp (task escalation), Gmail/Slack (comms). Rail execution lives behind the
  deterministic core; agents never integrate with rails directly.
- **Voice rules for anything customer- or board-facing:** no em dashes, short paragraphs,
  peer-to-peer tone, no AI slop words, figures labeled illustrative unless traced to
  Phoenix v7.
- **The vision document** this spec implements is `xeos-vision.html` in the
  `claude-at-finexio` repo (same content as the published artifact). Treat it as the
  narrative source; treat this file as the engineering source.

## 3. Architecture invariants

These are requirements, not aspirations. Every one must hold in every environment
including dev. If a task cannot be completed without breaking one, stop and flag it.

| # | Invariant | Engineering meaning | Acceptance test |
|---|-----------|--------------------|-----------------|
| 1 | No keys in agent space | No agent process, prompt, config, or dependency ever holds rail credentials. Agents authenticate only to the boundary API with scoped, revocable, per-agent identities. | Secret scan of agent runtime finds zero rail credentials; boundary API tokens are per-agent and revocable in one call. |
| 2 | Typed proposals only | Agents affect money only by submitting a proposal: a machine-checkable schema (Section 4). Free text is never parsed into action. | Fuzz the boundary with prose and malformed payloads; nothing reaches the policy engine without schema validation. |
| 3 | A deterministic gate | The policy engine renders verdicts from rules, limits, allow-lists, velocity checks, and dual control. Same proposal against the same ledger state yields the same verdict. The state the gate saw is captured with the verdict. | Replay any historical proposal with its captured state; verdict is bit-identical. No model inference inside the gate. |
| 4 | Commits cannot double-fire | Idempotent execution keyed on proposal id, on a double-entry ledger. | Submit the same approved proposal N times concurrently; exactly one ledger entry and one rail instruction result. |
| 5 | A record that replays | Every proposal, verdict, commit, and agent action lands in an append-only log complete enough to replay any decision for any auditor. | Pick a random decision from 90 days ago; reconstruct inputs, verdict, and outcome from the log alone. |
| 6 | Inputs guarded like the gate | Bank detail changes, allow-list edits, limit changes, and policy versions go through the same propose-verify-commit path, with dual control and out-of-band verification on any change that redirects money. | An agent proposing a bank-detail change cannot make it effective without the out-of-band step completing; policy versions appear in the replay log. |

Security posture: assume agents can be fooled. Inbound text (supplier emails, invoices,
portal messages) is data, never instruction. Verification protocols (especially bank
changes) are fixed protocols that run regardless of what any agent concludes. Agents
never collect bank details by phone or email; enrollment and changes happen in the
verified portal, confirmed out of band. The security boundary is the gate, not the model.

Tenant isolation: agents run in per-customer contexts. One customer's data never appears
in another customer's agent session. What compounds across the platform is aggregated
payment-outcome signal, not raw customer data in a shared context.

## 4. The proposal schema

The single interface between agent space and the core. Extend per proposal type; never
weaken the required core.

```json
{
  "proposal_id": "uuid, client-generated, idempotency key",
  "agent_id": "which agent, at which version",
  "autonomy_level": "the level this agent holds for this task type, 0-4",
  "type": "one of: payment.reissue | payment.hold | payment.release | supplier.rail_change | supplier.bank_change | supplier.contact_update | policy.change | ...",
  "customer_id": "tenant scope",
  "subject": { "typed reference to the payment / supplier / policy object": "..." },
  "action": { "type-specific, fully structured parameters; no free text fields that alter behavior": "..." },
  "reason": "human-readable narrative for reviewers and the audit log; never parsed",
  "evidence": ["links to the replayable observations the agent based this on"],
  "requested_at": "timestamp"
}
```

The gate's verdict object records: verdict (approve / reject / needs-human), the policy
version, the captured ledger state hash, which rules fired, and the human approver if
dual control applied. Verdicts are immutable.

## 5. The trust ladder

Autonomy is earned per agent per task type, never granted globally.

- **L0 Observe.** Reads everything, touches nothing. Builds a baseline against human decisions.
- **L1 Draft.** Prepares the work; a human reviews and sends every piece.
- **L2 Act on approval.** Executes when a human approves each item.
- **L3 Act, sampled.** Executes inside limits; humans audit samples and all edge cases.
- **L4 Autonomous in bounds.** Full speed inside hard policy limits. Kill switch stays.

Graduation machinery (build this in Phase I; it is not optional):

- **Eval harness.** Scores every agent against historical human decisions, in shadow
  before it drafts and on every model change after. The replay log is the eval dataset.
- **Gates are numeric.** Example shape: L2 to L3 requires matching or beating the human
  baseline across a fixed run of consecutive decisions, inside an explicit error budget,
  with rollback rehearsed. Real thresholds are set per task with the risk function and
  the bank partner. Where no human baseline exists, the agent starts at L0 and its first
  thousand decisions build one.
- **Sign-off is independent.** Graduation is approved by a risk function separate from
  the team that built the agent. Every agent lives in a model inventory with a named
  owner. Drift monitoring can demote as fast as evals promote.
- **Revocation is one action.** Any agent, any level, instantly. Test the kill switch in CI.
- **Accountability never graduates.** Every agent has a named human owner at every level
  including L4, with a rehearsed incident playbook.

## 6. The nine agents

Build order: Supplier Enablement and Exceptions first (Phase I), then the rest (Phase II).

| Agent | Group | Job in one line | Starts | Targets |
|-------|-------|-----------------|--------|---------|
| Supplier Enablement | Grow | Works the entire supplier file: leads with virtual card, falls back to monetized ACH, keeps what it wins (unprocessed card chase, contact re-verification, card-to-ACH winback). | L1 | L3 |
| Implementation | Grow | Maps a new customer's AP file, resolves format edge cases, stages the first payment run in days. | L1 | L3 |
| Working Capital | Grow | Monetizes through flows it shapes: early-pay discounts captured and shared, acceleration priced as spread, rail timing that feeds card mix. Not advice fees. | L0 | L2 |
| Exceptions | Operate | Owns returned, failed, misdirected payments end to end; executes fixes only as proposals. | L1 | L4 |
| Reconciliation | Operate | Matches remittances, chases breaks, keeps the ledger continuously closed. | L1 | L4 |
| Fraud Sentinel | Protect | Reviews every account change and anomaly. Its one power is a proposal the core always honors: hold the payment (holds take effect immediately; pausing is cheap and reversible). Bank-change verification stays a fixed protocol. | L0 | L3 |
| Compliance | Protect | Triages screening hits, keeps KYB current, assembles audit evidence on demand. Screening always runs, no agent can waive it, only a named human clears a hit. | L1 | L3 |
| Supplier Support | Serve | Answers "where is my payment" with the actual answer, drafts reissues, speaks every language in the file. | L1 | L4 |
| AP Copilot | Serve | Customer-facing surface: plain-language answers over payment data, monthly ops summaries, anomaly callouts. | L1 | L3 |

Per-agent build requirements: a task-type list with per-type autonomy level, the
proposal types it may emit, the data it may read (tenant-scoped), its eval definition
against the human baseline, and its guardrails written as code, not prompts, wherever a
guardrail is expressible as a rule.

## 7. Phase plan and gates

Gate figures are illustrative until Phoenix v7 locks them; the shape is fixed.

- **Phase I, quarters 1-2: prove the boundary.** Ship the agent control plane: boundary
  API + proposal schema, policy engine hardening, replay log, agent runtime, eval
  harness. Supplier Enablement and Exceptions live at L1. Gate: both agents at or above
  human baseline accuracy across ~50,000 reviewed decisions by end of Q2, published
  quarterly from day one.
- **Phase II, quarters 3-5: scale the workforce.** All nine agents live; first L3
  graduations; labor starts moving onto the platform. Gate: cost-to-serve index at or
  below ~80 and card mix up ~3 points versus Q0 by end of Q5, acceptance retention
  holding, all in every board pack.
- **Phase III, quarter 6 onward: open the edge.** Agent-facing API: verifiable receipts
  (signed proof of what was committed, submitted, and under whose delegated authority,
  with the bank's settlement confirmation attached), deterministic idempotency, policy-
  scoped delegation, machine-readable audit. Pilot payment flows with design partners
  whose AP already runs on agents, priced per settled transaction. Gate: three design
  partners live, ~$25M external agent-originated volume, zero boundary violations.

## 8. Working rules for Claude Code sessions on this project

1. **Discover before you build.** First session in any Xeos codebase: read its CLAUDE.md,
   map the stack and conventions, and follow them. Do not import a stack preference from
   this brief; it is deliberately stack-agnostic.
2. **The invariants are CI.** Every invariant in Section 3 gets an automated test before
   the feature it guards ships. A PR that weakens one does not merge.
3. **Determinism is a test target.** Anything inside the gate must be property-tested for
   same-input-same-output. Model calls inside the gate are a build failure.
4. **Agents are configuration plus evals, not just prompts.** Every agent ships with its
   task-type list, autonomy levels, proposal types, eval definition, and kill-switch
   registration. No agent goes live without its eval harness entry.
5. **Human review tiers from day one.** Every agent task type is classified auto-draft /
   act-on-approval / human-required, visible in output, mirroring the discipline Finexio
   already runs internally.
6. **Test against real data before calling anything done.** First run against a real
   supplier file or real exception queue always reveals gaps. Validation pass is not done.
7. **No fabricated numbers anywhere.** Trace to Phoenix v7 or label illustrative. In code,
   in dashboards, in copy.
8. **Prose discipline in anything human-facing:** no em dashes, short paragraphs, honest
   positioning.

## 9. Open items owned by the CEO (do not resolve these in code)

1. Confirm the Section 2 actuals with finance. Known flag: the Crossing deck says
   "$10B+ AP flow"; Phoenix v7 says $6.2B directly processed. The brief uses $6.2B.
   Do not mix the two.
2. Set the ask (dollar figure) in the vision brief's Part 8 card from Phoenix v7
   scenario sizing.
3. Bless or adjust the illustrative gate thresholds (50K decisions, cost index 80,
   +3pts card mix, $25M external volume) with the risk function and bank partner.
4. Bank partner conversation: the commit boundary and ladder are designed to slot into
   the issuer's model risk framework; that review has to be scheduled, not assumed.

## 10. Pointers

- Vision brief (published): https://claude.ai/code/artifact/aee945a0-3bac-4b5f-ae57-9ab048f1d340
- Vision brief (source): `xeos-vision.html` in `booyajones/claude-at-finexio`, PR #1
- Financial model: Phoenix v7 (Google Sheet; canonical ID in the finance reference notes)
- Internal agentic discipline this productizes: the Claude @ Finexio stack (skills,
  review tiers, two-layer audits, weekly QA) documented in this repo's `index.html`
