# PayGuard — Tech stack, architecture, and flow

**Version:** 0.2  
**Date:** 2026-10-30  
**Status:** Agreed for v1 implementation  
**Companion:** [`PRD.md`](./PRD.md) (product rules). This file is **how we build it**.

---

## 1. What we are building (one paragraph)

A **learning** India payroll BPO app: one operator runs a monthly cycle for **several client companies**. **Java** prorates LOP and calculates PF / ESI / PT / TDS in **paise**. That result is **frozen** as a hashed **packet**. Optional **LLM agents** write **two independent reviews** (analysis + opinion each). Agent 2 **never overwrites** Agent 1. A **second Java pass** produces evidence (packet vs re-calc). The human **approves or rejects** after seeing both agents and the evidence. **No payout.** **No Kafka.** Work between phases is a **Postgres job table** polled by a worker.

If the LLM is down, calc + freeze + Java verify + approve still work (`AGENTS_ENABLED=false`).

---

## 2. Tech stack (v1)

### 2.1 Chosen

| Layer | Technology | Role |
| --- | --- | --- |
| Language / runtime | **Java 21** | API, calc, workers |
| Framework | **Spring Boot 3.x** | HTTP, JPA, scheduling, config |
| Database | **PostgreSQL 16** | Source data, packets, jobs, audit — **system of record** |
| Migrations | **Flyway** | Schema versioning |
| JSON | **Jackson** | Packets + canonical serialization for hashing |
| Hash | **SHA-256** (`MessageDigest`) | `content_hash` on every packet |
| Money | **`long` / `BIGINT` paise** | No `double` for pay |
| Calc | **Pure Java module** (no Spring in the core) | Replay-stable ruleset |
| Agents | **Optional** HTTP LLM (Spring AI or `WebClient`) | Narrative JSON only |
| Job trigger | **`pipeline_jobs` table** + `@Scheduled` poller + `FOR UPDATE SKIP LOCKED` | Not Kafka |
| API | **REST** | UI and (optional) curl |
| Frontend | **React + TypeScript + Vite** | One card per employer; phase status |
| Data fetching | **TanStack Query** | Poll cycle status while jobs run |
| Auth | **Spring Security**, one demo user | Record `actor_id` on approve |
| Tests | **JUnit 5**, **AssertJ**, **Testcontainers** (Postgres) | Calc golden files + tenant isolation |
| Local | **Docker Compose** (Postgres + API; UI via Vite) | Repeatable fixtures |

### 2.2 Explicitly out of v1

Kafka, Redis, microservices, Kubernetes, Keycloak/SSO, Mongo as system of record, LangChain as calculator, payout / UPI / NEFT, EPFO filing.

Kafka may return later as an **outbox doorbell**. The folder of record stays PostgreSQL.

### 2.3 Repo layout (MVC)

Backend is **package-by-layer Spring MVC**. Frontend mirrors that: pages = views, `api/` = HTTP, components = UI.

```
docs/                              PRD, research, this file
infra/docker-compose.yml           PostgreSQL 16
apps/api/                          Spring Boot (com.payguard)
  src/main/java/com/payguard/
    controller/                    REST only (thin)
    service/                       Cycle, Packet, Verify, Approval
    repository/                    Spring Data JPA
    model/                         JPA entities
    dto/                           API + packet JSON
    mapper/                        entity ↔ dto
    calc/                          Pure Java maths (no Spring)
    agent/                         Optional LLM clients
    job/                           DB job poller
    config/                        Security, CORS
    exception/                     API errors
    audit/                         Append-only events
  src/main/resources/db/migration  Flyway
  src/test/java/.../calc           Golden payslip tests
apps/web/                          React + Vite + TypeScript
  src/app/                         Routes
  src/pages/                       Screens (views)
  src/layouts/                     App chrome
  src/components/                  Reusable UI
  src/hooks/                       React Query
  src/api/                         fetch to /api
  src/types/                       Shared TS models
  src/styles/                      Global CSS
```

**MVC rules (API):** controllers call services only; services call repositories/calc/agents; `calc` has no Spring and no HTTP.

**UI rules:** pages do not call `fetch` directly; they use `api/` + hooks. No PF/TDS logic in React.

Calc tests must run **without** Postgres and **without** an API key.

---

## 3. Principles (architecture)

