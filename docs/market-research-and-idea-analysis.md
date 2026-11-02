# PayGuard — How this idea actually works (beginner’s research)

**Version:** 0.1  
**Date:** 2026-10-28  
**Audience:** Someone new to payroll, staffing, and IT-services billing  
**Purpose:** Explain the real-world industry, correct a few common mix-ups, show where a two-agent + human-admin product can win, and list questions we must answer before building.

This is **research and positioning**, not a PRD. The product spec lives in [`PRD.md`](./PRD.md) (**v0.3**: India payroll BPO; salary breakup + LOP; PF/TDS in code; stop at admin approval; learning project).

---

## 1. Start here (one page)

Your instinct is right: **many companies sit between “the client who uses the person” and “the person who gets paid.”** Infosys is a famous example of that middle layer. The mistake beginners (and many first product ideas) make is treating **the client’s rate** and **the employee’s salary** as the same number with tax subtracted.

They are **two different pipelines**:

| Pipeline | Who pays whom | Typical name | Used for |
| --- | --- | --- | --- |
| **Billing** | Client company → Infosys (or a staffing firm) | **Bill rate** | Invoice, GST, revenue |
| **Payroll** | Infosys (or staffing firm) → employee | **Pay rate / CTC / salary** | Payslip, PF, ESI, TDS, bank credit |

**Deductions happen on payroll, not on the client invoice.**  
The employee’s take-home is **not** “client hourly rate minus PF.”  
The company’s profit is roughly: *what the client paid* minus *what it cost to employ the person* (salary + employer PF + benefits + overhead).

A useful product in this world is **not** “another greytHR.” Those tools already calculate Indian payroll. The gap that matches your agent idea is:

> For **each client (employer)**, assemble a clear pack of *who worked, what they should be paid, and why*; have a **second check** catch mistakes; show **both packs to an admin** who makes the only decision that matters.

That is **maker–checker** (used in banks) applied to multi-client payroll. Agents speed up the writing. **Rules and a human** must still own the numbers.

---

## 2. The Infosys picture, slowly

### 2.1 Who is who

Imagine Infosys has a contract with a bank (call it **Client A**) and another with a retailer (**Client B**).

- Priya is an Infosys employee. She is **on Infosys’s rolls** (Infosys is her legal employer).
- She is **deployed** to Client A’s project. She may sit in the bank’s office or work remotely. Day-to-day work is for the project; **HR, salary, PF, Form 16** still belong to Infosys.
- Client A does **not** run Priya’s payroll. Client A pays **Infosys’s invoice**.

So Infosys is **one employer** with **many customers**. That is different from “PayGuard runs payroll for many separate companies, each with their own employees.” Both are valid businesses. They need different products. See §4.

### 2.2 How Infosys (and TCS, Wipro, etc.) usually bill

For a large share of work, Indian IT services still use **Time & Material (T&M)**:

1. Priya fills a **timesheet** (hours per day/week).
2. A project manager or client **approves** those hours (leave and non-billable time often stay out).
3. Invoice line ≈ **approved hours × role rate** (sometimes a person-specific rate).
4. **GST** is added on the invoice (commonly 18% on manpower/IT services in India; the client may claim input credit).
5. The client may **deduct TDS** when paying Infosys (tax on the vendor, not Priya’s salary TDS).

There are also **fixed-price** and, increasingly, **outcome-based** deals. Those change *how the client is billed*. They do **not** by themselves change Priya’s monthly CTC formula.

Industry colour (not something you must copy): wage costs at large Indian IT firms are often **around half of revenue** (public reporting has put TCS/Infosys-class wage-to-revenue in the mid-50% range in recent years). That is the *spread* between billing and people cost at company level — not a per-employee formula.

### 2.3 How Priya is paid (this is payroll)

Separately, Infosys HR/payroll has Priya’s **offer / CTC**. Each month:

1. Start from her **salary structure** (basic, HRA, allowances, …).
2. Adjust for **attendance / LOP** (loss of pay), unpaid leave, overtime if any, bonuses, recoveries.
3. Subtract **employee** statutory and tax items (see §3).
4. Credit **net pay** to her bank.
5. Infosys **additionally** pays **employer** PF (and ESI if she is in the ESI band), and later filings (PF, ESI, TDS, Form 16).

