# PayGuard — Product Requirements Document

**Version:** 0.3  
**Status:** Draft (v1 data inputs locked: salary breakup + LOP, India only)  
**Date:** 2026-10-28  
**Product:** India payroll BPO control plane — prepare, verify, human approve (no payout in v1)

---

## 0. Decisions locked (2026-10-28)

| Decision | v1 choice | Later |
| --- | --- | --- |
| **Who PayGuard is for** | A **payroll BPO**: you process payroll for **many companies**; each company remains the **legal employer** | May expand to staffing, Infosys-style internal tools, and product-company vendor checking — **not in v1** |
| **Who employs the worker** | The **client company** (Employer A, B, C…). PayGuard is the processor, not the employer | — |
| **Country** | **India only** | Other countries = new rulesets |
| **Gross-to-net** | **Yes, calculate** PF / ESI (if in band) / PT / TDS in **code**, with a **documented simplified ruleset**. Agents explain; they do not invent rates | Real EPFO/IT filings, labour-code edge cases |
| **End of v1 workflow** | **Admin approve or reject** per employer. **No payout**, no bank file, no simulated NEFT | Payout + fund-isolation execution |
| **Project type** | **Learning and exploration** (portfolio-quality demo, not a live bureau) | Sellable product |
| **Money mixing** | Still **never mix employers** in data, packets, or totals — even without paying | Fund guard on real wallets |
| **Salary input** | **(B) Full monthly breakup** per employee: at least **basic**, **HRA**, **other allowances**. Employer decides the structure; PayGuard does **not** invent CTC splits | CTC-only auto-structure; more components (LTA, bonus as separate heads) |
| **Attendance** | **LOP days** in v1. Earnings are prorated; then PF/ESI/PT/TDS run on **payable** earnings | Hourly timesheets, OT, paid leave vs unpaid |
| **Operator UI** | **One BPO admin** sees all client employers (cards clearly labelled) | Per-client logins |

---

## 1. Executive summary

PayGuard v1 is a **learning prototype of an Indian payroll BPO**.

A BPO operator runs **one pay cycle** for **several client companies**. Each company stays the employer: their people, their **salary breakup**, their PF establishment (in the real world). PayGuard **must not blend** Employer A’s employees or numbers into Employer B.

Each employee’s master holds a **monthly structure** (basic, HRA, other allowances) plus **LOP days** for the period. Code prorates earnings for unpaid days, then calculates India deductions.

For each employer, **Agent 1 (Preparer)** reads that employer’s inputs and produces a **payroll packet**: payable earnings, each deduction line, net pay, employer PF/ESI, and a short written analysis. **Agent 2 (Verifier)** plus **deterministic code** recheck the packet against **source data** (not the preparer’s story). A **human admin** sees both analyses and **approves or rejects** that employer. **v1 stops there** — no money movement.

Calculations (PF, TDS, etc.) are a **simplified India rules engine in code**, labelled **illustrative / not for statutory filing**. The educational goal is the **maker–checker loop** and **per-employer isolation**, not replacing greytHR.

---

## 2. Idea analysis

### 2.1 What is strong about the idea

| Principle | Why it matters |
| --- | --- |
| **Split prepare vs verify** | Dual control. The verifier never “reviews the preparer’s homework”; it rebuilds checks from **source records**. Correlated agent error is the main failure mode of naive multi-agent payroll. |
| **Human is the only authority to move money** | Correct for high-stakes, regulated-adjacent work. Agents propose; humans decide; code executes. |
| **Approval bound to an exact packet** | Prevents time-of-check/time-of-use (TOCTOU): if employees, amounts, or totals change after approval, the approval is invalid. |
| **Fund isolation as a hard gate** | Multi-employer payroll’s worst bug is paying Employer B with Employer A’s cash. This must be **code**, not a prompt. |
| **Deterministic execution** | LLMs must not call payment APIs, invent routing numbers, or “fix” totals at send time. |
| **Full audit trail** | Who prepared, verified, approved, which packet, when. |

### 2.2 What is dangerous if left vague