1. **Java owns money.** Agents never set PF%, net, or LOP.
2. **PacketService freezes.** Approval is of `content_hash`, not of a live query.
3. **Verify = Java second calc + evidence.** Agent 2 explains evidence; it does not judge paise.
4. **Postgres is the queue and the truth.** UI reads statuses from DB.
5. **Fail closed.** Bad JSON, timeout, hash mismatch, calc mismatch → cannot approve.
6. **One `employer_id` on every query.** No cross-client employee lists.
7. **Agents optional.** `AGENTS_ENABLED` default true in demo, false still a complete control plane.
8. **Illustrative payroll.** UI disclaimer; not for EPFO / Form 16.

---

## 4. Service architecture

### 4.1 One deployable

v1 is **one Spring Boot process** (HTTP + scheduled worker). React is a separate static/dev server. Not two agent microservices.

```
┌─────────────────────────────────────────────────────────────────┐
│  apps/web (React)                                               │
│  Cycle board → employer cards → packet + evidence + narratives  │
│  Status from GET /cycles/{id}  (poll 2–3s while not terminal)   │
└────────────────────────────┬────────────────────────────────────┘
                             │ REST
┌────────────────────────────▼────────────────────────────────────┐
│  apps/api  (Spring Boot)                                        │
│                                                                 │
│  CycleController    PacketController    ApprovalController      │
│          │                  │                    │              │
│          ▼                  ▼                    ▼              │
│  ┌─────────────┐   ┌────────────────┐   ┌──────────────┐        │
│  │ CycleService│   │ PacketService  │   │ApprovalService│       │
│  │ statuses    │   │ freeze + hash  │   │ hash check   │        │
│  └──────┬──────┘   └───────┬────────┘   └──────┬───────┘        │
│         │                  │                    │               │
│         │         ┌────────▼────────┐                           │
│         │         │  CalcEngine     │  ← used at freeze AND     │
│         │         │  (pure Java)    │    at verify              │
│         │         └────────┬────────┘                           │
│         │                  │                                    │
│  ┌──────▼──────────────────▼──────────────┐                     │
│  │ VerifyService (Java rules + evidence)  │                     │
│  └──────┬─────────────────────────────────┘                     │
│         │                                                       │
│  ┌──────▼────────────┐  ┌─────────────────┐                     │
│  │ AgentClient       │  │ JobPoller       │                     │
│  │ (optional LLM)    │  │ @Scheduled      │                     │
│  └───────────────────┘  └────────┬────────┘                     │
│                                  │ SKIP LOCKED                  │
└──────────────────────────────────┼──────────────────────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │         PostgreSQL          │
                    │  masters, packets, reports, │
                    │  approvals, jobs, audit     │
                    └─────────────────────────────┘
```

### 4.2 Component responsibilities

| Component | Does | Does not |
| --- | --- | --- |
| **CalcEngine** | LOP floor proration; PF/ESI/PT/TDS; line + totals; `ruleset_version` | HTTP, DB, LLM |
| **PacketService** | Build packet JSON from engine + roster; canonicalize; SHA-256; save JSONB | Invent pay; approve |
| **VerifyService** | Re-run CalcEngine; roster/totals/LOP rules; **diff packet vs Java**; persist **Java evidence** + **Java integrity result** | Erase or overwrite Agent 1’s analysis/opinion |
| **Agent 1 (preparer)** | After freeze: English analysis of the frozen packet + **its own opinion** (`pass` / `pass_with_notes` / `fail`) | Change paise; be the only verdict on screen |
| **Agent 2 (verifier)** | After Java evidence: English analysis citing finding ids + **its own opinion** (same three labels) | Change paise; **overwrite Agent 1**; replace Java evidence |
| **JobPoller** | Claim `QUEUED` jobs; run prepare-narrative or verify; update cycle status | Be a message broker |
| **ApprovalService** | Recompute hash; allow approve only if status ready and **Java integrity** is not a hard fail (calc/hash/tenant mismatch) | Pay anyone; pick a winner between agents |
| **AuditService** | Insert-only events | Update/delete events |

### 4.3 Trust boundary

| Trusted for money and **integrity** (can we even approve?) | Untrusted for money; **opinions only** |
| --- | --- |
| CalcEngine, PacketService, VerifyService evidence, ApprovalService hash check | Agent 1 analysis + opinion, Agent 2 analysis + opinion, UI |

### 4.4 Agent 1 and Agent 2 (roles in detail)

Neither agent is a payroll calculator. Java already froze the packet. Agents are **two independent written reviews** of that packet. **Agent 2 must not overwrite Agent 1.** The admin sees **both** texts and **both** opinions, plus the **Java evidence table**.

