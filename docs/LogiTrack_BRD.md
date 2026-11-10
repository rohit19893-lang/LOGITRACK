# LogiTrack — Business Requirements Document (BRD)

**Document type:** Business Requirements Document
**Product:** LogiTrack — mobile operations app for a road-freight brokerage / transport business
**Business owner:** Mohit (owner-operator)
**Version:** 1.0 (Handoff edition)
**Date:** 10 November 2026
**Companion documents:** `LogiTrack_Functional_Spec.md` (the build specification), `logitrack_prototype.html` (working reference prototype), `LogiTrack_Handoff_Brief.md` (covering note)

---

## 0. How to read this document

This BRD states **what the business does and why**, in business language. It is the "why" document. Its companion, `LogiTrack_Functional_Spec.md`, states **exactly what to build** — every entity, field, screen, formula and validation. Build decisions should be taken from the Functional Spec; disputes about *intent* should be resolved from this BRD.

Every monetary figure in the worked scenarios of Section 9 was produced by running the reference prototype's actual settlement engine. They are not illustrative approximations — they are the exact outputs the replacement system must reproduce.

Terminology note: this is an Indian road-freight business and the domain vocabulary is Hindi/Urdu-derived trade language. Section 7 is a glossary. **Read the glossary before Section 8.** A developer who does not know what a "bilty" or "munshiyana" is cannot correctly implement this system, and guessing produces plausible-looking but wrong software.

---

## 1. Business context

Mohit runs a road-freight transport business in India. The business is primarily a **freight broker / commission transporter** rather than a pure fleet owner: its core economic activity is matching a customer who needs goods moved with a truck that can move them, and earning the spread plus commission between what the customer pays and what the truck is paid.

The business handles bulk commodity movements — the representative load is dolomite powder moving from the mining belt around Katni (Madhya Pradesh) to paint and industrial-cleaning manufacturers in Patna (Bihar) — using large multi-axle trucks, typically 22-wheelers, usually arranged through truck suppliers and brokers rather than owned outright.

Two distinct money relationships run in parallel on every single trip, and conflating them is the single largest source of error in the existing paper process:

1. **The party side** — what the customer (consignor, consignee, or a third party) owes the business for moving the goods, calculated at the **booking rate**.
2. **The truck side** — what the business owes the truck owner/driver for doing the move, calculated at the **hire rate**.

The business's gross margin on a trip is the difference between these two, plus or minus commission and a long tail of operational charges (halting, weighing, labour, shortage deductions, TDS). The app's central purpose is to compute, track and settle both sides correctly and separately.

The complicating factor — and the reason generic accounting software does not fit — is that **who physically hands money to whom varies per trip**. On a "To-Pay" booking the customer pays the driver directly in cash at the delivery point, which means the business has *already lost control of its own margin* and must afterwards **recover** money *from the driver*. On a "Paid" booking the business pays the truck itself and bills the customer separately. The arithmetic reverses direction between these two cases. Getting this reversal right, in both directions, including the edge case where a partial collection flips the business from recovering money to owing money, is the heart of the requirement.

### 1.1 Operating scale

- Fleet/suppliers: a working set of roughly 5–20 trucks, mostly arranged rather than owned.
- Trips: a modest daily volume — tens per month, not thousands. **This is not a high-throughput system.** Correctness, clarity and speed of single-record entry matter enormously; bulk-processing throughput does not.
- Users: the owner plus one to two office staff. See Section 3.

### 1.2 How work arrives today

Work arrives by phone. A party calls needing a truck, or a supplier calls offering one. The office must capture a booking in under a minute, while still on the call, from a mobile phone — often with incomplete information (a truck number but no driver yet; a party name that is not yet in any register). **Any design that requires complete, validated master data before a booking can be recorded will be rejected by its users and abandoned.** This constraint drives a large number of the functional decisions in the companion spec, particularly the pervasive "accept free text now, link to a master record later" pattern.

---

## 2. The problem being solved

The business currently runs on paper and memory:

A handwritten **bilty book** (carbon-copy consignment note pads) is the legal and commercial record of each consignment. It is slow to write, illegible when carbon-copied, impossible to search, physically vulnerable, and cannot be sent to a customer without photographing a page.

A **truck-account notebook** holds the per-trip settlement arithmetic with each truck. This is where the real money lives, and it is where the real errors live. Because the arithmetic reverses between To-Pay and Paid bookings, because advances given at loading time are easy to forget by settlement time, and because deductions accumulate across a trip, reconciliation disputes with drivers and suppliers are routine. A forgotten ₹10,000 advance is a ₹10,000 loss.

**Master data lives in phone contacts and in the owner's head.** Truck documents (RC, insurance, permit, fitness, pollution), driver licences and party bank details are photographs scattered across a phone gallery, retrievable only by scrolling.

**Nothing is shareable.** Sending a party their loading details, or sending a truck owner their account statement, means retyping it into WhatsApp from a notebook, every time.