1. **“Payroll” is an enormous product.** A production India engine (all states’ PT, both tax regimes, perquisites, 80C, Form 16, EPFO files) is years of work. v1 **does** calculate PF/TDS, but only via a **small, versioned, documented ruleset**. It is a teaching engine, not a filing product.
2. **LLMs are unreliable at arithmetic and roster integrity.** A verifier that only “reads the packet and comments” will miss swapped digits and duplicate payees. Numeric and structural checks must be **deterministic**; the model writes **analysis**, it does not own the verdict math.
3. **Two agents sharing a model, prompt, or context window are not independent.** Independence requires **separate runs**, **no packet-internal justification as the source of truth**, and **source-data access** for the verifier.
4. **Agents with payment credentials are an incident waiting to happen.** Agents produce artifacts. Only the executor, behind policy, talks to rails.
5. **A single human “Approve all” button** will be clicked under time pressure. The UI must make **one employer, one decision**, with fail/notes impossible to miss.
6. **Mock money and real money look the same in a demo.** The product needs an explicit **execution mode** (dry-run / paper / live) so a prototype cannot accidentally become a live rail.

### 2.3 Recommended product thesis

> PayGuard v1 is a **segregated, dual-control India payroll processor for a BPO**. Agents explain packets. **Code** computes deductions and invariants. **Humans** are the only principals who can approve a hashed packet. **Nothing is paid in v1.**

That thesis is buildable as an MVP, demo-able, and honest about risk.

### 2.4 Comparable patterns (not competitors)

- Dual control / maker-checker in banking
- Content-addressed approvals (git commit = payload hash)
- Multi-tenant wallets with per-tenant ledgers
- Agent-propose / human-approve (not agent-execute) for irreversible actions

---

## 3. Problem and opportunity

### 3.1 Problem

Running payroll for **several employers in one operation** fails in predictable ways:

- Packets are assembled by hand or by a single opaque system; errors are found after money moves.
- Review is a spreadsheet dump, not a **verdict with evidence**.
- Approval is not bound to a frozen payload, so last-minute edits pay the wrong people.
- Employer funds sit in a pooled account or a shared “payroll” balance; mix-ups are possible.
- After the fact, nobody can prove who signed off on **which** numbers.

### 3.2 Opportunity

A small, rigorous system that:

1. Builds a **canonical packet** per employer per pay period.
2. Runs **independent verification** (rules + narrative).
3. Presents **one human decision per employer**.
4. Stops at a **human approve/reject** bound to that hash (v1 does not pay).
5. Leaves an **immutable log**.

Payout and fund-guard execution stay designed for later, so a future expansion does not rip up the model.

---

## 4. Goals and non-goals

### 4.1 Goals (v1)

- Run a **monthly India pay cycle** across **N ≥ 3 client employers** in a sandbox.
- For each employer, take **stored salary breakup** (basic, HRA, other allowances) and **LOP days**, **prorate** earnings, then **compute** PF, ESI if eligible, professional tax for **one documented state**, and TDS under a **simplified new-regime** estimate, all in **integer paise**.
- Produce a **complete, hashed payroll packet** per employer (lines + totals + analysis).
- Verify each packet **against that employer’s source data**, with structured flags and a verdict.
- Require **explicit human approve/reject per employer**; bind approval to `content_hash`.
- **Isolate tenants:** no query or packet may mix two employers’ employees or totals.
- Persist an **audit log** for prepare, verify, and approve/reject.
- Label every calculated pack **illustrative — not for EPFO / income-tax filing**.

### 4.2 Non-goals (v1)

- **Paying anyone** (real or simulated bank/UPI/NEFT). Workflow ends at approval.
- Filing PF/ESI/TDS, generating legal Form 16, or talking to EPFO/IT portals.
- A complete India tax engine (old regime, 80C, HRA exemption, perquisites, all PT states, labour-welfare fund, bonus act, gratuity payout).
- Staffing bill rates, Infosys T&M invoicing, or product-company vendor-invoice checking.
- Becoming an HRIS (hiring, PTO, performance).
- Multi-country payroll, FX, SSO, employee self-service.
- Autonomous approval.
- Agents inventing CTC splits, basic, LOP, or tax percentages.
- Overtime, paid leave vs unpaid beyond LOP days, mid-month joiner/exit as a separate feature (a joiner can be approximated with LOP days in fixtures if needed).

### 4.3 Later candidates (explicitly out of v1)

- Simulated then real payout + fund guard (original PayGuard “execute” story).
- The other three buyer types (staffing, IT-services internal, principal-employer vendor check).
- Partner or full statutory engine; multi-state PT; old tax regime.
- Multi-approver policy, client-portal login per employer, GL export.