#### Agent 1 — Preparer (cover letter on the stamped register)

**When:** After PacketService has stored the packet and `content_hash`. Status `NARRATING`.

**Sees:** Frozen packet (payees, LOP, payable lines, totals, vs last period if any). Does **not** invent breakup or tax %.

**Produces (saved on `preparer_analyses`, never updated by Agent 2):**

| Field | Meaning |
| --- | --- |
| `analysis` | Plain language: who this employer is this month, notable LOP, new/removed people, large net changes, totals in words |
| `highlights[]` | Short bullets the admin can scan |
| `opinion` | Agent 1’s recommendation only: `pass` / `pass_with_notes` / `fail` |
| `opinion_reason` | One short why (e.g. “two staff with 8 LOP days”) |
| `model` / timestamp | For audit |

**Opinion is not approval.** It is “if I were briefing you, I would be comfortable / cautious / unhappy.”

**Must not:** change paise, rewrite the packet, call Approve, delete evidence.

#### Agent 2 — Verifier (second briefing, after Java re-check)

**When:** After VerifyService has re-run CalcEngine and built the **evidence table**. Status still `VALIDATING` until this (optional) write finishes.

**Sees:** Same frozen packet **plus** Java findings (match/mismatch per employee, roster flags, LOP flags). **Does not receive Agent 1’s analysis as the source of truth.** Optionally it may be told “Agent 1 existed” without copying Agent 1’s opinion into Agent 2’s row. Best: **do not pass Agent 1’s text into Agent 2’s prompt**, so the two reviews stay independent.

**Produces (saved on `verification_reports`, a **different row/table** from Agent 1):**

| Field | Meaning |
| --- | --- |
| `analysis` | Cover letter that **cites finding ids** (`F-014`, …) |
| `opinion` | Agent 2’s own recommendation: `pass` / `pass_with_notes` / `fail` |
| `opinion_reason` | Tied to evidence (“F-015 net mismatch 200 paise”) |
| `model` / timestamp | For audit |

**Must not:** overwrite `preparer_analyses`; change packet bytes; replace Java `findings`; set `employer_run` approve/reject.

#### Three columns the admin always sees

| Column | Who | Role |
| --- | --- | --- |
| **Packet + Java evidence** | Code | What was paid in the packet vs re-calc; integrity (hash, tenant, LOP formula) |
| **Agent 1** | Preparer LLM | First narrative + **opinion A** |
| **Agent 2** | Verifier LLM | Second narrative + **opinion B** |

If they disagree (Agent 1 `pass`, Agent 2 `fail`), **keep both**. The UI shows two badges, not a single merged “system verdict” from Agent 2.

#### What can still block Approve (not an agent override)

- **Java integrity fail:** packet ≠ second calc, bad hash, wrong employer on a line, illegal LOP. Admin should not be able to Approve a mathematically broken packet. That is **code**, not Agent 2 beating Agent 1.
- **Human:** Approve / Reject is the only decision that counts. They can follow Agent 1, Agent 2, neither, or both notes.

If agents are off: those two columns are empty; Java evidence + human still work.

#### Persistence rule

```
preparer_analyses     = Agent 1 only  (insert once, never updated by verify)
verification_reports  = Java findings + Agent 2 opinion/text  (Agent 2 fills text/opinion; Java fills findings)
approvals             = human only
```

No `UPDATE preparer_analyses SET opinion = agent2.opinion`.

---

## 5. Data model (logical)

All money: **BIGINT paise**. Every tenant table includes **`employer_id`**.

| Table | Purpose |
| --- | --- |
| `employers` | Client companies (`pt_state` = `KA` in v1 fixtures) |
| `employees` | Belongs to one employer |
| `salary_structures` | Monthly `basic`, `hra`, `other_allowances` |
| `pay_periods` | Date range, pay date, `calendar_days` |
| `attendance_lop` | `employee_id` + `pay_period_id` + `lop_days` |
| `payroll_cycles` | A run across employers for one period; or one row per employer-in-cycle (see below) |
| `packets` | Frozen JSONB, `content_hash`, `ruleset_version`, `status` |
| `preparer_analyses` | Agent 1 **only**: analysis, highlights, **opinion**, reason, model. Never updated by Agent 2 |
| `verification_reports` | Java **findings** + **integrity_result**; Agent 2 **analysis** + **opinion** (separate columns, not a overwrite of Agent 1) |
| `approvals` | decision, actor, hash snapshot |
| `pipeline_jobs` | Queue |
| `audit_events` | Append-only |