**There is no position.** The owner cannot answer "how much is owed to us right now, and by whom" without manually working through the notebook.

### 2.1 Quantified pain points

| # | Pain | Business consequence |
|---|---|---|
| P1 | Settlement arithmetic done by hand, direction-dependent | Direct revenue leakage; recurring disputes with drivers and suppliers |
| P2 | Advances given at loading forgotten at settlement | Double payment — a pure, unrecoverable cash loss |
| P3 | Handwritten bilty | Slow; illegible; not shareable; physically at risk |
| P4 | Truck/driver/party documents unfindable | Delays at checkposts and at loading; repeated requests to parties |
| P5 | No consolidated outstanding position | Cash-flow blindness; delayed follow-up on recoveries |
| P6 | Every outbound communication retyped | Hours per week; transcription errors |
| P7 | Internal margin visible on party-facing documents | Commercially damaging — the hire rate must never leak to the party |

Pain point **P7** deserves emphasis because it shapes the document design throughout: the business's buy price (hire rate) and sell price (booking rate) sit on the same trip record, but the customer-facing document must show only what that customer is permitted to see. The system must make leaking the internal rate difficult by default rather than merely possible to avoid.

---

## 3. Stakeholders and users

| Stakeholder | Role | Relationship to the system |
|---|---|---|
| **Owner (Mohit)** | Admin. Books trucks, negotiates both rates, approves settlements, needs the financial position. | Primary user. Full access. Uses it daily on a phone. |
| **Office staff (1–2)** | Data entry: bilties, loading challans, POD records, payment entries. | Secondary users. Non-technical. Must be productive with no formal training. |
| **Truck owners / suppliers** | Supply trucks. Receive settlement statements. | **Not users.** Receive output via WhatsApp. |
| **Drivers** | Drive. Collect To-Pay freight. Deliver POD. | **Not users — explicit decision.** Communicate by phone/WhatsApp; staff enter data on their behalf. |
| **Parties (consignor / consignee / third party)** | Customers. Receive bilties and loading slips. | **Not users.** Receive output via WhatsApp. No portal. |
| **Brokers** | Arrange trucks or loads; earn brokerage. | Not users. Recorded as a party role. |

**Design implication — the system is single-sided.** Exactly one organisation uses it, with at most three humans. There is no counterparty-facing login, no driver app, no customer portal. Every external communication is an outbound WhatsApp message or PDF composed by a staff member. This is a deliberate decision that keeps the product dramatically simpler, and it should not be revisited during replication.

---

## 4. Business objectives and success measures

| # | Objective | Success measure |
|---|---|---|
| O1 | Eliminate settlement arithmetic errors | Every settlement figure computed by the system; zero hand arithmetic. No advance can be omitted from a settlement it belongs to. |
| O2 | Replace the handwritten bilty book | 100% of consignments raised digitally; each exportable as a branded PDF and shareable to WhatsApp within seconds. |
| O3 | Make the financial position visible on demand | Total pending recovery and total pending payable available in two taps, at any time. |
| O4 | Make master data instantly retrievable | Any truck, driver or party record with its documents found by typing three characters. |
| O5 | Eliminate retyping for outbound communication | Every outbound message composed by the system with per-field selection of what is disclosed. |
| O6 | Remain usable by non-technical staff | A new staff member raises a correct loading challan on their first day without training. |
| O7 | Never leak internal commercials | Internal hire rate and margin excluded from party-facing output by default, with disclosure only on an explicit per-document opt-in. |

---

## 5. Scope

### 5.1 In scope

**Master data:** trucks (with owner details and documents), drivers, parties (any counterparty role), products/materials, locations, application users, charge types.

**Transactions:** the loading challan (booking), the bilty/GR (consignment note), the freight challan (truck settlement account), the POD record (proof of delivery), and standalone payment entries.

**Money:** per-trip settlement computation in both directions, advances, a dynamic charge list, shortage deduction, TDS, a running payment/recovery ledger per trip, and settlement closure.

**Output:** branded bilty document and PDF export; field-selective WhatsApp sharing for party, driver, truck, loading challan and truck settlement.

**Operational structure:** a business-wide company profile used for document branding, and multi-account support allowing several entirely separate businesses in one installation.

**Reporting:** commission earned, pending recoveries, pending payables, payment summary.

### 5.2 Out of scope — and why

These are deliberate exclusions, each previously considered and rejected. A replicating team should not reintroduce them without the owner's explicit instruction.

| Excluded | Reason |
|---|---|
| Driver-facing app or login | Keeps the system single-sided and simple. Drivers communicate by phone. |
| GPS / live tracking | Status is updated manually. Not a business need at this scale. |
| Online payment collection / gateway | Payments are *recorded* after the fact (cash/UPI/bank), never collected in-app. Avoids PCI scope entirely. |
| GST / tax computation on the bilty | The business is not GST-registered and charges no GST. Revisit only on registration. |
| Customer portal | Parties receive documents; they do not get accounts. |
| Expense & fuel management | Confirmed as a real future need; explicitly deferred. |
| Enforced role-based permissions | User *records* exist with roles; enforcement is deferred. The data model must support it from day one so it is not a rework. |