**Client A’s rate card does not appear on Priya’s payslip.**  
If Priya is on the **bench** (not billed), Infosys may still pay her salary. That is a company cost problem, not “rate minus deductions.”

### 2.4 Contract staff inside the same industry

Large IT firms also use **staffing vendors**. Then:

- The **staffing firm** is often the **legal employer** of the contractor.
- **TCS/Infosys** is the **client** (and, in Indian labour law, often a **principal employer** with extra duties — §4.2).
- TCS pays the vendor a **bill rate**; the vendor pays the person a **pay rate**.

TCS has publicly been reported as revising **staffing-firm rate cards** (skill-based pricing for contract engineers). Permanent-employee billing is described as more tied to **that person’s compensation**, not the same card.

This vendor layer is where **“many employers / many clients in one payroll factory”** is most real.

---

## 3. India: from “package” to money in the bank

Numbers below are **illustrative of how the machine works**, not legal advice. Indian labour and tax rules moved in 2025–2026 (labour codes; PF wage ceiling discussions). A real product must use a **rules engine or a licensed payroll engine**, not an LLM’s memory of rates.

### 3.1 Three numbers people confuse

| Term | Meaning |
| --- | --- |
| **CTC (Cost to Company)** | Everything the **employer** spends on you in a year: gross salary **plus** employer PF, gratuity provision, employer ESI, sometimes insurance. |
| **Gross / earnings** | What the payslip shows before **employee** deductions. |
| **In-hand / net** | What actually hits the bank: gross minus employee PF, ESI (if any), professional tax, TDS, and other recoveries. |

Employer PF is **part of CTC** and is **not** taken out of the employee’s gross the way employee PF is. Beginners often subtract employer PF twice.

### 3.2 Typical monthly deduction stack (employee side)

- **Employee PF:** commonly **12% of basic (+ DA)**. There is a **statutory wage ceiling**; contribution above the ceiling is a **policy choice** of the employer. Ceiling amounts have been changing (older common figure ₹15,000 wage; industry writing in 2026 also cites a higher ceiling). **Do not hard-code this in an agent prompt.**
- **ESI:** if wages are under the ESI ceiling (commonly discussed as **₹21,000 gross/month**), employee ~**0.75%**, employer ~**3.25%**.
- **Professional tax:** **state** tax, not all states, small amount (Constitutional cap in the low thousands per year).
- **TDS (income tax):** employer estimates annual tax (new vs old regime, declarations, standard deduction) and deducts **monthly**. Getting this wrong is a **company** problem, not only the employee’s.
- **Others:** loans, food coupons, LWF in some states, etc.

### 3.3 Employer extra cost (not a “deduction” from in-hand)

- Employer PF (~12% on the chosen wage base) + small admin/EDLI lines  
- Employer ESI if applicable  
- **Gratuity** accrual (often modeled around ~4.81% of wages in costing)  
- Insurance, bonuses, variable pay  

So: **in-hand ≠ CTC ÷ 12**, and **in-hand ≠ bill rate − 18% GST**.

### 3.4 Tiny numeric sketch (for intuition only)

Suppose monthly **gross** ₹1,50,000, PF only on a **capped** wage, no ESI (over ceiling), ignore TDS for a low-tax case:

- Employee PF might be a few thousand rupees.  
- Net is still close to gross if tax is nil.  
- CTC is **higher** than gross because of employer PF and gratuity.

A **bill rate** for the same person on a staffing contract might be **~15–30% above pay** once statutory load and vendor margin are included (industry staffing write-ups; not a law). GST sits **on the invoice to the client**, not on the payslip.

**Worked staffing economics (from public explainers, rounded):**  
gross pay ₹1,50,000 → loaded employment cost ~₹1,57,000 → client **pre-GST bill** ~₹1,88,000 → invoice with 18% GST ~₹2,22,000.  
The employee never sees ₹1,88,000. The client never pays Priya directly.