**Recommended grain:** one **cycle** (period + started_at) and one **employer_run** (`cycle_id` + `employer_id`) with its own `status`. UI is one card per employer_run.

### 5.1 Statuses (employer_run)

These are what the UI shows. Stored on `employer_run` (or `packets.status` if you keep a single row).

| Code | UI label | Meaning |
| --- | --- | --- |
| `QUEUED` | Queued | Admin started cycle; work not started |
| `CALCULATING` | In progress | CalcEngine + PacketService |
| `PREPARED` | Prepared | Packet + hash stored; Java numbers frozen |
| `NARRATING` | In progress | Agent 1 writing (skipped if agents off) |
| `PREPARED_WITH_ANALYSIS` | Prepared | Packet + Agent 1 text |
| `VALIDATING` | Validating | Java re-calc + evidence; then Agent 2 if on |
| `READY_FOR_REVIEW` | Ready to approve | Verdict set; human may act |
| `APPROVED` | Approved | Human signed this hash. **Not paid** |
| `REJECTED` | Rejected | Human refused; new packet later |
| `FAILED` | Failed | Technical or calc/hash failure; not approvable |

**Approve** is allowed only in `READY_FOR_REVIEW` and **Java integrity** is not a hard fail (hash / calc / tenant / illegal LOP). Agent 1 `fail` or Agent 2 `fail` **does not** hide the other agent and **does not** by itself merge into one system opinion. They are warnings on the card.  
**Reject** allowed in `READY_FOR_REVIEW` (and optionally `FAILED`).  
Java integrity fail → Approve disabled; admin Rejects or fixes source and re-runs.

### 5.2 Job types

| `job_type` | When enqueued | Worker does |
| --- | --- | --- |
| `FREEZE_PACKET` | Cycle start, per employer | Calc + PacketService; status `PREPARED`; enqueue next |
| `PREPARE_NARRATIVE` | After freeze, if agents on | Agent 1; save analysis |
| `VERIFY` | After freeze (and after narrative if agents on) | VerifyService; optional Agent 2; `READY_FOR_REVIEW` |

If `AGENTS_ENABLED=false`: `FREEZE_PACKET` → enqueue `VERIFY` directly.

---

## 6. Detailed flow

### 6.1 Sequence (one employer in a cycle)

```
Admin                API                 JobPoller           CalcEngine        PacketService      DB
  │                   │                     │                    │                  │              │
  │ POST /cycles      │                     │                    │                  │              │
  │ (pay_period)      │                     │                    │                  │              │
  ├──────────────────►│ insert cycle +      │                    │                  │              │
  │                   │ employer_runs QUEUED│                    │                  │              │
  │                   │ jobs FREEZE_PACKET  │                    │                  │              │
  │                   │ audit cycle_started │                    │                  │              │
  │◄──── 202 ─────────┤                     │                    │                  │              │
  │                   │                     │                    │                  │              │
  │ GET /cycles (poll)│                     │ SELECT SKIP LOCKED │                  │              │
  │                   │                     ├───────────────────►│                  │              │
  │                   │                     │ CALCULATING        │                  │              │
  │                   │                     ├───────────────────┬► run per employee │              │
  │                   │                     │                   │ lines + totals    │              │
  │                   │                     ├───────────────────┴──────────────────►│ freeze JSON  │
  │                   │                     │                   │                  │ SHA-256      │
  │                   │                     │                   │                  ├─────────────►│ packets
  │                   │                     │ PREPARED          │                  │              │
  │                   │                     │ job PREPARE_NARR. │                  │              │
  │                   │                     │   or VERIFY       │                  │              │
  │                   │                     │                   │                  │              │
  │                   │                     │ [Agent 1 optional]│                  │              │
  │                   │                     │ NARRATING → save  │                  │              │
  │                   │                     │ analysis          │                  │              │
  │                   │                     │                   │                  │              │
  │                   │                     │ VALIDATING        │                  │              │
  │                   │                     ├───────────────────┬► second run      │              │
  │                   │                     │                   │ evidence vs pkt  │              │
  │                   │                     │ Java evidence     │                  │              │
  │                   │                     │ [Agent 2 optional; own opinion]     │              │
  │                   │                     │ READY_FOR_REVIEW  │                  │              │
  │                   │                     │ audit verify_done │                  │              │
  │                   │                     │                   │                  │              │
  │ POST approve/     │ recompute hash      │                   │                  │              │
  │      reject       ├────────────────────────────────────────────────────────────┼─────────────►│
  │                   │ APPROVED / REJECTED │                   │                  │              │
  │                   │ audit approval      │                   │                  │              │
```