---

## 5. Users and roles

| Role | Human or system | Authority |
| --- | --- | --- |
| **Operator / Approver** | Human | Only principal who can approve or reject a packet. Sees one summary per employer. |
| **BPO operator / Approver** | Human | One operator for the learning app. Approves or rejects **per client employer**. |
| **Payroll admin** | Human | Maintains source records: employers, employees, salary inputs, attendance (v1 may seed fixtures). |
| **Preparer Agent** | System | Assembles packet + written analysis from source + **calc engine outputs**. Cannot approve or pay. |
| **Verifier Agent** | System | Rechecks from source + calc engine + rules. Writes analysis. Cannot approve or pay. |
| **India calc engine** | Deterministic code | PF, ESI, PT, TDS, totals. Versioned ruleset id stored on every packet. |
| **Executor / Fund Guard** | Deterministic code | **Not invoked in v1.** Kept in the model for a later payout phase. |
| **Audit log** | System | Append-only record of events. No silent edits. |

v1 assumes **one human approver** for the whole installation. Multi-user auth is a later milestone; the **data model** should still record `actor_id`.

---

## 6. Product principles (non-negotiable)

1. **Agents propose; humans dispose; code calculates.** No agent-set tax rates.
2. **Verifier reads source of truth, not the preparer’s rationale.**
3. **Math and isolation are not LLM jobs.**
4. **Approval is of a hash, not of a screen.**
5. **Fail closed.** Invalid calc, missing data, or hash mismatch → cannot approve.
6. **No cross-employer mixing** of people, packets, or totals.
7. **v1 does not pay.** Approval is the last step. A future executor must still bind to the same hash.
8. **Audit is complete or the cycle is invalid.**
9. **Every packet stores `ruleset_version`.** Changing PF/TDS rules is a new version, not a silent prompt edit.

---

## 7. Core concepts

### 7.1 Employer

A **client company** of the BPO: a legal employer with its own employee roster and salary inputs. Employers never share a packet or a pay file. A **wallet** is optional in v1 (no payout); if shown, it is display-only.

### 7.2 Employee (payee)

A person (or contractor, if in scope) belonging to **exactly one** employer for a given packet. A person who works for two employers appears as **two payee records**, never as one blended line.

### 7.3 Pay period

Inclusive date range + pay date. A packet is unique on `(employer_id, pay_period_id)` for a given cycle attempt. Re-runs after rejection or mutation get a **new packet id**.

### 7.4 Payroll packet

The **unit of approval**. It is a structured document, not a chat transcript.

Minimum contents:

- `packet_id` (opaque id)
- `employer_id`
- `pay_period` (start, end, pay_date)
- `currency`: `INR`
- `ruleset_version` (India calc engine id)
- `calendar_days` in the pay period (inclusive)
- `payees[]`: employee_id, legal name, `pt_state`, `lop_days`, `payable_days`, full-month earnings (`basic`, `hra`, `other_allowances`), payable earnings (same three + `payable_gross`), `employee_pf`, `employee_esi`, `professional_tax`, `tds`, `net_pay`, `employer_pf`, `employer_esi`
- `totals`: payee count, sum of gross, each deduction, net, employer statutory
- `preparer`: agent id, model/version, timestamp
- `content_hash` (canonical serialization, e.g. SHA-256 over canonical JSON)
- `schema_version`
- `disclaimer`: illustrative, not for filing
- `prior_period_packet_id` (optional, for delta analysis)
- `notes` from preparer (optional, non-authoritative)

### 7.5 Salary structure and LOP (v1 source of truth)

Each employee belongs to **one** employer and has a **monthly structure** stored by that employer (paise):

| Field | Meaning |
| --- | --- |
| `basic` | Basic wages for a full month |
| `hra` | House rent allowance, full month |
| `other_allowances` | Everything else treated as taxable earnings in v1 (one bucket) |
| `lop_days` | Unpaid days in **this** pay period (integer ≥ 0) |

**Full-month gross** = basic + hra + other_allowances. PayGuard does **not** derive these from CTC.

**LOP proration (deterministic):**

1. `calendar_days` = number of calendar days in the pay period (inclusive).
2. Reject the employee line if `lop_days > calendar_days` or `lop_days < 0`.
3. `payable_days` = `calendar_days - lop_days`.
4. For each earning component:  
   `payable = (full_month_paise × payable_days) // calendar_days`  
   (integer floor division — no floats).