---

## 4. Four businesses people mix up (choose one)

Your sentence *“a company like Infosys… service to multiple companies… rates decided by those employers… employee is paid after deductions”* can mean **four different products**. PayGuard can only be one of them at first.

### 4.1 IT services firm (Infosys itself)

| | |
| --- | --- |
| **Legal employer** | Infosys |
| **Clients** | Many, but they are **customers**, not payroll employers |
| **Rate** | Bill rate on the **SOW / rate card**; salary on the **offer letter** |
| **Hard problem** | Timesheet → invoice vs HR payroll **don’t match** (wrong hours, wrong role, person billed after exit) |
| **Existing tools** | SAP/Workday/Darwinbox internally; PSA (project) tools; not a startup-shaped “payroll for many employers” |

**PayGuard fit:** a **control tower** — “for each client account this month, here is billed vs paid vs exceptions” — **not** replacing Infosys payroll.

### 4.2 Staffing / third-party payroll (closest to “rates + many companies”)

| | |
| --- | --- |
| **Legal employer** | The **staffing company** (usually) |
| **Clients** | Many companies where people sit |
| **Rate** | Client **bill rate**; employee **pay rate**; markup in the middle |
| **Hard problem** | Attendance at client site → pay register → PF/ESI actually deposited → invoice the client. Errors and fraud are common. |
| **Indian law extra** | Under **CLRA** (contract labour), the **client is principal employer**. If the contractor doesn’t pay wages/PF/ESI, **liability can fall on the client**. So clients demand **proof** before paying the vendor. |

Monthly cycle used in the industry (condensed):

1. Joint sign-off on **attendance / hours** (not “trust the vendor invoice”).  
2. Compute wages vs **minimum wage**, attendance, OT.  
3. Run payroll, generate payslips.  
4. Deposit PF/ESI; keep **TRRN** and ESI evidence.  
5. Client **checks last month’s compliance** before releasing **this** month’s vendor payment.  
6. Wages often expected by **~7th–10th** of the month (Payment of Wages practice).

**PayGuard fit (strong):** software for the **staffing firm** *or* for the **principal employer’s checker**. Agent 1 builds the pack per client. Agent 2 + rules validate against attendance and master data. Admin approves pay / invoice.

### 4.3 Payroll BPO / “we process your payroll”

| | |
| --- | --- |
| **Legal employer** | Each **client company** remains employer |
| **You** | Operate a factory that runs **many separate payrolls** |
| **Rate** | Each client’s own salary structure; you don’t set bill rates |
| **Hard problem** | Never mix Company A’s money or data with Company B; maker–checker; filings per TAN/PF code |
| **Competitors** | ADP, Ramco/Excelity-class bureaus, accountants + greytHR |

**PayGuard fit:** the original [`PRD.md`](./PRD.md) (packets, hash, fund isolation, audit). Good if you want a **multi-tenant payroll control plane**, not Infosys billing.

### 4.4 EOR / PEO (global hiring brands: Deel, Remote, US PEOs)

- **EOR:** provider **is** the legal employer in that country.  
- **PEO:** mostly **US co-employment**; you already have a US entity.  
- **“Global PEO”** in marketing is often actually **EOR**.

This is a **heavily capitalized, legally heavy** market. Not a first version of a student/early project.

---

## 5. What Agent 1 and Agent 2 should actually do

Your idea:

> Agent 1 reads inputs per employer and says what to pay each employee.  
> Agent 2 validates that analysis against employer data.  
> Admin sees both and decides.

That maps to banking **maker–checker**. It is a good idea **if** you split **language** from **math**.

### 5.1 Inputs Agent 1 should read (staffing / multi-client payroll)

Per client (employer), for one month:

- Employee master (name, id, client, pay rate or CTC, location/state, PF UAN, bank last-4)  
- **Attendance / timesheets** (approved)  
- Rate card or salary structure  
- Last month’s packet (to spot new/removed people and huge jumps)  
- Client wallet / “can we pay from this client’s funds?” if you hold money  
- Policy: LOP rules, OT, incentives, recoveries  