### 6.2 Step by step

**A. Admin starts a cycle**

1. Chooses a pay period (fixtures may have one period: e.g. 2026-10-01 .. 2026-10-31).
2. `POST /api/cycles` creates a cycle and one `employer_run` per client employer.
3. Inserts a `FREEZE_PACKET` job per employer (`QUEUED`).
4. Returns immediately. UI shows **Queued / In progress**.

**B. Freeze (Java only) — PacketService**

1. Poller claims job (`FOR UPDATE SKIP LOCKED`), sets run status `CALCULATING`.
2. Load **only that** `employer_id`: employees, breakup, LOP, `calendar_days`.
3. **CalcEngine** per employee (PRD §7.5 / §9.4).
4. **PacketService**:
   - Build packet object (schema version, ruleset, payees, totals, disclaimer).
   - Canonical JSON (sorted keys, no extra whitespace policy documented in code).
   - `content_hash = SHA-256`.
   - Insert `packets` row.
5. Status `PREPARED`. Audit `packet_frozen`.
6. Enqueue `PREPARE_NARRATIVE` or `VERIFY`.

PacketService is the **freezer**: calculator already ran; this prints, stamps, and files the register.

**C. Agent 1 (optional)**

1. Status `NARRATING`.
2. Prompt input: packet summary + roster deltas (from DB), **not** a request to compute tax.
3. Output JSON `{ "analysis": string, "highlights": string[], "opinion": "pass"|"pass_with_notes"|"fail", "opinion_reason": string }`. Invalid JSON → job `FAILED`, run `FAILED`.
4. Save on **`preparer_analyses` only**. **Ignore any numbers in the text for persistence of pay.** Pay stays in packet.
5. Enqueue `VERIFY`. Agent 2 must **not** read this row to copy/overwrite opinion.

**D. Verify (Java required, Agent 2 optional)**

1. Status `VALIDATING`.
2. Load packet JSON + source snapshot for that employer.
3. Recompute hash; mismatch → fail.
4. **CalcEngine again** on source (not on packet lines as input).
5. Compare line-by-line (paise) and totals → **evidence rows**.
6. Rules: employer_id match, duplicates, LOP range, proration formula, roster new/removed vs prior packet if any.
7. Optional Java flag if Agent 1 cited a net that is not in the packet (`analysis_inconsistent`) — this is a **finding**, not a rewrite of Agent 1’s row.
8. Persist **Java `integrity_result`** on the verification report: `ok` | `notes` | `hard_fail` (hard_fail = hash/calc/tenant/illegal LOP).
9. If agents on: Agent 2 gets **findings JSON + packet hash only** (not Agent 1’s analysis). Writes its **own** analysis + **own** `opinion`. Save beside Java findings. **Do not UPDATE `preparer_analyses`.**
10. Status `READY_FOR_REVIEW`. Audit `verify_completed`.

**E. Human**

1. UI card: status, hash prefix, totals, LOP count, **Java evidence table**, **Agent 1 analysis + opinion badge**, **Agent 2 analysis + opinion badge** (both always visible when present; disagreement is OK), payee grid.
2. **Approve:** `POST` with `packet_id` + client-shown hash. Server recomputes hash; must match stored; **Java integrity** not `hard_fail`; status = ready. Persist approval. Status `APPROVED`. Banner: **Approved — not paid (v1).** Agent opinions do not have to agree.
3. **Reject:** reason required. Status `REJECTED`. To pay a fix: change source, **new** freeze (new packet id/hash).

**F. Isolation**

Every SELECT/UPDATE in B–E includes `employer_id` from the job. Tests must fail if employer B’s employee appears on A’s packet.

### 6.3 Job poller (Postgres, not Kafka)

```sql
SELECT id FROM pipeline_jobs
WHERE status = 'QUEUED' AND run_after <= now()
ORDER BY id
FOR UPDATE SKIP LOCKED
LIMIT 1;
```

- Mark `RUNNING`, `started_at`, `locked_by` (instance id).
- On success: `DONE`; update `employer_run.status`; insert follow-up job if needed.
- On failure: `FAILED` or retry with `attempts` + backoff (`run_after`). After max attempts: employer_run `FAILED`.
- **Idempotent verify:** if report already exists for `packet_id` + `content_hash`, skip LLM, keep status.