5. `payable_gross` = sum of payable components.
6. Run PF / ESI / PT / TDS on **payable** figures (see §9.4).  # keep
7. `net_pay` = payable_gross − employee_pf − employee_esi − professional_tax − tds.

Zero LOP ⇒ payable equals full month (because `payable_days == calendar_days`).

### 7.6 Verification report

Produced **after** the packet exists. Includes:

- `packet_id` + `content_hash` (must match)
- Deterministic findings (machine)
- Written analysis (agent)
- `verdict`: `pass` | `pass_with_notes` | `fail`
- `flags[]` (see §9)
- `verifier`: agent id, model/version, timestamp, ruleset version

A `fail` **must not** be approvable. v1 omits override. Failed packets are rejected or sent back to prepare after source data is fixed.

### 7.7 Approval

- `packet_id`, `content_hash`, `approver_id`, `decision` (`approved` | `rejected`), timestamp, optional comment.
- Stored independently of the UI session.
- **Invalid** if current packet hash ≠ approved hash.

### 7.8 Payment instruction (later phase)

Not generated in v1. When added: one instruction per payee from an approved hash only; fund guard; idempotency key = `hash(packet_id + payee_id + content_hash)`.

---

## 8. End-to-end workflow

```
Source records (per employer: employees, salary inputs, attendance)
        │
        ▼
[0 CALC]     India rules engine → line-level PF/ESI/PT/TDS/net (paise)
        │
        ▼
[1 PREPARE]  Preparer Agent → packet + written analysis + content_hash
        │
        ▼
[2 VERIFY]   Re-run calc + roster/totals rules + Verifier Agent narrative
             → Verification Report + verdict
        │
        ▼
[3 APPROVE]  Human: one summary per employer → approve | reject
             Reject → stop (fix data, new packet later)
             Approve → Approval(record, hash)  ★ v1 ends here
        │
        ▼
[AUDIT]      prepare, verify, decision already appended
```

**Re-approval rule:** Any change to source data or packet bytes requires a **new packet**, new verify, new human approval. There is no “edit in place.”

**Parallelism:** Employers can be prepared/verified in parallel. Packets never share a batch.

---

## 9. Verification: rules vs language

### 9.1 Deterministic checks (must implement in code)

| Check | Fail if |
| --- | --- |
| Schema / required fields | Packet invalid |
| Hash stability | Recomputed hash ≠ stored hash |
| Employer consistency | Any payee’s employer_id ≠ packet employer_id |
| Duplicate payees | Same `employee_id` twice, or same payout account twice in one packet (configurable) |
| Totals | Sum of line nets ≠ `totals.net` (integer paise) |
| Calc replay | Re-running the calc engine on the same inputs ≠ packet lines |
| Cross-tenant | Any employee or total from another employer |
| Roster: new employees | Employee in packet not in source, or first appearance vs prior period |
| Roster: removed employees | Active source employee missing from packet vs prior period / expected roster |
| Amount outliers | Net is 0, negative, or change vs prior period exceeds threshold (e.g. 20%) — **flag** |
| LOP range | `lop_days` not an integer in `[0, calendar_days]` |
| Proration replay | Payable components ≠ floor formula in §7.5 |
| Stale data | Source records mutated after prepare timestamp (packet invalid) |

**Verdict mapping (v1 default):**

- Any integrity/math/employer-mismatch/duplicate/hash/stale-source issue → **`fail`**
- Only roster deltas and unusual-but-positive amounts with consistent math → **`pass_with_notes`**
- No flags → **`pass`**
- Calc mismatch → **`fail`**

Thresholds for “unusual change” should be **config per employer**, with a global default.

### 9.2 Verifier Agent (language)

Input: source extracts (roster, prior packet summary, current packet, deterministic findings). **Not** the preparer’s chain-of-thought.

Output: short written analysis (plain language for the human): what changed, what is risky, what was checked. Must **cite flags by id**, not invent numbers. If the model’s implied totals disagree with code, **code wins** and the report is marked `analysis_inconsistent` → treat as **`fail`**.

### 9.3 Independence requirements

- Separate agent identity and run (no shared scratchpad).
- Verifier cannot write packets.
- Prefer different model or at least different system prompt and temperature 0 for structured parts.
- Deterministic ruleset version is logged.