**Output of Agent 1 (a packet, not a chat):**  
for each employee — earnings, each deduction line, net, employer statutory cost, and a **short explanation** (“3 days LOP”, “new joiner from 12th”, “OT 8 hours”). Plus client totals.

**Important:** In v1, Agent 1 should **fill a template from data**, not invent CTC. If the source says ₹50,000 and the model writes ₹80,000, **code must reject the packet**.

### 5.2 What Agent 2 should validate

Against **the same source files**, not against Agent 1’s story:

- Every person belongs to **this** client only  
- Hours ≤ calendar / approved timesheet  
- Arithmetic: earnings − deductions = net; column totals match  
- No duplicate employee or duplicate bank account in the same run  
- New joiners / exits vs last month  
- Net or gross change above a threshold (e.g. 20%)  
- PF/ESI **eligibility** vs wage (rule engine)  
- Bill rate vs pay rate vs contract (if you are in staffing)  
- Enough **client-specific** funds  

Agent 2’s **written** response: “3 flags, here is what a human should look at.”  
**Pass / pass with notes / fail** should be decided by **rules**, with the essay attached.

### 5.3 What the admin screen must show

One **card per client**, not a 200-row spreadsheet dump:

- Verdict and flags  
- Agent 1 vs Agent 2 **disagreements** highlighted  
- Total to pay vs funds  
- Drill-down to one employee  
- Approve / reject **this client only**

No money movement until that click. That matches your original PayGuard story and is how serious payroll teams already think.

### 5.4 What agents must never do

- Call the bank or UPI  
- “Fix” a total to make the sheet balance  
- Approve  
- Mix Client A and Client B in one payout batch  

---

## 6. Market research (how the idea can best work)

### 6.1 The big payroll market (you will not beat this head-on)

- Cloud payroll software is a **large, growing** global market (industry reports put it in the **tens of billions of USD**, with ~10% class growth — treat exact billions as directional).  
- Global names: **ADP, Workday, Paycom, Paylocity, SAP, Rippling, Gusto, Deel**.  
- **India is local:** greytHR, **Keka**, Zoho Payroll, Darwinbox, RazorpayX Payroll, PeopleStrong, Pocket HRMS, Ramco. US tools like Gusto generally **do not** do Indian PF/ESI.  
- Buyers pay for **statutory correctness** (PF, ESI, PT, TDS, Form 16), attendance, and payslips. That race is already lost for a greenfield agent app.

**Conclusion:** Do not pitch “AI payroll calculator for India.” Pitch **verified multi-client pay packets with a human lock**.

### 6.2 Where software is weaker today

| Gap | Who feels it | Why agents + checker help |
| --- | --- | --- |
| Spreadsheets across **20 clients × 200 contractors** | Staffing firms, small BPOs | Packet per client; exceptions in English |
| Principal employer **doesn’t trust** vendor pay registers | Factories, IT parks, GCCs | Second-pass validation before invoice approval |
| T&M **hours billed ≠ hours paid** | IT services finance | Cross-check two systems |
| Last-minute roster changes | Everyone | Diff vs last month, force re-approval |
| Pooled bank file paying the wrong company | Multi-entity groups, bureaus | Fund isolation (your original idea) |

Vendor Management / **CLMS** tools in India already do parts of attendance + invoice approval. They are often **operations portals**, not “two independent analyses + bind approval to an exact hash.” That binding is still a sharp edge.

### 6.3 Who would pay (realistic first customers)

**Best first beachhead (recommended):**  
an **Indian staffing / third-party payroll** operator with 3–30 client companies, or a **chartered accountant / payroll BPO** running monthly payroll for several SMEs.

Why:

- They already think in **per-client runs**.  
- Pain is **exceptions and proof**, not “I wish I had Workday.”  
- You can demo with **fake data** and never touch live EPFO in v1.  
- The Infosys story is easy to **explain**, even if Infosys itself will never buy a student product.

**Worse first beachhead:**

- Compete with Darwinbox inside Infosys.  
- Become Deel (EOR).  
- Go live on **actual salary payments** in month one (legal + RBI + trust).