### 5.3 Deliberately deferred (Phase 2+)

Expense & fuel management; the full admin-vs-staff permission matrix; a change-audit log recording every create/edit with who and when; the party-side receivables ledger (the counterpart to the existing truck-side settlement); trip register and truck/driver-wise revenue reports.

The **party-side receivables ledger** is the most significant of these. The current system deliberately tracks only the truck side of each trip in depth. What the party owes the business is visible on the bilty but has no dedicated ledger, no ageing and no statement. This is a known, accepted gap, and it is the most likely first major extension — Section 13 of the Functional Spec treats it as the primary integration opportunity with the parent application.

---

## 6. Constraints and assumptions

**Platform.** Android and iOS both required. The owner and staff work from phones. The reference prototype is a mobile-viewport web application; the production build's platform is the receiving team's decision.

**Offline operation is a first-class requirement, not an enhancement.** Staff are at depots, mines and roadside with poor or absent connectivity. Creating and editing records offline, with automatic sync on reconnection, must be designed into the data layer from the start. Retrofitting offline-first onto a server-round-trip architecture is a rewrite, so this decision must be taken before development begins.

**Currency and locale.** All money is Indian Rupees, formatted in the Indian digit-grouping convention (`₹1,20,500`, not `₹120,500`). Dates display as `10 Nov 2026`. Weights are metric tonnes (MT) with decimals. Mobile numbers are Indian 10-digit, normalised to `91XXXXXXXXXX` for WhatsApp deep links.

**No GST.** No tax fields anywhere on the bilty.

**Assumptions.** WhatsApp is installed on the users' phones and is the business's universal communication channel. The business continues to operate without GST registration. Settlement with any one truck is per-trip, not a rolling running account across trips. Trip volume stays in the tens per month.

---

## 7. Domain glossary

**This section is required reading.** These are Indian road-freight trade terms. Implementing the system without understanding them produces subtly wrong software.