### 9.4 India calc engine (v1 teaching ruleset)

All money in **paise**. Percentages live in a versioned rules file, not in agent prompts. Verifier **recomputes**. If Agent 1’s numbers differ, **fail**.

Apply **after** LOP proration (§7.5). “Basic” and “gross” below mean **payable_basic** and **payable_gross**.

| Item | v1 teaching behaviour |
| --- | --- |
| PF employee / employer | 12% of payable basic, **capped** at a single documented wage ceiling (same cap both sides) |
| ESI | If payable gross ≤ documented ceiling: employee 0.75%, employer 3.25%; else 0 |
| Professional tax | **Karnataka slabs only** in v1 (all fixtures use KA). Other `pt_state` values → **fail** that line until a later ruleset |
| TDS | **New regime only**. Monthly taxable ≈ payable_gross − employee_pf − employee_esi − PT (as documented). Annualise × 12; apply simplified slabs + standard deduction + rebate; divide by 12; floor to paise. No 80C, no HRA exemption |
| Gratuity / bonus / LWF | Out of v1 (0) |

---

## 10. Fund protection

### 10.1 Ledger model (v1)

Each employer has a **wallet** with:

- `employer_id`
- `balance` in minor units
- `funding_source_id`
- optional `hold` amount during a cycle

**Never** a single pooled “PayGuard balance” that is later split.

### 10.2 Pre-payout guard (hard)

For each payment instruction:

1. Debit employer wallet A only.
2. Credit payees of employer A only.
3. Sum(debits) = sum(credits) + explicit fees **if and only if** fees are also debited from A (v1: **no fees**, 1:1).
4. Reject the entire employer execution if any instruction references another employer’s wallet or a global pool.
5. Reject if balance < total.
6. Reject if `content_hash` ≠ approved hash.

A mix-up is **not** a retryable warning. It is a **blocked cycle** for that employer.

### 10.3 Invariants to test

- Property: ∀ payment, `instruction.wallet_employer_id == packet.employer_id == payee.employer_id`
- Property: no ledger row exists with two employer_ids
- Property: executing employer A cannot decrease employer B’s balance

---

## 11. Execution

**v1: none.** After approval the UI shows “Approved — not paid (v1).” Executor, wallets, and fund guard remain specified for a later phase so we do not paint ourselves into a corner.

---

## 12. Audit log

Append-only events. Suggested types:

| Event | Payload (minimum) |
| --- | --- |
| `prepare_completed` | packet_id, hash, preparer_id, employer_id |
| `verify_completed` | packet_id, hash, verifier_id, verdict, flag_ids |
| `approval_recorded` | packet_id, hash, actor_id, decision |
| `source_mutated` | entity ids (if admin edits records) |

Rules:

- No update/delete of events in v1 (corrections = new compensating events).
- Store hashes so a later auditor can recompute packet bytes.
- Include timestamps in UTC.

---

## 13. Human approval UX (v1)

**One page (or one card) per employer**, not a combined blob.

Must show:

- Employer name and pay period
- Total amount and payee count
- Currency INR; **ruleset_version**
- Gross, PF, ESI, PT, TDS, **net** totals (not only net)
- Headcount with **LOP** (how many people had unpaid days)
- Verifier **verdict** (color/label: pass / notes / fail)
- Flag list in plain language (new/removed people, % changes, duplicates, LOP outliers)
- Verifier written analysis
- Expandable payee table (name, LOP days, payable gross, net, vs last period if available)
- Packet hash (short prefix visible; full hash copyable)
- Actions: **Approve** | **Reject** (reject requires a short reason)

Must not:

- Approve all employers in one click (v1)
- Approve a `fail` packet
- Hide notes behind a tooltip only

After decision, show confirmation: “You approved hash `abc123…` for Employer X, ₹Y net, N payees.”

---

## 14. Data sources (v1)

Seeded **source records** (no greytHR):

- Client employers (name, id, `pt_state` = `KA` in v1 fixtures)
- Employees per employer (ids must not leak across clients)
- Monthly breakup per employee: `basic`, `hra`, `other_allowances` (paise)
- `lop_days` per employee per pay period
- Optional prior-period packets for deltas

Fixtures: **at least three employers**, including messy cases (new hire, exit, **several LOP values including 0 and mid-month**, duplicate id attempt, huge net jump, one person in ESI band and one well above).