### 6.4 How “best work” looks as a product wedge

**Phase A — Review copilot (safest, still useful)**  
Inputs: CSV/Excel of attendance + salary masters.  
Outputs: per-client analysis, validation report, admin approve/reject.  
Payments: **simulated**. You still prove the workflow.

**Phase B — Control plane on top of a real engine**  
Let **Zoho / greytHR / a rules library** compute PF/TDS. PayGuard **prepares, verifies, freezes, audits**. You do not maintain the entire Income-tax Act.

**Phase C — Only if you have lawyers and a company**  
Bank files, EPFO, client-segregated accounts.

This is how the idea “best works”: **narrow buyer, frozen packets, human final call, don’t rebuild ADP.**

### 6.5 Pricing intuition (later)

Staffing and BPO buyers understand **per employee per month** or **per client per run**. Global EOR is hundreds of USD per employee; Indian SMB payroll software is often **hundreds to a few thousand rupees per month** for a small company. Your v1 should not worry about price. Your **story** should be: fewer wrong pays and faster admin review, not cheaper than greytHR.

---

## 7. Risks (plain language)

| Risk | Why it matters | What to do |
| --- | --- | --- |
| LLM invents a salary | People get under/overpaid | Packet must match source; code checks |
| You claim “we do PF legally” | Wrong PF is an EPFO and employee dispute | Rules engine or partner; agents only explain |
| Principal-employer liability | Client can be dragged into vendor PF failure | Evidence pack (payslips, TRRN) in the packet |
| Mixing client money | The original PayGuard nightmare | Separate ledgers; block mixed batches |
| Labour codes / PF ceiling changes | Prompts go stale | Versioned rules, not model trivia |
| PII (Aadhaar, bank, salary) | DPDP Act; trust | Mask in UI; never log full accounts in prompts |

---

## 8. What we recommend you build (given “very initial idea”)

For an early project that still looks like **PayGuard**:

1. **Setting:** a fictional staffing company, **Aarohan Staffing**, with **3 client companies**.  
2. **Each client** has 8–15 people, messy on purpose (new joiner, exit, LOP, overtime, duplicate, one client short of funds).  
3. **Agent 1** produces a **payroll packet + explanation** per client from JSON/CSV masters.  
4. **Agent 2 + Python rules** produce a **validation report**.  
5. **Admin UI:** three cards, approve/reject, no pay on fail.  
6. **Ledger:** pretend UPI/NEFT; prove isolation.

That teaches you the domain **and** stays honest. It is closer to **§4.2 staffing** than to “we are Infosys.” You can still **tell** the Infosys story in a pitch: *same control idea, smaller buyer.*

If your heart is **Infosys T&M**, rename the objects: Client Account, billed hours, bill rate, **vs** employee CTC. Then Agent 1’s “what should be paid” is **salary payroll**, and a **fourth** column is **what we invoiced**. That’s a **reconciliation** product (finance), slightly different from payroll.

---

## 9. Glossary

| Word | Meaning |
| --- | --- |
| **Legal employer** | Whose name is on the appointment letter and PF |
| **Principal employer** | Client using contract labour; extra Indian compliance duty |
| **Bill rate** | What the **client** is charged |
| **Pay rate** | What the **worker** is paid (gross), before employee deductions |
| **Markup / margin** | Gap used for statutory load, overhead, vendor profit (people quote these inconsistently — always ask “percent of what?”) |
| **T&M** | Billing by time × rate |
| **CTC** | Employer’s total annual cost of that person |
| **Maker–checker** | One party prepares, another checks, a principal approves |
| **EOR / PEO / BPO** | See §4 |
| **CLRA** | Contract Labour (Regulation and Abolition) Act — India |
| **EPFO / UAN** | Provident fund authority / employee PF number |
| **TRRN** | Proof number when PF is paid |

---

## 10. Sources (starting points)

Use these to go deeper; they are not a substitute for a payroll consultant.