UI **does not** read the job table for labels. It reads `employer_run.status`. Optional: poll every 2s while status is `QUEUED`, `CALCULATING`, `NARRATING`, `VALIDATING`.

### 6.4 Admin cycle board (UI)

- List of employer cards: name, net total, status chip, **Agent 1 opinion** and **Agent 2 opinion** as two chips when present, plus a Java integrity hint if `hard_fail`.
- Click → detail as PRD §13.
- Cycle-level “Run all” starts jobs; no “Approve all” in v1.
- Disclaimer on every packet view.

---

## 7. REST sketch (v1)

| Method | Path | Purpose |
| --- | --- | --- |
| `POST` | `/api/cycles` | Start cycle for all (or listed) employers |
| `GET` | `/api/cycles/{cycleId}` | Cycle + employer_run statuses |
| `GET` | `/api/cycles/{cycleId}/employers/{employerId}` | Packet + Agent 1 + Agent 2 + Java evidence (all side by side) |
| `POST` | `/api/cycles/{cycleId}/employers/{employerId}/approve` | Body: expected hash, optional comment |
| `POST` | `/api/cycles/{cycleId}/employers/{employerId}/reject` | Body: reason |
| `GET` | `/api/cycles/{cycleId}/audit` | Events for that cycle |

No payout endpoints.

---

## 8. Evidence object (admin-facing)

Java verify persists something the UI can render as a table:

```json
{
  "packet_id": "...",
  "content_hash": "...",
  "ruleset_version": "india-teaching-1",
  "integrity_result": "notes",
  "findings": [
    {
      "id": "F-014",
      "severity": "note",
      "code": "LOP_PRESENT",
      "employee_id": "E-22",
      "message": "3 unpaid days; payable gross reduced by floor formula",
      "packet_paise": 8700000,
      "recalc_paise": 8700000,
      "match": true
    },
    {
      "id": "F-015",
      "severity": "fail",
      "code": "NET_MISMATCH",
      "employee_id": "E-09",
      "packet_paise": 5000000,
      "recalc_paise": 4999800,
      "match": false
    }
  ]
}
```

This JSON is **Java evidence only**. Agent 2 may cite `F-014`; it must not invent a third net. Agent 1 and Agent 2 each store their own `opinion` in other tables; Agent 2 never overwrites Agent 1.

---

## 9. Config

| Property | Meaning |
| --- | --- |
| `payguard.agents.enabled` | If false, skip both LLMs |
| `payguard.jobs.poll-ms` | Poller interval (e.g. 1000) |
| `payguard.jobs.max-attempts` | Then `FAILED` |
| `payguard.ruleset-version` | e.g. `india-teaching-1` |
| LLM keys | Env only; never in packet/audit |

---

## 10. Testing (minimum)

| Test | Asserts |
| --- | --- |
| Golden payslip LOP=0 | Known breakup → exact paise |
| Golden payslip LOP>0 | Gross strictly below full month; floor formula |
| Hash stability | Same packet bytes → same SHA-256 |
| Tenant | Employee of B cannot appear in A’s packet |
| Verify replay | Tamper packet net → `NET_MISMATCH` + Java `hard_fail` |
| Approve | Wrong hash rejected; Java `hard_fail` cannot approve |
| Agents independent | After verify, Agent 1 row unchanged; Agent 2 opinion can differ |
| Jobs | Two threads cannot complete the same job twice (SKIP LOCKED) |
| Agents off | Freeze → verify → ready still works |

---

## 11. Later (do not build now)

- Kafka outbox publishing `employer.prepared` (consumer still loads packet **by id from DB**).
- Payout + fund-guard ledger.
- Other buyer types (staffing, T&M, vendor invoice check).
- Multi-state PT, old tax regime, real EPFO.

---

## 12. Document control

| Version | Date | Notes |
| --- | --- | --- |
| 0.1 | 2026-10-30 | Stack, single API, PacketService, Java verify + evidence, optional agents, DB job poll, statuses, flow |
| 0.2 | 2026-10-30 | Agent 1 and Agent 2 are independent opinions; Agent 2 never overwrites Agent 1; admin sees both |
| 0.3 | 2026-10-30 | Concrete MVC package layout for api + web |

Product rules stay in [`PRD.md`](./PRD.md). If this file and the PRD disagree on *what* payroll is, **PRD wins**. If they disagree on *how it is wired*, **this file wins** until the PRD is updated.