---

## 15. Functional requirements

### Packet lifecycle

- **FR-1** System can create a packet per employer from source records for a given pay period.
- **FR-2** Packet serialization is canonical; `content_hash` is stored and recomputable.
- **FR-3** Preparing the same sources twice with no change yields the same hash (stability).

### Verify

- **FR-4** Verifier pipeline loads **source records + packet**, not preparer notes as authority.
- **FR-5** Deterministic rules in §9.1 run and persist structured flags.
- **FR-6** Agent analysis is stored; verdict is stored; `fail` blocks approval.

### Approve

- **FR-7** Human can list employers in a cycle with verdict and totals.
- **FR-8** Human can approve or reject per employer; approval stores hash + actor.
- **FR-9** Rejected packets are not executable.

### Calc

- **FR-10** Calc engine prorates breakup by LOP using §7.5, then produces PF/ESI/PT/TDS/net in paise; output is replay-stable.
- **FR-11** Packet lines must match a fresh calc on the same inputs or the packet is `fail`.
- **FR-10b** `lop_days` outside `[0, calendar_days]` fails the employee line (and typically the employer packet).

### Execute and guard (later)

- **FR-12–FR-13** Reserved: payout + fund isolation. **Not in v1.**

### Audit

- **FR-14** Every transition in §12 is logged.
- **FR-15** A cycle report can answer: who prepared, who verified, who approved or rejected, which hash, when.

### Tenancy

- **FR-16** Queries and execution paths are scoped by `employer_id`; no “run all employers as one payment batch” API.

---

## 16. Non-functional requirements

| ID | Requirement |
| --- | --- |
| **NFR-1** | Money stored as integer **paise**, never float. |
| **NFR-2** | Packet JSON (or equivalent) canonicalization documented (key order, no insignificant whitespace). |
| **NFR-3** | Agent timeouts: prepare/verify fail closed if the agent does not return valid structured output. |
| **NFR-4** | Secrets (model keys, future bank creds) never in packets or audit payloads. |
| **NFR-5** | Payee account details tokenized or masked in UI (last 4 only). |
| **NFR-6** | Prototype may be single-node; data must still survive process restart (DB or embedded store). |
| **NFR-7** | Automated tests for hash binding, tenant isolation, and calc replay — these are the product. |

---

## 17. System architecture (logical)

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│ Source DB   │     │ India calc   │     │ Packet store    │
│ per-employer│────►│ engine       │────►│ + content_hash  │
│ employees & │     │ (code)       │     └────────┬────────┘
│ salary input│     └──────┬───────┘              │
│             │            │                      │
│             │     ┌──────▼───────┐              │
│             │     │ Preparer     │──────────────┤
│             │     │ Agent        │              │
│             │     └──────────────┘              │
│             │     ┌──────────────┐              │
│             │────►│ Re-calc +    │◄─────────────┤
│             │     │ rules +      │              │
│             │     │ Verifier     │              │
└─────────────┘     └──────┬───────┘              │
                           ▼                      │
                    Verification reports          │
                           ▼                      │
                    Approval UI (human) ★ v1 end  │
                           ▼                      │
                    Approvals + audit log         │