- Bill rate vs pay rate (staffing quote anatomy): [supersourcing.com/blog/bill-rate-vs-pay-rate](https://supersourcing.com/blog/bill-rate-vs-pay-rate/)  
- T&M timesheet-to-invoice reconciliation (India IT): [terra-insight.com — T&M billing reconciliation](https://www.terra-insight.com/insights/time-and-material-billing-reconciliation-india/)  
- TCS staffing rate-card revision (news context): [Livemint, TCS staffing firms](https://www.livemint.com/companies/news/tcs-to-revise-rates-for-staffing-firms-11698164949368.html)  
- Third-party payroll / legal employer vs principal employer (vendor explainer): [tmservices.co.in — third-party payroll](https://tmservices.co.in/third-party-payroll-services/)  
- Contractor payment + PF evidence before releasing vendor pay: [MYND — contractor payment process](https://www.myndsolution.com/best-practices/setting-up-contractor-payment-process-in-staffing-and-manpower-management-in-india/)  
- EOR vs PEO vs staffing: [eorhq.com decision matrix](https://eorhq.com/guides/eor-vs-peo-vs-staffing/)  
- India payroll software landscape: comparisons from Darwinbox/Zendikt-style 2026 roundups (greytHR, Keka, Zoho, Darwinbox)  
- CTC vs in-hand / PF ESI PT TDS explainers: e.g. [itrsaral.in salary breakup](https://itrsaral.in/tools/salary-breakup-calculator), [openaccountants.com India payroll](https://www.openaccountants.com/skills/india-payroll)

Statutory **rates and ceilings change**. Recheck EPFO, ESIC, and Income Tax Department before any real calculation.

---

## 11. How this changes the earlier PRD

| Earlier PRD | After this research |
| --- | --- |
| “Multi-employer payroll bureau” | Still valid — that’s **§4.3** |
| Amounts already stored | Fine for a prototype; **real** staffing needs attendance → calc |
| Infosys-like narrative | Explain as **one employer, many clients** *or* switch buyer to **staffing** |
| Don’t build a tax engine | Still correct; even more true after seeing PF/TDS complexity |
| Fund isolation | Essential if you ever payout; also essential as **invoice vs client cost centre** |

**Locked in PRD v0.3:** buyer = payroll BPO; legal employer = each client company; **India only**; PF/TDS calc in code; **salary breakup (B)**; **LOP days**; v1 ends at approve/reject; learning/exploration. Other buyer types stay a later expansion.

---

## 12. Questions — status

| # | Topic | Status |
| --- | --- | --- |
| Buyer | Payroll BPO | **Locked** |
| Legal employer | Each client company | **Locked** |
| Country | India only | **Locked** |
| Calc | PF/ESI/PT/TDS in code | **Locked** |
| Payout | Stop at admin approval | **Locked** |
| Intent | Learning / exploration | **Locked** |
| Later | May expand to all 4 buyer types | Noted in PRD Phase 3 |
| Salary input | **(B)** monthly breakup: basic, HRA, other allowances | **Locked** |
| Attendance | **LOP days**; prorate then deduct | **Locked** |

### What “rate” means in this BPO (resolved)

There is **no Infosys bill rate**. Each client company stores **basic + HRA + other allowances** per employee. PayGuard prorates for **LOP days**, then calculates Indian deductions. It does not invent a CTC split.

---

## 13. Bottom line

- Infosys-like firms **bill clients on rates and hours**; they **pay employees on CTC and labour law**. Those are not the same pipeline.  
- **Deductions** (PF, ESI, PT, TDS) apply to **payroll**. GST/TDS on the **vendor invoice** is a different tax story.  
- Your **two agents + admin** idea is strongest as a **maker–checker control plane** for **many client payrolls** (staffing or BPO), not as a new full payroll suite.  
- The market is crowded on **calculation**; it is still clumsy on **explanation, independent checking, frozen approval, and not mixing clients**.  
- v1 demo: **three Indian client employers**, salary breakup + LOP, two agents, admin approve/reject, **no payout**.

Scope is now frozen in [`PRD.md`](./PRD.md) v0.3. Next step when you want to build: packet schema + calc engine with golden payslip tests.