| Term | Meaning |
|---|---|
| **Bilty** (also **GR**, **LR**, consignment note) | The core commercial and quasi-legal document for a consignment: who is sending, who is receiving, what goods, which truck and driver, what freight. Traditionally handwritten in a carbon-copy book. Travels with the driver and is the party's proof of the contract of carriage. |
| **Loading Challan** (also loading slip) | The fast booking record created when a truck is arranged, typically while still on the phone. Captures truck, party, route, both rates and weight. Precedes and may produce the bilty. Not every booking needs a bilty. |
| **Freight Challan** | **Not** a party document. This is the **truck's own settlement account** for one booking — what the business owes the truck, or must recover from it, after all charges and advances. Internal only. The digital replacement for the truck-account notebook. |
| **POD** (Proof of Delivery) | The signed/photographed confirmation that goods were delivered. Gates final settlement: no POD, no closure. |
| **Hire Rate** | What the **business pays the truck** — the buy price. **Internal. Must never appear on a party-facing document unless explicitly disclosed.** |
| **Booking Rate** | What the **party pays the business** — the sell price. The margin is Booking Rate − Hire Rate. |
| **Bilty Rate** | An override of the booking rate for billing purposes, used when the party or broker has agreed a different rate for this specific load than the internally recorded booking rate. Changes only the party side; never affects what the truck is paid. |
| **To-Pay** | Freight is collected **at the destination, by the driver, from the party, in cash**. The business therefore does not touch its own margin at delivery and must **recover** commission and margin *from the driver* afterwards. |
| **Paid** | Freight already settled by/with the business; the business pays the truck itself. |
| **To-Be-Billed** | The business will invoice the party later and pays the truck itself in the meantime. **Economically identical to Paid from the truck's perspective**, and the system deliberately treats the two as one settlement branch. |
| **Advance** | Money given to the driver before or during the trip — cash at loading, a UPI transfer en route, or diesel filled at a pump. Deducted from the final settlement. **Forgetting an advance is the business's single most expensive routine error.** |
| **Shortage** | Delivered weight less than loaded weight. Recovered from the truck, usually at the hire rate for the missing quantity, plus a penalty ("cutting") on the portion beyond an agreed free tolerance. |
| **Shortage Tolerance** | A free allowance of shortage (in MT) below which no penalty cutting applies — natural handling loss. |
| **Cutting Rate** | The penalty rate (₹ per MT) applied to shortage **beyond** the tolerance, over and above recovering the freight on the missing weight. |
| **Commission** | The business's earning on arranging the trip. On a To-Pay booking it must be recovered from the driver, since the driver collected the whole freight. |
| **Munshiyana** | A small clerical/handling fee, traditionally the munshi's (clerk's) cut. Recovered from the truck. |
| **Halting / Detention Charges** | Compensation to the truck for being detained beyond agreed free time at loading or unloading. **Payable to** the truck. |
| **Kanta Charges** | Weighbridge/weighing fees. |
| **Dala** | Loading/unloading labour charges. |
| **Brokerage** | A broker's fee for arranging the truck or the load. |
| **TDS** | Tax Deducted at Source — a statutory deduction withheld from the truck's payment. Flat amount or a percentage of truck freight. |
| **Consignor** | The sender of the goods. |
| **Consignee** | The receiver of the goods. |
| **Third Party** | A counterparty who is neither sender nor receiver but is involved — commonly a broker or a referrer — and who may be the one who actually settles the freight. |
| **Supplier** | The party who supplied the truck to the business (as distinct from the truck's registered owner). |
| **Party** | Any external counterparty. One master list covers all the roles above; the role is determined by *which field* the party is used in, not by a permanent type on the record. |
| **Referred By** | Who introduced this party or truck to the business. Relationship provenance; used for informal relationship tracking. |

---

## 8. The end-to-end business lifecycle

A trip passes through up to five records. Understanding their relationships and the two alternative paths is essential.

### 8.1 The documents and how they relate

```
                        LOADING CHALLAN
                      (the booking — always first)
                                │
              ┌─────────────────┴──────────────────┐
              │                                    │
      Bilty Needed = Yes                   Bilty Needed = No
              │                                    │
          BILTY / GR                               │
    (party-facing consignment note)                │
              │                                    │
              └─────────────────┬──────────────────┘
                                │
                        FREIGHT CHALLAN
             (the truck's settlement account — internal)
                                │
                          POD RECORD
                  (delivery confirmation — gates closure)
                                │
                   PAYMENT / RECOVERY LEDGER
                 (actual money movement, by date)
```

**Rule:** exactly **one** freight challan per loading challan, never a duplicate, regardless of which path reached it. Both paths converge on the same record.

### 8.2 Path 2 — with a bilty (the common case)

1. A party calls needing a truck. The office raises a **loading challan**: truck number, party, loading and unloading points, **hire rate** (what the truck gets), **booking rate** (what the party pays), weight, and any special notes. Both rates are recorded per MT or as a fixed trip amount, independently. "Bilty Needed?" is answered **Yes**.
2. The loading slip is shared to the party on WhatsApp — the office chooses field by field what to include. The hire rate is never among the offered fields.
3. A **bilty** is created from the challan, inheriting everything already captured. The office adds what the booking call did not have: confirmed consignor and consignee as separate parties, driver, goods lines with product, quantity and weight, and the **payment type** (To-Pay / Paid / To-Be-Billed) with **who actually pays the freight**. Freight amount is computed from weight × booking rate and remains manually overridable. Advances given at loading and any party-side expenses are logged here.
4. Per-field disclosure toggles decide what the printed bilty shows the party: freight amount, hire rate, booking rate, advances, expenses. Defaults protect the business.
5. A **freight challan** is created from the bilty. It inherits the hire and booking rates, the final weight from the bilty's goods lines, the payment type, who pays the freight, the supplier, the broker, and — critically — **any advance already logged on the bilty is carried across automatically**, and stays in sync if the bilty's advances are edited later. This single automation addresses pain point P2 directly.
6. On the settlement screen the office records what actually happened: unload weight, shortage tolerance and cutting rate, a dynamic list of **charges** each marked *recoverable from* or *payable to* the truck, TDS, diesel, and further advances. The system computes the settlement live but **marks it an estimate until POD**.
7. At delivery a **POD record** is raised with at least one photograph and a POD date. For a To-Pay trip the system **requires** confirmation of how much the driver actually collected from the party. Saving the POD flips the bilty's trip-completed flag, sets the freight challan's POD-received flag, and carries the received weight into the unload weight if that was still blank.
8. With POD received, the settlement figures become final and the office logs actual money movement as dated **payment** or **recovery** entries until the balance reaches zero. The settlement, in account-statement form, can be shared to the truck owner's WhatsApp.

### 8.3 Path 1 — without a bilty

Some bookings need no consignment note. "Bilty Needed?" is answered **No** and the freight challan is created directly from the loading challan. Because there is no bilty to inherit from, the system asks for the two decisions the bilty would have carried — **payment type** and **who pays the freight** — on a dedicated short screen before creating the record. Consignor and consignee both pre-fill from the loading challan's single party field and remain editable on the settlement screen, which for this path also hosts the bilty-rate override that would otherwise live on the bilty.

### 8.4 Where the two paths differ

| Aspect | Path 2 (with bilty) | Path 1 (no bilty) |
|---|---|---|
| Payment type & who pays freight | Decided on the bilty, inherited | Asked on a "Start Freight Challan" screen |
| Final load weight | From the bilty's goods lines | From the loading challan, editable on the settlement screen |
| Consignor / consignee | From the bilty, read-only on the settlement screen | Pre-filled from the challan's party, editable |
| Billing-rate override | On the bilty (`billAtBookingRate` / `biltyRate`) | On the freight challan (`billAtBookingRate` / `billRate`) |
| Advances | Logged on the bilty, auto-synced forward | Logged directly on the freight challan |

### 8.5 The POD gate

POD is a deliberate control, not a formality. Before POD is received the settlement is shown as a live **estimate**; logging money movement and marking the trip fully settled are **blocked**. The business reason is simple: on a To-Pay trip, the recovery due from the driver depends entirely on how much the driver actually collected, and that is unknown until delivery. Settling on an assumption is how money is lost.

A POD may legitimately be recorded **before** its freight challan exists — the records are not created in a fixed order. The system must therefore look for a pre-existing POD at freight-challan creation time and carry its weight, status, note, damage flag and To-Pay collection confirmation forward, retroactively linking the two. This is a real, observed sequence, not a theoretical edge case.

---

## 9. Worked scenarios

These are the specification's executable heart. **Every figure below is the verified output of the reference settlement engine.** A replacement implementation that reproduces all of them is arithmetically correct; one that does not is not, however reasonable its own logic appears.

Common setup unless stated otherwise: load weight 18 MT, hire rate ₹1,000/MT, booking rate ₹1,150/MT, one commission charge of ₹1,000 marked *recoverable from the truck*, one cash advance of ₹5,000 already given to the driver, no TDS, no shortage.

### Scenario A — To-Pay, consignee pays, driver collects in full

The normal, healthy To-Pay trip. The consignee pays the driver the full expected freight at delivery.

| Step | Figure |
|---|---|
| Truck freight (18 MT × ₹1,000) | ₹18,000 |
| Expected party freight (18 MT × ₹1,150, consignee) | ₹20,700 |
| Driver actually collected (confirmed at POD) | ₹20,700 |
| Balance still due from the party | ₹0 |
| Commission (recoverable) | − ₹1,000 |
| **Truck gross** (after charges, shortage, TDS) | **₹17,000** |
| Net position: ₹17,000 − ₹20,700 | **− ₹3,700** |
| **Amount to recover from the driver** | **₹3,700** |
| Recovery balance outstanding | ₹3,700 |
| Settled? | No — until ₹3,700 is recovered |

**Business reading:** the driver collected ₹20,700 but had only earned ₹17,000, so the business must get ₹3,700 back — its ₹2,700 rate margin plus its ₹1,000 commission. Note that the ₹5,000 advance does **not** reduce the recovery in this branch. It appears only in a separate reference figure (`truckFreightFinal`, here ₹12,000), because on a To-Pay trip the driver holds the party's cash and the advance is netted in the course of handing the recovery back. **This asymmetry with the Paid branch is intentional and owner-confirmed. Do not "fix" it.**

### Scenario A0 — the same trip before POD

With no POD confirmation yet, the system assumes full collection and shows the identical ₹3,700 as a **live estimate**, flagged as such. Logging recoveries and marking settled remain blocked.

### Scenario B — To-Pay, driver collects only part of the freight

The consignee pays the driver only ₹15,000 of the ₹20,700 expected, confirmed at POD.

| Step | Figure |
|---|---|
| Expected party freight | ₹20,700 |
| Driver actually collected | ₹15,000 |
| **Balance still due from the party** | **₹5,700** |
| Truck gross | ₹17,000 |
| Net position: ₹17,000 − ₹15,000 | **+ ₹2,000** |
| Amount to recover from the driver | ₹0 |
| **Extra payable to the driver** | **₹2,000** |

**Business reading — this is the critical reversal.** The driver collected less than he had earned, so the business now **owes the driver ₹2,000** rather than recovering anything, *and* is separately owed ₹5,700 by the party. The direction of the trip's ledger has flipped. The system must flip with it: the trip now accepts **payment** entries (business → driver), not recovery entries, and its settlement status must track the ₹2,000 down to zero. An implementation that only ever looks at "amount to recover" will show this trip as fully settled the moment it is created, with no way to record the ₹2,000 actually being paid. **This was a real defect in the reference implementation and was fixed; do not reintroduce it.**

### Scenario B2 — closing out Scenario B

After paying the driver ₹2,000: extra payable ₹2,000, paid ₹2,000, **balance ₹0, settled = Yes**.

### Scenario C — shortage, with the consignor paying the freight

Loaded 20 MT, delivered 19.5 MT. Tolerance 0.2 MT, cutting rate ₹500/MT. Consignor pays.

| Step | Figure |
|---|---|
| Shortage (20 − 19.5) | 0.5 MT |
| Freight on missing weight (0.5 × ₹1,000) | ₹500 |
| Penalty cutting on the excess beyond tolerance (0.3 × ₹500) | ₹150 |
| **Total shortage deduction** | **₹650** |
| Truck freight (20 × ₹1,000) | ₹20,000 |
| Expected party freight — **billed on delivered weight** (19.5 × ₹1,150) | ₹22,425 |
| Truck gross (₹20,000 − ₹1,000 commission − ₹650 shortage) | ₹18,350 |
| **Amount to recover from the driver** | **₹4,075** |

**Note the asymmetry, which is correct:** the truck is paid on **loaded** weight less the shortage deduction, while the party is billed on **delivered** weight. The business absorbs nothing — the shortage deduction is what makes it whole.

### Scenario C2 — the same shortage, but the consignee pays, plus halting

Same shortage. Consignee pays. A ₹1,500 halting charge is *payable to* the truck.

| Step | Figure |
|---|---|
| Shortage | 0.5 MT (deduction ₹650 computed…) |
| …but **skipped on the truck side** because the consignee pays | **₹0 applied** |
| Truck freight | ₹20,000 |
| Expected party freight — **full loaded weight, plus halting** (20 × ₹1,150 + ₹1,500) | ₹24,500 |
| Truck gross (₹20,000 − ₹1,000 + ₹1,500 halting − ₹0 shortage) | ₹20,500 |
| **Amount to recover from the driver** | **₹4,000** |

**Business rule being expressed:** when the **consignee** settles the freight, the shortage is dealt with between the consignee and the truck at the delivery point, so the business does not also deduct it from the truck — that would be recovering the same loss twice. The party is correspondingly billed on the full loaded weight. Additionally, halting charges payable to the truck are passed through into the party's expected freight in this case. **These two consignee-specific behaviours are genuine business rules, not quirks.**

### Scenario D — Paid / To-Be-Billed

16 MT at hire rate ₹1,200/MT. A ₹1,500 halting charge payable to the truck. TDS at 1% of truck freight. ₹10,000 already advanced. ₹5,000 since paid.

| Step | Figure |
|---|---|
| Truck freight (16 × ₹1,200) | ₹19,200 |
| Halting (payable) | + ₹1,500 |
| TDS (1% of ₹19,200) | − ₹192 |
| **Net payable to the truck** | **₹20,508** |
| Advances already given | − ₹10,000 |
| **Final payable to the truck** | **₹10,508** |
| Paid so far | ₹5,000 |
| **Balance payable** | **₹5,508** |

**Business reading:** here the business pays the truck itself, so advances **do** reduce what remains payable. Contrast with Scenario A. The party side of this trip — what the customer will be invoiced — is deliberately **not tracked by this module**; that is the deferred receivables ledger of Section 5.3.

### Scenario E — party billed at an agreed rate different from the booking rate

Scenario A, but the party has agreed ₹1,250/MT rather than the internally recorded ₹1,150/MT.

| Step | Figure |
|---|---|
| Effective billing rate | ₹1,250/MT |
| Expected party freight (18 × ₹1,250) | ₹22,500 |
| Truck gross — **unchanged** | ₹17,000 |
| **Amount to recover from the driver** | **₹5,500** |

**Business rule:** the override moves the **party side only**. The truck's own entitlement is calculated from the hire rate and is never affected. The additional ₹100/MT is additional margin, and it flows through as additional recovery.

### Scenario F — fixed trip rates rather than per-MT

Hire ₹20,000 fixed, booking ₹24,000 fixed, loaded 18 MT, delivered 17 MT.

| Step | Figure |
|---|---|
| Truck freight | ₹20,000 |
| Shortage | 1 MT |
| **Shortage deduction** | **₹0 — not applied** |
| Expected party freight | ₹24,000 |
| **Amount to recover from the driver** | **₹5,000** |

**Business rule:** on a fixed-rate trip there is no per-MT basis, so weight variance has no monetary effect on either side. Shortage is still *recorded* but deducts nothing. Hire and booking rates carry **independent** fixed/per-MT flags — a trip may legitimately be fixed on one side and per-MT on the other.

---

## 10. Business rules catalogue

Numbered for traceability. Each is independently testable.

### Rates and weights
- **BR-01** Hire rate and booking rate are independent, and each may be per-MT or a fixed trip amount.
- **BR-02** Truck freight = hire rate if fixed, else loaded weight × hire rate.
- **BR-03** The loading challan's weight is a booking-time estimate only, never the final figure. When a bilty exists its goods-line weight total is the final load weight; otherwise the challan weight is the starting point and is editable on the settlement screen.
- **BR-04** Where goods lines exist, bilty freight recalculates as total goods weight × the effective billing rate whenever either changes, and remains manually overridable. An override does not survive a subsequent weight or rate change.
- **BR-05** The bilty-rate override changes only the party-side figure. The truck side is never affected.

### Shortage
- **BR-06** Shortage = max(0, loaded weight − delivered weight). Delivered weight defaults to loaded weight until known.
- **BR-07** Shortage deduction = (shortage × hire rate) + (max(0, shortage − tolerance) × cutting rate).
- **BR-08** On a fixed hire rate, the shortage deduction is zero.
- **BR-09** When the **consignee** settles the freight and a shortage exists, the shortage deduction is **not** applied against the truck.

### Charges
- **BR-10** Every charge carries a direction: *recoverable from* the truck (reduces what the truck gets) or *payable to* the truck (increases it).
- **BR-11** Commission, Shortage and Halting Charges have **locked** directions — Recoverable, Recoverable and Payable respectively — and cannot be flipped. All other charge types default to a direction but may be flipped per instance, so new kinds of charge need no code change.
- **BR-12** Commission may be a fixed amount or a percentage.
- **BR-13** When the consignee settles the freight, halting charges are added to the party's expected freight.

### TDS, diesel, advances
- **BR-14** TDS is a flat amount or a percentage of truck freight, recomputed live so a later correction to weight or hire rate flows through rather than freezing a stale figure. It always acts as a recoverable deduction.
- **BR-15** Diesel issued to the driver is recorded separately (amount plus pump name) but affects the settlement exactly as an advance does.
- **BR-16** Advances total = all advance entries plus diesel.
- **BR-17** On a **Paid / To-Be-Billed** trip, advances reduce the final amount payable to the truck.
- **BR-18** On a **To-Pay** trip, advances do **not** reduce the recovery figure; they appear only in a separate reference total. *(Owner-confirmed. Intentional.)*
- **BR-19** Each advance records both the staff member who physically handed it over and **which party funded it** (the business itself, consignor, consignee, third party, broker or other), so a trip funded from several sources stays traceable.
- **BR-20** An advance logged on a bilty is carried onto its freight challan automatically, both at creation and on subsequent edit, tagged as bilty-sourced so re-syncing never disturbs an advance entered directly on the freight challan.

### Settlement direction
- **BR-21** To-Pay: truck gross = truck freight + charge adjustments − shortage deduction (unless BR-09) − TDS.
- **BR-22** To-Pay net position = truck gross − freight actually collected by the driver. Negative means recover from the driver; positive means the business owes the driver.
- **BR-23** When the net position is positive, the trip's ledger direction **reverses** to payments (business → driver) and settlement tracks that balance to zero.
- **BR-24** Paid / To-Be-Billed: net payable to the truck = truck freight + adjustments − shortage − TDS; final payable = net payable − advances; balance = final payable − payments made.
- **BR-25** Paid and To-Be-Billed are one settlement branch. The truck is paid by the business either way.

### Party-side expectation
- **BR-26** Expected party freight is: the effective billing rate if fixed; else, when the consignee settles, loaded weight × rate plus any halting; else min(loaded, delivered) × rate.
- **BR-27** Before POD, expected party freight is assumed collected in full and all figures are a flagged **estimate**.
- **BR-28** After POD, the **confirmed collected amount** drives the settlement; the expected figure is retained and displayed alongside, with the shortfall shown as still due from the party.

### POD
- **BR-29** A POD requires at least one photograph and a POD date.
- **BR-30** For a To-Pay trip, the POD **must** record whether and how much the driver collected before it can be saved.
- **BR-31** Both answers — Yes and No — require an amount, and whatever amount is entered is what drives the settlement. *(An earlier implementation silently assumed full collection on "Yes"; this was corrected.)*
- **BR-32** Saving a POD sets the bilty's trip-completed flag and the freight challan's POD-received flag, copies the photographs across, and carries the received weight into the unload weight **only if that field is still blank**.
- **BR-33** A POD recorded before its freight challan exists must be found and carried forward at freight-challan creation, and retroactively linked.
- **BR-34** Exactly one POD record per delivery, regardless of whether it was raised from the bilty or the freight challan.
- **BR-35** Until POD is received, money entries and settlement closure are blocked.

### Documents and disclosure
- **BR-36** Freight amount is always shown on a To-Pay bilty (the party must know what to pay) and is a per-bilty opt-in otherwise, defaulting to hidden.
- **BR-37** Hire rate is never shown unless explicitly switched on per bilty. Default off.
- **BR-38** When the booking rate is disclosed and a bilty-rate override is active, the document shows the **rate actually being billed**, as a single row — never both the internal booking rate and the billed rate. *(Corrected defect; the internal rate must not leak.)*
- **BR-39** Every WhatsApp share presents a per-field checklist. Individually sensitive line items — charges, advances, ledger entries — start **unticked** so nothing is disclosed by accident.
- **BR-40** Where no valid mobile number is on file, the composed message is copied to the clipboard rather than discarded.
- **BR-41** The settlement statement is composed in account-statement order: freight → charges cut or added → gross → final figure → advances → each dated receipt or payment → balance → POD status.

### Master data
- **BR-42** A loading challan may be saved with a free-text truck number and a free-text party name, with no master record. Speed at booking time outweighs referential purity.
- **BR-43** A bilty **requires** master-linked consignor, consignee, truck and driver, plus a freight amount. This is the point at which data is made clean.
- **BR-44** Any reference field offers a searchable autocomplete with an "add as new" option that opens the relevant master form pre-filled, and returns to the in-progress form with the new record selected and nothing else lost. Backing out without saving also restores the in-progress form.
- **BR-45** A party carries zero or more role tags, and the same party may act in different roles on different trips. The operative role is decided by the field it is used in, not by the record.
- **BR-46** Names are normalised to Title Case and truck numbers to upper case on save. **Capitalisation only — spelling is never altered.**
- **BR-47** Locations and "referred by" names typed into any form are silently remembered and offered as suggestions thereafter, with no prompt.
- **BR-48** A truck requires only a registration number. A driver requires a name and a licence number. A party requires a name and a contact number.

### Multi-account
- **BR-49** Multiple fully separate businesses may exist in one installation, each with its own masters, transactions, numbering and company profile. **Data never merges across accounts.** Switching saves the current account's state before loading the target's.

### General
- **BR-50** One freight challan per loading challan. Both creation paths converge on the same record.
- **BR-51** A cancelled loading challan retains its cancellation reason and timestamp, and editing its other details never silently clears that state.
- **BR-52** No browser-native alert, confirm or prompt dialogs anywhere. Every confirmation or reason capture uses an in-app form. *(Standing owner instruction.)*
- **BR-53** A focused numeric field must never change value on an accidental scroll.

---

## 11. Reporting requirements

Four reports exist today, each computed live from transaction data rather than stored:

**Commission report** — commission per freight challan plus a total, across all trips.
**Pending recoveries** — every unsettled To-Pay trip with its outstanding recovery, plus a total. This is "money the drivers owe us."
**Pending payables** — every unsettled Paid / To-Be-Billed trip with its outstanding balance, plus a total. This is "money we owe the trucks."
**Payment summary** — total out, total in, net, and a breakdown by category, from standalone payment entries.

Each row links through to the underlying trip. Together, pending recoveries and pending payables satisfy objective O3.

Deferred: trip register, truck-wise and driver-wise revenue, monthly revenue versus expense, party-wise outstanding (which depends on the deferred receivables ledger).

---

## 12. Priority of requirements

| Priority | Content |
|---|---|
| **Must have** | All masters; loading challan; bilty with branded output and disclosure controls; freight challan with the complete settlement engine in both directions; POD with its gating; the payment/recovery ledger; WhatsApp sharing; company profile; the four reports; offline-capable entry. |
| **Should have** | Multi-account support; location and referred-by memory; standalone payment entries; selective-disclosure sharing on master records. |
| **Could have** | PDF export refinements; truck-list export. |
| **Won't have (this phase)** | Expense & fuel; enforced permissions; change-audit log; party-side receivables ledger; driver or customer access of any kind. |

---

## 13. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Settlement direction implemented in one direction only | Severe — silent revenue loss, and the system shows trips as settled that are not | Implement all nine scenarios of Section 9 as automated tests before any UI work |
| Offline-first treated as a later enhancement | Severe — architectural rewrite | Decide the data layer before development starts (Section 6) |
| Master-data validation imposed at booking time | High — users abandon the system and revert to paper | Honour BR-42 strictly: free text at booking, cleaned at bilty |
| Internal hire rate leaking onto party documents | High — commercial damage | Default every disclosure toggle to off; implement BR-38 exactly |
| Advance carried from bilty to freight challan silently dropped | High — double payment | BR-20 plus a regression test |
| Terminology misread by implementers | Moderate — plausible but wrong behaviour | Section 7 glossary is mandatory reading |

---

## 14. Phasing

**Phase 1 — core paperwork and the truck account.** All masters, company profile, multi-account, loading challan, bilty with branded PDF and sharing, the freight challan settlement engine, POD, the payment/recovery ledger, the four reports, user records, offline-capable entry, both platforms. This phase alone replaces the bilty book, the master registers and the truck-account notebook.

**Phase 2 — money and control.** Expense & fuel management; the party-side receivables ledger with ageing and statements; the admin/staff permission matrix enforced; the change-audit log; the remaining reports.

**Phase 3 — polish.** Advanced reporting, backup and export, and GST fields if and only if the business registers.

---

## 15. Acceptance

The system is accepted when:

1. All nine worked scenarios in Section 9 reproduce exactly.
2. Every business rule in Section 10 is demonstrable.
3. Both lifecycle paths in Section 8 complete end to end.
4. A loading challan can be raised on a phone, from a standing start, in under a minute, with only a truck number and a party name.
5. No party-facing document displays the hire rate unless explicitly switched on for that document.
6. Records can be created offline and sync on reconnection.
7. Pending recoveries and pending payables are reachable in two taps and tie to the underlying trips.

---

*Prepared as a handoff document for integration into a parent application. The authoritative build detail — every field, type, validation, screen and formula — is in `LogiTrack_Functional_Spec.md`. The working reference implementation is `logitrack_prototype.html`.*