```

**Trust boundary:** Anything labeled Agent is untrusted for money movement. Anything labeled code/guard/ledger is trusted for invariants.

**Implementation (stack, statuses, DB job poll, PacketService, agent optionality):** [`architecture-and-tech-stack.md`](./architecture-and-tech-stack.md).

---

## 18. Agent contracts (interfaces)

Agents return **JSON that validates against a schema**. Free-form chat is not a packet.

### Preparer

- **In:** `employer_id`, `pay_period_id`, source snapshot (or snapshot id).
- **Out:** packet object matching schema. Invalid JSON → prepare failed, no packet.

The preparer **must not** invent employees, salary breakup, LOP days, or tax percentages. Deductions come from the **calc engine**. Post-prepare code asserts roster ⊆ source and lines = engine output. **v1 policy: assembler + narrator only.**

### Verifier

- **In:** source snapshot, packet, deterministic findings.
- **Out:** `{ analysis: string, highlights: string[] }`  
  Verdict is **computed by code** from findings + optional consistency check on analysis, not chosen solely by the model.

This is a deliberate tightening of the original idea: the model **does not get the last word on pass/fail**.

---

## 19. Success metrics

| Metric | Target (prototype) |
| --- | --- |
| Dual-control demo | 3 employers, pass / pass_with_notes / fail in one cycle |
| Calc honesty | Replay of ruleset matches packet; LLM cannot change PF% |
| Zero mix-ups | Tests cannot put Employer B’s employee on Employer A’s packet |
| Review time | Human can decide from the summary without raw JSON |
| Audit completeness | Every cycle has prepare, verify, approve or reject |

Production metrics (later): error rate after approval, time-to-approve, % fail packets, fund-block incidents.

---

## 20. Phased roadmap

### Phase 0 — Foundations (this PRD)

- Domain model, hash spec, event list, UI wire for one-employer card.

### Phase 1 — Vertical slice (MVP / this learning project)

- Fixtures for 3 Indian client employers with **basic / HRA / other** and mixed **LOP**.
- Calc engine + tests (golden payslips, including LOP = 0 and LOP > 0).
- Packet build (assembler + optional LLM narrative).
- Verifier + approval UI.
- Audit log viewer.
- **Stop at approval.**

### Phase 2 — Agent independence and richer India

- Separate prepare/verify runs; analysis vs numbers consistency.
- Prior-period deltas; LOP/attendance if not in Phase 1.
- Optional second PT state.

### Phase 3 — Other buyer types (your stated later expansion)

- Staffing (bill vs pay).
- Infosys-style T&M vs CTC reconciliation.
- Product-company vendor invoice checking.
- Each is a **new packet type**, not a fork of isolation rules.

### Phase 4 — Payout (only if you choose to)

- Simulated ledger + fund guard, then (much later) real rails.

---

## 21. Risks and mitigations

| Risk | Mitigation |
| --- | --- |
| LLM hallucinates a payee or amount | v1 preparer is an assembler; code asserts packet ⊆ source |
| Verifier rubber-stamps preparer | Independent snapshot; verdict from rules; no shared scratchpad |
| Human click-through | Per-employer decision; fail cannot approve; hash shown |
| Hash mismatch after “small fix” | No in-place edit; new packet |
| Float rounding | Integer paise |
| Demo looks like legal payroll | UI disclaimer; no EPFO export |
| Teaching PF rates go stale | `ruleset_version`; never bury rates only in prompts |
| Insider admin changes source after prepare | Stale-snapshot check; mutate events in audit |

---

## 22. Open questions

**Locked (see §0 and §7.5):** payroll BPO; each client is legal employer; **India only**; calculate PF/TDS; salary breakup **(B)**; **LOP days**; stop at approval; learning project; one BPO admin sees all clients; v1 PT = **Karnataka** only.

**Not blocking for a first build:**

1. `pass_with_notes` — extra checkboxes before Approve?
2. Contractors on the same packet as employees? (Recommend **employees only**.)
3. LLM provider / how much PII goes into prompts?
4. Exact PF wage ceiling and TDS slab table to copy into `ruleset_version` when coding (document in code; do not ask the model).

---

## 23. v1 acceptance criteria (definition of done)

A reviewer can:

1. Load seeded data for **at least three employers**.
2. Run **prepare** and see a hashed packet per employer.
3. Run **verify** and see **pass**, **pass_with_notes**, and **fail** represented.
4. **Approve** only non-fail packets; **reject** others.
5. Observe that **changing source data** after prepare invalidates the old hash; a new approval is required.
6. Observe that Employer B’s employee **cannot** appear on Employer A’s packet.
7. Open an **audit view**: who prepared, verified, approved or rejected, for which hash, when.
8. Recalculate a known fixture payslip and get the **same paise** twice.
9. A fixture employee with **LOP > 0** has lower payable gross than the same structure with LOP = 0, by the floor formula in §7.5.

Until these are true, PayGuard is a slide, not a product.

---

## 24. Document control

| Version | Date | Notes |
| --- | --- | --- |
| 0.1 | 2026-10-28 | Initial PRD from product idea |
| 0.2 | 2026-10-28 | Locked: India payroll BPO; calc PF/TDS; stop at approval; learning |
| 0.3 | 2026-10-28 | Locked: salary breakup (B); LOP proration; India only; KA PT |
| 0.3.1 | 2026-10-30 | Pointer to architecture-and-tech-stack.md |

Changes to principles in §6 or fund isolation in §10 require an explicit PRD revision, not a silent code change.
