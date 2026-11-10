# LogiTrack — Functional & Technical Specification

**Document type:** Functional Requirements Specification / Build Specification
**Product:** LogiTrack — mobile operations app for an Indian road-freight transport business
**Version:** 1.0 (Handoff edition — as-built, technology-agnostic)
**Date:** 10 November 2026
**Companion documents:** `LogiTrack_BRD.md` (business requirements, glossary, worked scenarios), `logitrack_prototype.html` (working reference implementation), `LogiTrack_Handoff_Brief.md`

---

## 1. How to use this document

### 1.1 What this is

This is a complete, self-contained build specification for LogiTrack. It documents a **working system** that was designed and refined over 23 rounds of iteration with the business owner, and that exists today as a single-file HTML/CSS/JavaScript prototype (`logitrack_prototype.html`, ~4,530 lines) covering every screen and every calculation described here.

**Sections 2 through 12 describe the system exactly as it exists.** They are the as-built specification and should be treated as authoritative. **Section 13 is clearly separated and contains recommended improvements** — known gaps, deferred features and upgrades for a production build. Nothing in Section 13 is part of the current system.

### 1.2 Required reading order

1. **`LogiTrack_BRD.md` Section 7 (glossary) — first, and non-optionally.** This is Indian road-freight trade vocabulary. A developer who does not know what a bilty, a freight challan, munshiyana or a To-Pay booking is cannot build this correctly, and the failure mode is plausible-looking software that computes the wrong numbers.
2. **`LogiTrack_BRD.md` Sections 8 and 9** — the lifecycle and the nine worked numeric scenarios.
3. This document, Sections 3, 4 and 7 — reference data, data model, calculation engine.
4. The rest of this document as needed per module.

### 1.3 This specification is technology-agnostic

It states **what** to build, not **how**. No framework, database, or language is prescribed. The reference prototype's single-file, in-memory implementation is a *design artifact*, not an architectural recommendation — it has no persistence at all. Section 11 states the non-functional requirements (offline-first, platform, etc.) that will constrain the receiving team's stack choice.

Where this document writes field names in `camelCase`, those are the reference implementation's identifiers. They are given so that behaviour described here can be traced into the prototype source. **Rename freely** to suit the parent application's conventions; the names carry no requirement.

### 1.4 Verification

The nine worked scenarios in BRD Section 9 were produced by executing the reference implementation's actual settlement function. **They are the acceptance tests for the calculation engine.** Build them as automated tests first, before any UI work. Section 12 lists the full acceptance set.

---

## 2. System overview

### 2.1 Purpose

A mobile application that digitises document and money management for a freight brokerage: master records, the booking-to-delivery document chain, and per-trip settlement with the truck.

### 2.2 Users

Exactly one organisation, 1–3 users (one owner/admin, one to two office staff). No external party, driver or customer has access of any kind. Every outbound communication is composed in-app and sent by a staff member via WhatsApp. See BRD Section 3.

### 2.3 Functional architecture

The system comprises three layers of records:

**Master data** (slowly changing reference records): Truck, Driver, Party, Product, Location, User, Charge Type, plus a singleton Company Profile.

**Transactions** (the document chain): Loading Challan → Bilty → Freight Challan → POD Record, plus standalone Payment Entries.

**Derived views** (computed, never stored): the settlement calculation, the four reports, all list aggregates and badges.

A hard architectural rule runs through the whole design: **money figures are computed on read, never stored.** The settlement function is pure — it takes a freight challan (and resolves its linked bilty) and returns a complete result object. Nothing in the calculation is persisted, so correcting an input weeks later automatically corrects every derived figure, report and balance. Any replacement implementation must preserve this property. Storing computed totals in the database would reintroduce exactly the class of stale-figure bug the design avoids.

### 2.4 Multi-account containment

The entire data set above (all masters, all transactions, the numbering counters, and the company profile) is scoped to an **account**. An installation may hold several accounts representing entirely separate businesses. Data never merges or is visible across accounts. Switching accounts persists the current account's complete state before loading the target's.

In the reference implementation this is done by swapping the working collections; in a database-backed build, the natural equivalent is an `account_id` discriminator on every table with mandatory filtering, or separate schemas. The requirement is isolation, not a particular mechanism.

---

## 3. Reference data and enumerations

These lists are referenced throughout. Several of them are the owner's own wording and ordering and should be reproduced exactly.

### 3.1 `TRUCK_TYPES` — 24 values, in this order

```
Pickup, Mini Truck, LCV, 6 Wheeler, 10 Wheeler, 12 Wheeler, 14 Wheeler,
16 Wheeler, 18 Wheeler, 22 Wheeler, 24 Wheeler, 32 Feet, 34 Feet, 40 Feet,
45 Feet, Trailer, Container, Tanker, Bulker, Tipper, Reefer, Car Carrier,
Flatbed, Other
```

Note that this list deliberately mixes three different axes — wheel-count classes, length classes and body-type classes — because in practice a truck is described by whichever is most relevant to it. **Do not "normalise" this into separate fields.** It is the owner's own vocabulary; `22 Wheeler` is the business's workhorse.

### 3.2 `PARTY_TYPES` — role tags on a party

```
Consignor, Consignee, Supplier, Broker, Third Party, Transporter, Other
```

A party carries **zero or more** of these as tags. They are descriptive only. **No field anywhere in the application filters a party picker by type** — every party-referencing field searches the full party list, and the operative role is determined by which field the party was selected into. This was an explicit design reversal: role used to be a required single value, then became a multi-tag, then the tag editor was removed from the party form entirely. Tags survive only via quick-add pre-tagging (Section 6.3.3).

### 3.3 `FREIGHT_PAYER_ROLES` — who actually settles the freight

```
Consignor, Consignee, Third Party, Transporter, Broker, Other
```

Used on the Bilty, the Start Freight Challan screen, and the freight challan settlement screen. Selecting `Other` reveals a short free-text label field. **`Consignee` carries special calculation behaviour** — see Section 7.

### 3.4 `PAYMENT_MODES` — one shared list, used everywhere a payment mode is asked

```
Cash, NEFT, RTGS, IMPS, Cheque, UPI, Other
```

Applies to: bilty advances, freight-challan advances, freight-challan ledger entries (payment and recovery), the mark-fully-settled form, and standalone payment entries. This single shared list replaced several divergent per-screen lists; **it must not fragment again.**

### 3.5 `ADVANCE_SOURCES` — which party funded a payment

```
Self (Company), Consignor, Consignee, Third Party, Broker, Other
```

Captured on every advance and every ledger entry, answering "whose money was this?" — distinct from "who physically handed it over". Selecting `Other` reveals a free-text label.

### 3.6 `PAYMENT_TYPES`

```
To-Pay, Paid, To-Be-Billed
```

`Paid` and `To-Be-Billed` are **one settlement branch** (the business pays the truck either way). `To-Pay` is the other. See Section 7.

### 3.7 Charge Type Master — a seeded, backend-only list

No list, add or edit screen exists for this, by explicit owner instruction. It surfaces only as the Type picker inside the freight challan's Add Charge form.

| Charge type | Default direction | Locked? |
|---|---|---|
| Commission | Recoverable | **Yes** |
| Shortage | Recoverable | **Yes** |
| Halting Charges | Payable | **Yes** |
| Munshiyana | Recoverable | No |
| Other Expenses | Payable | No |
| Kanta Charges (Weighing) | Payable | No |
| Dala (Loading/Unloading Labour) | Payable | No |
| Loading Charges | Payable | No |
| Unloading Charges | Payable | No |
| Service Charge | Payable | No |
| Brokerage | Recoverable | No |
| Other | Payable | No |

`Recoverable` means *recoverable from the truck* — it reduces what the truck receives. `Payable` means *payable to the truck* — it increases it. The three locked rows are firm business rules; every other row's direction is a default the user may flip per instance, so new kinds of charge need no code change.

### 3.8 Other small enumerations

| List | Values |
|---|---|
| Rate type (hire, booking, bilty, bill) | `PerMT`, `Fixed` — each rate carries its own flag independently |
| Record status (masters) | `Active`, `Inactive` |
| Location type | `Loading`, `Unloading` |
| User role | `Admin`, `Staff` (records only; not enforced) |
| Payment entry direction | `Payment Out`, `Receipt In` |
| Payment entry category | `Freight`, `Advance`, `Party Balance`, `Office Expense`, `Other` |
| Bilty expense types | The **same Charge Type Master list** as Section 3.7, reused as the Type picker. The expenses themselves are display-only and separate from freight-challan charges |
| Advance paid-to | `Driver`, `Owner` — **two options only** |
| Product default UOM | `MT`, `KG`, `Box`, `Bag`, `Piece` |
| Goods line unit (bilty) | A free-text field with a 22-value suggestion datalist: `Bags`, `Bales`, `Barrels`, `Boxes`, `Bundles`, `Cans`, `Cartons`, `Coils`, `Crates`, `Drums`, `KG`, `Litres`, `MT`, `Nos`, `Packets`, `Pallets`, `Pieces`, `Quintal`, `Rolls`, `Sacks`, `Sheets`, `Tonnes`. **Any value may be typed** — the list only suggests |
| TDS mode | `Fixed`, `Percentage` |
| Commission mode | `Fixed`, `Percentage` |
| Yes/No toggles | `Yes`, `No` — rendered as two-option segmented toggles, never dropdowns |

---

## 4. Data model

Types below are logical. `money` is a decimal amount in INR (stored as a string in the reference implementation; **use a decimal numeric type in production — never a float**). `weight` is decimal MT. `date` is a calendar date with no time component. Arrays marked *(embedded)* are sub-records owned by their parent with no independent identity.

### 4.1 Truck

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | id | yes | |
| `number` | text | **yes** | Registration number. **Normalised to UPPER CASE on save.** The only required field. |
| `type` | enum | no | From `TRUCK_TYPES`. Defaults to the first value. |
| `capacity` | weight | no | MT |
| `status` | enum | no | `Active` / `Inactive`, default `Active` |
| `photos` | text[] | no | Multiple. Truck photos and documents: RC, insurance, permit, fitness, pollution, TDS declaration |
| `ownerName` | text | no | **Title Case on save.** Free text — see Section 13.1 |
| `ownerMobile` | text | no | |
| `ownerPan` | text | no | |
| `ownerAadhar` | text | no | |
| `ownerAddress` | text | no | |
| `referredBy` | text | no | **Title Case on save.** Fed to and from the shared referred-by suggestion list |

### 4.2 Driver

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | id | yes | |
| `name` | text | **yes** | Title Case on save |
| `contact` | text | no | |
| `license` | text | **yes** | Driving licence number |
| `status` | enum | no | default `Active` |
| `photos` | text[] | no | Driver photos, DL and Aadhaar documents |
| `referredBy` | text | no | Title Case on save |

### 4.3 Party

One master covering every external counterparty role.

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | id | yes | |
| `name` | text | **yes** | Title Case on save |
| `contact` | text | **yes** | |
| `types` | enum[] | no | Zero or more `PARTY_TYPES`. Descriptive only; never filters a picker |
| `city` | text | no | Title Case on save |
| `address` | text | no | |
| `gstin` | text | no | |
| `pan` | text | no | |
| `referredBy` | text | no | Title Case on save |
| `location` | text | no | Google Maps URL. **A pasted "share location" blob is parsed and only the URL retained** |
| `acc` | text | no | Bank account number |
| `ifsc` | text | no | |
| `accName` | text | no | Account holder name |
| `status` | enum | no | default `Active` |

### 4.4 Product

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | id | yes | |
| `name` | text | yes | The material carried |
| `category` | text | no | e.g. `General`, `Fragile` |
| `uom` | text | no | Default unit of measure: `MT`, `KG`, `Box`, `Bag`, `Piece` |
| `status` | enum | no | default `Active` |

### 4.5 Location

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | id | yes | |
| `name` | text | yes | |
| `city` | text | no | |
| `state` | text | no | |
| `address` | text | no | |
| `type` | enum | no | `Loading` / `Unloading` |
| `status` | enum | no | default `Active` |

Locations are also **created silently**: any loading or unloading point typed as free text on a loading challan is remembered and offered as a suggestion thereafter, with no prompt and no confirmation.

### 4.6 User

Record-keeping only. No authentication exists in the current system.

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | id | yes | |
| `name` | text | yes | |
| `contact` | text | no | |
| `role` | enum | no | `Admin` / `Staff` |
| `branch` | text | no | |
| `status` | enum | no | default `Active` |

### 4.7 Company Profile (singleton per account)

| Field | Type | Notes |
|---|---|---|
| `companyName` | text | Shown in the persistent header bar and on every bilty letterhead |
| `ownerName` | text | |
| `adminName` | text | |
| `contact` | text | On the bilty letterhead |
| `address` | text | On the bilty letterhead |
| `gstin` | text | On the letterhead if present |
| `pan` | text | On the letterhead if present |
| `logo` | text | Letterhead logo |

### 4.8 Referred-By suggestion list

A flat list of names previously typed into any "Referred By" field, shared between Truck Master and Party Master and used only to power suggestions. Silently maintained; no screen.

### 4.9 Loading Challan

The booking record. Created first, always.

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | id | yes | |
| `serialNo` | text | yes | Auto-generated `LC-####`, manually editable |
| `date` | date | yes | |
| `truckId` | ref → Truck | **one of** | |
| `truckNumber` | text | **one of** | **Free text accepted when no master link exists.** Upper-cased on save. At least one of `truckId` / `truckNumber` required |
| `driverMobile` | text | no | Driver's number for this trip (no driver master link at booking time) |
| `ownerMobile` | text | no | |
| `ownerName` | text | no | Title Case on save |
| `loadingPoint` | text | **yes** | Autocomplete against Location Master; free text accepted |
| `unloadingPoint` | text | **yes** | Same |
| `hireRate` | money | **yes** | **What the truck is paid. Internal.** |
| `hireRateType` | enum | no | `PerMT` / `Fixed`, default `PerMT` |
| `bookingRate` | money | **yes** | What the party pays |
| `bookingRateType` | enum | no | `PerMT` / `Fixed`, default `PerMT` |
| `partyId` | ref → Party | **one of** | |
| `partyName` | text | **one of** | Free text accepted. Title Case on save. At least one of `partyId` / `partyName` required |
| `partyNumber` | text | no | Used as the WhatsApp target for the loading slip |
| `weight` | weight | no | **Booking-time estimate only — never the final weight** |
| `biltyNeeded` | enum | **yes** | `Yes` / `No`. **Determines which lifecycle path is taken.** No default — must be answered |
| `notes` | text[] | no | Repeatable free-form notes. Shared to the party in WhatsApp bold markup |
| `supplierId` | ref → Party | no | Who supplied the truck |
| `supplierName` | text | no | Free text accepted. Title Case on save |
| `loadBrokerId` | ref → Party | no | |
| `loadBrokerName` | text | no | Free text accepted. Title Case on save |
| `linkedBiltyId` | ref → Bilty | no | Set when a bilty is created from this challan |
| `linkedFreightChallanId` | ref → FreightChallan | no | **Enforces one freight challan per loading challan** |
| `cancelled` | bool | no | |
| `cancelReason` | text | no | Required when cancelling |
| `cancelledAt` | timestamp | no | |

**Critical:** editing a challan must never clear `linkedBiltyId`, `linkedFreightChallanId`, `cancelled`, `cancelReason` or `cancelledAt`. Both were real defects in the reference implementation.

### 4.10 Bilty / GR

The party-facing consignment note.

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | id | yes | |
| `biltyNo` | text | yes | Auto-generated `LT-####`, manually editable |
| `date` | date | yes | |
| `consignorId` | ref → Party | **yes** | Must resolve to a master record |
| `consigneeId` | ref → Party | **yes** | Must resolve to a master record |
| `thirdPartyId` | ref → Party | no | Optional — broker, referrer, etc. |
| `truckId` | ref → Truck | **yes** | Must resolve to a master record |
| `driverId` | ref → Driver | **yes** | Must resolve to a master record |
| `goods` | *(embedded)* | yes | Repeatable lines: `{ productId, qty, unit, weight }`. Lines without a product are discarded on save |
| `hireRate` / `hireRateType` | money / enum | no | Carried from the challan. **Internal** |
| `bookingRate` / `bookingRateType` | money / enum | no | Carried from the challan |
| `billAtBookingRate` | enum | no | `Yes` / `No`, default `Yes`. `No` activates the billing-rate override |
| `biltyRate` | money | no | The agreed rate actually billed. Only retained when `billAtBookingRate = No` |
| `biltyRateType` | enum | no | `PerMT` / `Fixed` |
| `freight` | money | **yes** | What the party is billed. Auto-computed from goods weight × effective rate; manually overridable |
| `advances` | *(embedded)* | no | See 4.12 |
| `charges` | *(embedded)* | no | Bilty expenses, display-only: `{ type, amount }`. **Distinct from freight-challan charges** |
| `paymentType` | enum | yes | `To-Pay` / `Paid` / `To-Be-Billed` |
| `freightPaidByRole` | enum | yes | From `FREIGHT_PAYER_ROLES`, default `Consignor` |
| `freightPaidByOtherLabel` | text | no | Only retained when role = `Other` |
| `showFreight` | enum | no | Disclosure toggle. **Forced to `Yes` when `paymentType = To-Pay`**, else default `No` |
| `showHireRate` | enum | no | Disclosure toggle, default `No`. **The internal buy rate** |
| `showBookingRate` | enum | no | Disclosure toggle, default `No` |
| `showAdvance` | enum | no | Disclosure toggle, default `No` |
| `showExpenses` | enum | no | **Forced to `Yes` when `To-Pay`**, else default `No` |
| `tripCompleted` | bool | no | Set by the POD record, not directly |
| `remarks` | text | no | Pre-filled with the route from the source challan |
| `sourceChallanId` | ref → LoadingChallan | no | **Presence gates the "Create Freight Challan" action (Path 2)** |
| `ewayBillDoc` | text | no | |
| `invoiceDoc` | text | no | |
| `otherDocs` | text[] | no | Repeatable |
| `advance` | money | no | **Legacy. Always `'0'`. Superseded by `advances[]`. Do not carry forward** |

### 4.11 Freight Challan

The truck's settlement account. **Internal only — never shown to a party.** Always derived from a loading challan; never created standalone.

| Field | Type | Notes |
|---|---|---|
| `id` | id | |
| `serialNo` | text | Auto-generated `FC-####` |
| `date` | date | Creation date |
| `loadingChallanId` | ref → LoadingChallan | **Always present** |
| `biltyId` | ref → Bilty | Empty on Path 1. **Its presence switches several fields to read-only and changes where the billing override is read from** |
| `truckId` / `truckNumber` | ref / text | Carried from the challan; the free-text number is carried when there is no master link |
| `driverMobile`, `ownerMobile`, `ownerName` | text | Carried from the challan. **`ownerMobile` is the primary WhatsApp target for the settlement statement, falling back to `driverMobile`** |
| `hireRate` / `hireRateType` | money / enum | Carried; editable on the settlement screen |
| `bookingRate` / `bookingRateType` | money / enum | Carried |
| `loadWeight` | weight | Final loaded weight. From the bilty's goods total when a bilty exists and is non-zero, else the challan weight |
| `unloadWeight` | weight | Delivered weight. Blank until known; filled by the POD **only if still blank** |
| `shortageTolerance` | weight | Free allowance in MT, default `0` |
| `cuttingRate` | money | Penalty ₹/MT beyond tolerance, default `0` |
| `charges` | *(embedded)* | See 4.12 |
| `advances` | *(embedded)* | See 4.12 |
| `tdsMode` | enum | `Fixed` / `Percentage`, default `Fixed` |
| `tdsAmount` | money | Used when `Fixed` |
| `tdsPct` | decimal | Used when `Percentage` — **of truck freight, recomputed live** |
| `dieselAmount` | money | Diesel issued to the driver |
| `dieselPump` | text | |
| `supplierId` / `supplierName` | ref / text | Defaults from the challan; editable |
| `brokerId` / `brokerName` | ref / text | **Resolution order: the bilty's `thirdPartyId` first, else the challan's `loadBrokerId`.** Editable |
| `consignorId` / `consignorName` | ref / text | From the bilty when one exists (**read-only**); else pre-filled from the challan's party (editable) |
| `consigneeId` / `consigneeName` | ref / text | Same |
| `paymentType` | enum | From the bilty, or from the Start screen on Path 1 |
| `freightPaidByRole` | enum | Same. **Read-only when a bilty exists** |
| `freightPaidByOtherLabel` | text | |
| `billAtBookingRate` | enum | **Path 1 only.** On Path 2 the bilty's own value is used |
| `billRate` / `billRateType` | money / enum | **Path 1 only** |
| `podReceived` | bool | Set by the POD record. **Gates money entry and settlement closure** |
| `podDocs` | text[] | Copied from the POD record |
| `podNote` | text | |
| `podDamage` | enum | `Yes` / `No` |
| `podDamageNotes` | text | Only retained when damage = `Yes` |
| `toPayFreightReceived` | enum | `''` / `Yes` / `No` — confirmed at POD. **`''` means "not yet confirmed" and the settlement is an estimate** |
| `toPayAmountReceived` | money | The amount actually collected. **Drives the settlement once confirmed** |
| `payments` | *(embedded)* | Business → truck. See 4.12 |
| `recoveries` | *(embedded)* | Truck → business. See 4.12 |
| `commissionPaid` | enum | `''` / `Yes` / `No`. **Hidden entirely when a Commission charge exists** (it is then already netted) |
| `commissionPaidMode` | enum | Only when `Yes` |
| `commissionPaidDate` | date | Only when `Yes` |
| `balancePayableAt` | text | Where the balance is to be paid |
| `remark` | text | |
| `signatureCollected` | bool | |
| `financialYear` | text | Computed as `YYYY-YY` on an April–March year. Stored but never displayed |
| `commStatus` | text | **Legacy. Superseded by `commissionPaid`. Do not carry forward** |

### 4.12 Embedded sub-records

**Charge** (freight challan)

| Field | Notes |
|---|---|
| `type` | From the Charge Type Master |
| `direction` | `Recoverable` / `Payable`. Pre-filled from the master; **locked for Commission, Shortage and Halting Charges** |
| `amount` | For a percentage commission, the **computed** amount is stored |
| `commissionMode` | `Fixed` / `Percentage` — Commission only |
| `commissionPct` | Percentage only |
| `note` | Optional |

**Advance** (freight challan)

| Field | Notes |
|---|---|
| `mode` | From `PAYMENT_MODES` |
| `amount` | |
| `date` | |
| `givenBy` | The **staff member** who physically handed it over |
| `paidTo` | `Driver` / `Owner` — two options only |
| `paidBySource` | From `ADVANCE_SOURCES` — **which party funded it** |
| `paidBySourceOtherLabel` | Only when source = `Other` |
| `note` | |
| `fromBilty` | **Flag.** `true` marks this as a live mirror of a bilty advance. Such an entry **cannot be deleted here** — the user is directed to the bilty. See Section 8.2 |

**Advance** (bilty) — a reduced shape: `mode`, `amount`, `date`, `paidBySource`, `paidBySourceOtherLabel`, `note`.

**Bilty charge (expense)** — `type` (from the bilty expense list), `amount`. Display-only; **never enters the truck-side settlement**.

**Ledger entry** (`payments[]` / `recoveries[]`)

| Field | Notes |
|---|---|
| `date` | Defaults to today for a normal entry; **explicitly chosen** on the mark-fully-settled form |
| `amount` | Must be > 0 |
| `mode` | From `PAYMENT_MODES` |
| `refNo` | UTR / UPI reference / cheque number |
| `note` | On a mark-fully-settled entry, defaults to `"Marked fully settled"` |
| `source` | From `ADVANCE_SOURCES` — "Paid By" |
| `sourceOtherLabel` | Only when source = `Other` |
| `paidParty` | **Mark-fully-settled entries only** — who the money went to / came from |

### 4.13 POD Record

One record per delivery, cross-linked to whichever documents exist.

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | id | yes | |
| `biltyId` | ref → Bilty | either | |
| `freightChallanId` | ref → FreightChallan | either | **Either may be empty, but lookup matches on either, so one record serves both entry points** |
| `photos` | text[] | **yes — min 1** | Cannot be saved empty |
| `podDate` | date | **yes** | |
| `receivedWeight` | weight | no | Pre-filled from the freight challan's load weight, else the bilty's goods total |
| `damage` | enum | no | `Yes` / `No`, default `No` |
| `damageNotes` | text | no | Only retained when damage = `Yes` |
| `note` | text | no | |
| `toPayFreightReceived` | enum | **conditional** | **Mandatory for a To-Pay trip.** `Yes` / `No` |
| `toPayAmountReceived` | money | conditional | **Required for either answer.** Defaults to `'0'` |
| `status` | enum | yes | `Received` / `Not Received` — revertible and re-confirmable |
| `receivedAt` | timestamp | yes | Set on save |

### 4.14 Payment Entry (standalone ledger)

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | id | yes | |
| `direction` | enum | yes | `Payment Out` / `Receipt In`, default `Payment Out` |
| `partyId` / `partyName` | ref / text | no | Autocomplete against Party Master; free text accepted |
| `category` | enum | no | `Freight` / `Advance` / `Party Balance` / `Office Expense` / `Other` |
| `amount` | money | **yes** | Must be > 0 |
| `mode` | enum | no | From `PAYMENT_MODES` |
| `refNo` | text | no | |
| `date` | date | no | Defaults to today |
| `note` | text | no | |

**Scope note:** this module does **not** duplicate freight-challan ledger entries. Those remain owned by their trip. This module holds only entries not tied to any trip. *(The reference implementation's list shows only standalone entries; see Section 13.6.)*

### 4.15 Entity relationships

```
Account ──┬─< Truck ──< (ownerName: free text — NOT linked to Driver; see 13.1)
          ├─< Driver
          ├─< Party ───────────────┐
          ├─< Product              │
          ├─< Location             │
          ├─< User                 │
          ├─  CompanyProfile (1:1) │
          │                        │
          └─< LoadingChallan ──────┤  party, supplier, loadBroker
                 │  truckId        │
                 │                 │
                 ├──0..1── Bilty ──┤  consignor, consignee, thirdParty
                 │           │      │  truckId, driverId, goods→Product
                 │           │      │
                 └──0..1── FreightChallan  (exactly one per LoadingChallan)
                               │     consignor, consignee, supplier, broker
                               │
                        0..1 PodRecord  (links to Bilty and/or FreightChallan)

PaymentEntry ──0..1──> Party     (independent of any trip)
```

**Cardinality rules.** One loading challan yields at most one bilty and at most one freight challan. Both creation paths converge on the same freight challan — a duplicate must be impossible. One POD record per delivery, found by matching **either** linked id.

### 4.16 Identity and numbering

Per-account counters produce: `LC-2001+` (loading challan), `LT-1001+` (bilty), `FC-3001+` (freight challan). Serial numbers are auto-generated and **manually editable**. Internal ids are integers in the reference implementation; use whatever the parent application uses, but note that **id comparison is loose (`==`) in several places in the reference source** — a production build must use consistent, strict typing throughout.

---

## 5. Navigation and information architecture

### 5.1 Shell

Every screen shares a fixed chrome:

- **App title bar** — the app name, the screen title, a screen subtitle, a back affordance when the screen is not a tab root, and a `⋮` overflow menu.
- **Company bar** — a persistent strip below the title showing the account's company name and logo on *every* screen. Tapping it opens the Company Profile.
- **Content area** — scrolling.
- **Floating action button (FAB)** — shown only on screens with a single primary create action.
- **Bottom tab bar** — 7 tabs.

### 5.2 Bottom tabs

| Tab | Destination |
|---|---|
| 🏠 Home | Dashboard |
| 🚛 Trucks | Truck list |
| 🧑 Drivers | Driver list |
| 🏢 Parties | Party list |
| 📄 Bilty | Bilty list |
| 🧾 Loading Challan | Loading challan list |
| 🧮 Freight Challan | Freight challan list |

Selecting a tab **resets the navigation stack**; in-screen navigation pushes onto it and the back affordance pops it.

### 5.3 Screens not on a tab (reached from Home's `⋮` or in-flow)

Product Master, Location Master, User Management, POD Records, Payment Management, Reports & Analytics, Company Profile, and the forms/details belonging to each. "Expense & Fuel" appears in Home's overflow menu as a **visible but disabled** item tagged "Phase 2".

### 5.4 Complete screen inventory — 39 screens

| Group | Screens |
|---|---|
| Home | `home` |
| Truck | `truckList`, `truckForm`, `truckDetail` |
| Driver | `driverList`, `driverForm`, `driverDetail` |
| Party | `partyList`, `partyForm`, `partyDetail` |
| Product | `productList`, `productForm` |
| Location | `locationList`, `locationForm` |
| User | `userList`, `userForm` |
| Company | `companyProfile`, `companyProfileForm` |
| Loading Challan | `loadingChallanList`, `loadingChallanForm`, `loadingChallanDetail`, `cancelChallanForm` |
| Bilty | `biltyList`, `biltyForm`, `biltyDetail` |
| Freight Challan | `freightChallanStart`, `freightChallanList`, `freightChallanDetail`, `freightChallanForm` *(= the settlement-details editor)* |
| POD | `podList`, `podForm`, `podDetail` |
| Payment | `paymentList`, `paymentForm` |
| Reports | `reportsList`, `commissionReport`, `pendingRecoveriesReport`, `pendingPayablesReport`, `paymentSummaryReport` |

Product, Location and User have **no detail screen** — the list taps straight into the edit form. All other masters follow List → Detail → Form.

### 5.5 The overflow menu is contextual

Each screen populates `⋮` with only its own relevant actions, never a repeated global menu. Examples: Home carries the non-tab module links; Truck List carries "Export Truck List (PDF)"; Bilty Detail carries "Edit Bilty"; Freight Challan List carries "POD Records".

### 5.6 Home dashboard

Four stat tiles — total trucks, drivers, bilties this month, in progress (bilties where `tripCompleted` is false). Five quick actions — New Bilty, New Loading Challan, Add Truck, Add Driver, Add Party. Then the three most recent bilties, each with a `Booked` / `Delivered` badge.

---

## 6. Module functional specifications

### 6.1 Cross-cutting behaviours

These apply to every module and are specified once.

#### 6.1.1 Master record pattern

List (search, status badges, empty state, FAB to create) → Detail (read-only, document thumbnails, share, edit via `⋮`) → Form (create/edit, validate, save, return).

#### 6.1.2 Text normalisation on save

Names and places are normalised to **Title Case**; truck registration numbers to **UPPER CASE**. Normalisation collapses runs of whitespace and fixes capitalisation **only — spelling is never altered.** This is an explicit owner instruction: whoever types the name decides the spelling.

#### 6.1.3 Searchable autocomplete — the core interaction pattern

Every field referencing another record is a type-ahead, **never a long static dropdown**. Behaviour:

- Typing filters the source list by case-insensitive substring match on the display label.
- At most **8** suggestions are shown.
- A suggestion row may display **more** than what is placed into the field once picked. A driver row shows `name · mobile · DL {licence}` so two drivers sharing a name can be told apart; a truck row shows `number · type`. In both cases the field itself fills with just the plain name or number afterwards.
- When nothing matches and nothing else is offered, a "No match found" row appears.
- Whenever text has been typed, a final **"➕ Add '[typed text]' as new [Kind]"** row is appended.
- Some fields additionally offer a **"Self"** option, which appears only when the field is empty/just focused or when the typed text could itself mean "self" — never while actively searching a real name, so it cannot be picked instead of the genuine top match.
- Suggestions dismiss on blur, with a short delay so a tap registers.

#### 6.1.4 Quick-add with return-to-form

Tapping "➕ Add …" must: snapshot the **entire in-progress form** including repeatable sub-lists; open the relevant master's create form with the typed text pre-filled into the name/number field; and on save, return to the in-progress form with everything restored **and the new record already selected in the field it was added from**. Backing out without saving must also restore the in-progress form rather than losing it.

Three parallel return contexts exist — from the bilty, from the loading challan, and from the freight challan settlement screen — because all three can quick-add. A quick-add may also carry a **pre-tag**: adding a party from the Consignor field tags the new party `Consignor`. This is now the **only** way party type tags are set, since the tag editor was removed from the party form.

#### 6.1.5 Silent memory

Loading/unloading points and "referred by" names typed anywhere are remembered and offered as suggestions thereafter, with **no prompt, no confirmation and no toast**. The user simply finds them suggested next time.

#### 6.1.6 Selective WhatsApp sharing

Share actions never send a fixed payload. Each opens a **checklist of individual fields**; only ticked items are included. Rules:

- Composition is `wa.me/<number>?text=<urlencoded message>` — this **composes** the message; the user still taps send inside WhatsApp.
- Numbers are normalised to Indian WhatsApp format by this exact algorithm: strip every non-digit; if the result is 12 digits beginning `91`, drop that prefix; else if it is 11 digits beginning `0`, drop that leading zero; then, **if the result is not exactly 10 digits, treat the number as invalid**; otherwise return `91` + the 10 digits.
- **When no valid number is on file, the composed message is copied to the clipboard** (with a clipboard fallback for older environments) and the user told to paste it — never discarded, never shown only in a transient toast.
- Sensitive line items (individual charges, advances, ledger entries) **start unticked** so nothing is disclosed by accident.
- If nothing is ticked, the share is refused with a prompt to pick at least one detail.

Share field sets: **Party** — name, contact, address/city, saved maps location, GSTIN, bank details. **Driver** — name, contact, DL number, photos, DL/Aadhaar documents. **Truck** — number, type & capacity, RC/insurance/permit/fitness/pollution, TDS declaration form, photos.

#### 6.1.7 No native dialogs

**No `alert`, `confirm` or `prompt` anywhere.** Every confirmation, reason capture or acknowledgement is an in-app form or screen. Standing owner instruction. Transient feedback uses a toast.

#### 6.1.8 Numeric input protection

A focused numeric field must never change value on an accidental scroll. Implement globally, not per field — a silently corrupted amount anywhere in this application is a money error.

#### 6.1.9 Date grouping on transaction lists

The loading challan, bilty and freight challan lists group records into **date sections, newest date first**, so it is immediately clear which trucks moved on which day. Within a day, records keep their display order (most recently added first).

---

### 6.2 Module A — Truck Master

**Purpose.** The fleet/supplier truck register with owner details and documents.

**List.** Search by number. Each row: number, `type · capacity MT`, status badge. FAB creates. `⋮` offers "Export Truck List (PDF)".

**Form.** Truck Number *(required, upper-cased)*; Type *(dropdown, `TRUCK_TYPES`)*; Capacity (MT); Status toggle; multi-photo upload for truck photos and documents; then an **Owner Details** group — Owner Name *(Title Case)*, Owner Mobile, Owner PAN, Owner Aadhaar, Owner Address; then Referred By *(text-suggestion field against the shared list)*.

**Validation.** Truck Number only. Everything else optional — a truck number captured mid-phone-call is a valid record.

**Detail.** All fields, document thumbnails, selective WhatsApp share, edit.

**Note.** Owner Name is **plain free text with no autocomplete and no link to Driver Master.** This is the one remaining unlinked reference in the application. See Section 13.1 for the designed-but-unbuilt fix.

---

### 6.3 Module B — Driver Master

**Purpose.** The driver register with licence and documents.

**Form.** Name *(required, Title Case)*; Contact; DL Number *(required)*; Status; multi-photo upload (driver photos, DL, Aadhaar); Referred By.

**Validation.** Name **and** DL Number.

**Detail.** Full record, document thumbnails, selective share, edit.

---

### 6.4 Module C — Party Master

**Purpose.** One register for every external counterparty, whatever role it plays.

**Form.** Party Name *(required, Title Case)*; Contact Number *(required)*; City *(Title Case)*; Address; GSTIN; PAN; Referred By; Saved Location; Bank Details (Account Number, IFSC, Account Holder Name); Status.

**Deliberate omission.** There is **no Type/role selector on this form.** Role is chosen at the point of use. Existing tags are preserved on edit, and new tags arrive only via quick-add pre-tagging.

**Saved Location handling.** The field accepts a pasted WhatsApp/Maps "share location" blob and **extracts only the URL**, discarding the surrounding text. The detail view renders it as a tappable maps link.

**Detail.** Full record including bank details and maps link; selective share; edit.

---

### 6.5 Module D — Product Master

**Purpose.** The materials carried.

**Form.** Name; Category; Default Unit of Measure *(`MT`, `KG`, `Box`, `Bag`, `Piece`)*; Status. List taps straight into the form — no detail screen.

**Note.** The product's `uom` is the master default. The bilty's goods lines have their **own** free-text Unit field backed by a separate 22-value suggestion list (Section 3.8), because a single material is ordered in different units on different trips — cement in bags on one load, MT on another.

---

### 6.6 Module E — Loading Challan

**Purpose.** The fast booking record. **This is the most speed-critical screen in the application** — it is filled while the user is still on the phone.

**Form sections.**

1. *Identification* — Serial No (auto, editable), Date.
2. *Truck & contacts* — Truck Number *(autocomplete against Truck Master, **free text accepted**, quick-add available)*; Driver Mobile; Owner Mobile; Owner Name. **On picking a Truck from the master, Owner Mobile and Owner Name auto-fill from that truck's record — but only if those fields are still blank**, never overwriting typed input.
3. *Route* — Loading Point, Unloading Point *(both required; autocomplete against Location Master; free text accepted; silently remembered)*.
4. *Rates* — Truck Hire Rate *(required)* with its own `Fixed`/`Per MT` flag; Truck Booking Rate *(required)* with its own independent flag. **No margin is displayed on this screen or its detail view** — removed at the owner's request.
5. *Party* — Party Name *(required; autocomplete; **free text accepted**; quick-add)*; Party Number *(the WhatsApp target for the loading slip)*.
6. *Load* — Weight (MT) — **a booking-time estimate only**.
7. *Supplier & broker* — Supplier *(autocomplete over the full party list, free text accepted, quick-add, plus a "Self" option)*; Broker *(same)*.
8. *Bilty Needed?* — **required `Yes`/`No` with no default. This answer selects the lifecycle path.**
9. *Notes* — repeatable free-form notes.

**Validation.** Named per missing field, never a blanket message: Truck Number (master or free text), Party name (master or free text), Loading Point, Unloading Point, Truck Hire Rate, Truck Booking Rate, Bilty Needed.

**List.** Date-grouped. Each row shows the truck, the route, and a computed status. A filter panel is available and its active state is indicated.

**Detail.** The full booking, the notes, a WhatsApp share, and path actions:
- Bilty Needed = `Yes` → **"Create Bilty"**, becoming **"View Bilty"** once one exists.
- **"Create Freight Challan"**, becoming **"View Freight Challan"** once one exists.
- **"Cancel Challan"** → a dedicated in-app form capturing a **required** reason, which sets `cancelled`, `cancelReason` and `cancelledAt`.

**Loading slip WhatsApp message.** See Section 9.2.

---

### 6.7 Module F — Bilty / GR

**Purpose.** The party-facing consignment note. The point at which data is made clean: unlike the loading challan, **every reference here must resolve to a master record.**

**Creation from a loading challan.** Carries forward: both rates and their type flags; consignor **and** consignee both pre-filled from the challan's single party field (both editable); the truck (or its number as a hint); goods weight seeded from the challan weight; freight pre-computed as booking rate × weight (or the fixed amount); remarks pre-filled with the route; and `sourceChallanId`. Defaults: `paymentType = To-Pay`, `showFreight = Yes`, all other disclosure toggles `No`, `billAtBookingRate = Yes`.

The source challan's **notes** are shown as a **read-only reference block on the form only** — computed fresh, never written into the saved bilty. They exist to be asked about during entry; they must not leak onto a document a driver or party might see. *(This was corrected: they used to be baked into `remarks` permanently.)*

**Form sections.**

1. Bilty No (auto, editable), Date.
2. Consignor, Consignee *(both required master links; autocomplete; quick-add pre-tagging the role)*; Third Party *(optional — "broker, referral, etc.")*.
3. Truck, Driver *(both required master links; the driver suggestion row shows mobile and DL to disambiguate same-name drivers)*.
4. **Goods** — repeatable lines of Product *(autocomplete + quick-add)*, Quantity, Unit, Weight. Changing a weight **recomputes the freight** (Section 7.1).
5. **Rates and billing** — Hire Rate and Booking Rate (carried, internal); **"Bill the Party at the Booking Rate?"** `Yes`/`No`; when `No`, a **Bilty Rate** and its type flag appear and become the rate the freight recalculates from.
6. Freight Amount *(required; auto-computed, manually overridable)*.
7. **Expenses** — a repeatable mini-form (type + amount). **Display-only; never enters the truck settlement.**
8. **Advances** — a repeatable mini-form: Amount, Mode, Paid By *(funding source)*, Date, Note. A live running balance is shown.
9. Payment Type *(`To-Pay` / `Paid` / `To-Be-Billed`)* and **Freight Paid By** *(`FREIGHT_PAYER_ROLES`; `Other` reveals a label)*.
10. **Disclosure toggles** — Show Freight, Show Hire Rate, Show Booking Rate, Show Advance, Show Expenses. See Section 9.1.
11. Documents — E-way Bill, Invoice, repeatable Other Documents.
12. Remarks.

**Validation.** Consignor, Consignee, Truck and Driver must each be **selected from suggestions** (a typed-but-unresolved name is not acceptable) and Freight must be present. The message names what is missing.

**On save.** Goods lines without a product are dropped. `showFreight` and `showExpenses` are **forced to `Yes`** for To-Pay. `biltyRate` is retained only when the override is active; `freightPaidByOtherLabel` only when the role is `Other`. If this bilty was created from a challan, the challan's `linkedBiltyId` is set. **Advances are then synced forward to the freight challan** (Section 8.2).

**Detail.** Renders as the **printed document** (Section 9.1), followed by: Export as PDF, Share via WhatsApp, the POD action (**"Mark Trip Completed (POD Received)"**, becoming **"View/Edit POD"** with its status once a record exists), and — **only when `sourceChallanId` resolves to a real challan** — "Create Freight Challan" / "View Freight Challan".

---

### 6.8 Module G — Freight Challan (the settlement engine)

**Purpose.** The truck's own settlement account for one booking. The digital truck-account notebook. **Internal only.**

**Invariants.** Always derived from a loading challan, never standalone. **Exactly one per loading challan** — if one already exists, both entry paths simply open it.

#### 6.8.1 Creation

**Path 2 (from a bilty).** `paymentType` and `freightPaidByRole` are inherited from the bilty. Load weight is the bilty's goods-weight total when non-zero, else the challan weight. Consignor and consignee come from the bilty. Broker resolves from the bilty's `thirdPartyId` first, else the challan's broker. **Bilty advances are converted and carried in, tagged `fromBilty`.**

**Path 1 (from a loading challan directly).** A **"Start Freight Challan"** screen first demands the two decisions the bilty would have carried — Payment Type and Freight Paid By — so payment type can never silently default. Consignor and consignee both pre-fill from the challan's single party and remain editable.

**Both paths.** Any **pre-existing POD record** is looked up at creation time and its received weight, status, photos, note, damage fields and To-Pay collection confirmation are carried in; the POD record is then retroactively linked. *(A POD may legitimately be recorded before its freight challan exists.)*

After creation the user lands **directly on the editable settlement screen**, not the read-only detail view — there is nothing useful to look at until data is entered. The back affordance still returns to the originating challan or bilty.

#### 6.8.2 Settlement Details screen

| Group | Fields |
|---|---|
| Parties | Supplier, Broker *(autocomplete, free text, quick-add, "Self" option)*; Consignor, Consignee *(**read-only when a bilty exists**, editable on Path 1)* |
| Rates & weight | Hire Rate + type; Load Weight; Unload Weight |
| Shortage | Shortage Tolerance (MT); Cutting Rate (₹/MT) |
| **Charges** | A dynamic list via an Add Charge mini-form: Type *(Charge Type Master)* → Direction *(pre-filled; **disabled for Commission, Shortage, Halting**)* → Amount, **or for Commission a `Fixed`/`Percentage` choice where a percentage is applied to truck freight and the computed amount stored** → Note |
| TDS | Mode `Fixed`/`Percentage`; Amount or Percent *(of truck freight, **recomputed live**)* |
| Diesel | Amount; Pump Name |
| **Advances** | A dynamic list via an Add Advance mini-form: Amount, Mode, Paid To, Date, Given By *(staff)*, **Paid By** *(funding source)*, Note. **A `fromBilty` entry cannot be deleted here** — the user is told to remove it on the bilty |
| Payment | Payment Type; Freight Paid By *(**read-only when a bilty exists**)* |
| Billing override | **Path 1 only** — "Bill the Party at the Booking Rate?" plus Bill Rate and type |
| Commission status | `Yes`/`No`, and when `Yes`, mode and date. **Hidden entirely when a Commission charge exists** — it is then already netted |
| Misc | Balance Payable At; Remark; Signature Collected |

Financial Year is computed and stored but **never displayed** (the owner judged it unnecessary).

#### 6.8.3 Detail screen — the Truck Account

Shows the complete settlement in account-statement order, with labels that change with the computed direction (Section 7.4): truck freight → each charge marked *cut from* or *added to* the freight → shortage workings → TDS → **gross** → the **final settlement figure** → advances already given → the dated ledger → **balance**.

**Before POD**, every figure is labelled a **live estimate**, and adding a ledger entry or marking settled is **blocked**.

**After POD**, actions become available:
- **Add ledger entry** — an inline mini-form (Amount, Mode, Paid By, Reference No., Note; date defaults to today) writing to `recoveries[]` or `payments[]` **according to the computed ledger direction**.
- **Mark as Fully Settled** — an in-app confirmation form requiring Amount, a **chosen** Date, who the money went to/came from *(with `Other` requiring a label)*, Mode, Paid By, Reference No. and Note. It appends a real ledger entry (noted "Marked fully settled"), in the correct direction. **It does not push a silent hard-coded entry** — that was corrected.
- **Share Settlement via WhatsApp** — Section 9.3.
- **Mark POD Received** / **View POD**.

**POD alert banner.** Where POD is outstanding, a banner surfaces it.

---

### 6.9 Module H — POD (Proof of Delivery)

**Purpose.** Record delivery confirmation and, for To-Pay trips, the amount the driver actually collected. **Gates settlement.**

**One record per delivery.** Raised from either the bilty or the freight challan; the lookup matches on **either** linked id, so opening POD from either side finds the same record rather than creating a duplicate. Opening the form when a record already exists switches to editing it.

**Form.** A read-only context card (truck, loading date, linked bilty and/or freight challan). Then: **POD Date** *(required)*; Received Weight *(pre-filled)*; **POD Photos/Documents** *(at least one required)*; Damage `Yes`/`No` with notes when `Yes`; Note.

**The To-Pay question.** For a To-Pay trip only, a required question headed:

> **"Any payment received by the driver from *[payer name]*?"**

with a plain `Yes` / `No` toggle. The payer label resolves from the role to the actual party name where one is linked, falling back to the role, then to "the party". **Both answers require an amount.** A live hint shows the balance still due from that payer. Whatever amount is entered is what is saved and what drives the settlement — **`Yes` must not silently assume the full expected figure** *(a corrected defect)*.

**Validation.** At least one photo; a POD date; and for a To-Pay trip, the collection question answered.

**On save — the sync.** Saving writes through to the linked documents: the bilty's `tripCompleted` is set; the freight challan's `podReceived` is set, its `podDocs` replaced with the photos, its note/damage fields updated, and the To-Pay confirmation and amount copied across. **The received weight is copied into the freight challan's `unloadWeight` only if that field is still blank** — the "never overwrite existing input" convention.

**Detail.** The record, its photos, the linked documents, and actions to **revert to "Not Received"** and later **re-confirm** without redoing the form. Reverting propagates: `tripCompleted` and `podReceived` both clear.

---

### 6.10 Module I — Payment Management

A lightweight universal ledger for money movements **not** tied to a trip — settling an old party balance, an office expense, an advance given before a booking exists.

**List.** Totals for Payment Out and Receipt In, then entries newest first, each signed (`−`/`+`) with category, party, mode, reference and date, badged by direction.

**Form.** Direction toggle; Paid To / Received From *(party autocomplete, free text accepted)*; Category; Amount *(required, > 0)*; Mode; Reference/Transaction No.; Date; Note. Editing an entry also offers Delete.

---

### 6.11 Module J — Reports & Analytics

Four reports, each computed live and each row linking to its underlying trip.

| Report | Content |
|---|---|
| **Commission Report** | Commission per freight challan where non-zero, plus a grand total |
| **Pending Recoveries** | Every `recover`-mode trip that is not settled, with its outstanding recovery balance, plus a total |
| **Pending Payables** | Every `pay`-mode trip that is not settled, with its outstanding balance, plus a total |
| **Payment Summary** | Total out, total in, net, and a breakdown by category, from standalone payment entries |

---

### 6.12 Module K — Transport Company Profile

One profile per account, used for the persistent company bar, the bilty letterhead, and the signature line on shared loading slips. Fields per Section 4.7. Reached by tapping the company bar from any screen. Also hosts **account switching** and **Add New Account**.

---

### 6.13 Module L — Multi-Account Support

Several entirely separate businesses in one installation. Switching **persists the current account's complete state** (all masters, all transactions, counters, profile) before loading the target's, and resets all transient state — navigation stack, every return context, every open mini-form, and the working draft. **Nothing merges, ever.** Adding a new account creates an empty data set and opens its company profile form so it can be named immediately.

---

### 6.14 Modules M / N — Location Master, User Management

**Location Master** — name, city, state, address, type, status. Also populated silently from loading challan route entry.

**User Management** — name, contact, role, branch, status. **Records only; no authentication exists.** The structure exists so that real multi-role login is an addition rather than a rework.

---

### 6.15 Module O — Charge Type Master

**Backend-only by explicit instruction — no list, add or edit screen.** The seeded table in Section 3.7 surfaces only as the Type picker inside the Add Charge form. Commission, Shortage and Halting Charges carry locked directions; the rest default and may be flipped per instance.

---

## 7. The calculation engine

**This is the most important section of the specification.** Implement it first, as a pure function, with the nine worked scenarios of BRD Section 9 as automated tests, before building any user interface.

### 7.1 Bilty freight recalculation

Triggered whenever a goods weight, the booking rate, the bilty rate, or the billing-override toggle changes.

```
effectiveRate     = billAtBookingRate == 'No' ? biltyRate : bookingRate
effectiveRateType = billAtBookingRate == 'No' ? biltyRateType : bookingRateType

if effectiveRate is zero/absent: do nothing   ← a from-scratch bilty stays fully manual

totalWeight = sum(goods[].weight)
freight     = effectiveRateType == 'Fixed' ? effectiveRate
                                           : round(effectiveRate × totalWeight)
```

The freight field stays directly editable, so an override works — it simply will not survive a later weight or rate change.

**Bilty running balance** (form display):
```
total   = paymentType == 'To-Pay' ? freight + sum(bilty charges) : freight
balance = total − sum(bilty advances)
```

### 7.2 The settlement function

A **pure** function of one freight challan, which resolves its linked bilty for the billing override. It returns a complete result object and **persists nothing**.

#### 7.2.1 Common preamble — both branches

```
weight     = loadWeight                          (0 if absent)
hasUnload  = unloadWeight is neither '' nor null
unload     = hasUnload ? unloadWeight : weight   ← defaults to loaded until known
shortage   = max(0, weight − unload)
tolerance  = shortageTolerance                   (0 if absent)
cuttingRate= cuttingRate                         (0 if absent)

hireFixed    = hireRateType    == 'Fixed'
bookingFixed = bookingRateType == 'Fixed'

truckFreight = hireFixed ? hireRate : weight × hireRate

shortageAmount = hireFixed ? 0
               : (shortage × hireRate) + (max(0, shortage − tolerance) × cuttingRate)
```

> **BR-07/BR-08.** The shortage deduction recovers the freight on the missing weight **and** applies a penalty only to the portion beyond the free tolerance. On a fixed hire rate there is no per-MT basis, so the deduction is zero even though the shortage is still recorded.

```
commission  = Σ amount where type == 'Commission'
halting     = Σ amount where type == 'Halting Charges'
adjustments = Σ ( direction == 'Recoverable' ? −amount : +amount )
```

> `adjustments` is a single signed total across **all** charges. `commission` and `halting` are extracted separately because each has its own special role — commission for reporting, halting for the consignee pass-through rule.

```
tdsAmount = tdsMode == 'Percentage' ? round(truckFreight × tdsPct / 100) : tdsAmount
dieselAmt = dieselAmount
advancesTotal = Σ advances[].amount + dieselAmt

isConsignee                = freightPaidByRole == 'Consignee'
skipShortageForConsignee   = isConsignee AND shortage > 0
```

**The billing override — where it is read from:**

```
linkedBilty = biltyId ? the bilty : null
billSource  = linkedBilty ?? this freight challan
overrideRate = linkedBilty ? linkedBilty.biltyRate : this.billRate

billOverrideActive    = billSource.billAtBookingRate == 'No' AND overrideRate > 0
effectiveBookingRate  = billOverrideActive ? overrideRate : bookingRate
effectiveBookingFixed = billOverrideActive
                          ? (linkedBilty ? linkedBilty.biltyRateType : this.billRateType) == 'Fixed'
                          : bookingFixed
```

> **BR-05.** The override affects **only** the party-side expectation below. `truckFreight` — the driver's own entitlement — is never touched by it.

Then the function branches on payment type.

#### 7.2.2 Branch 1 — `To-Pay` → result mode `recover`

The party pays the driver directly, so the business recovers its margin and commission **from the driver**.

```
bookingFreightExpected =
    effectiveBookingFixed ? effectiveBookingRate
  : isConsignee          ? (weight × effectiveBookingRate) + (halting > 0 ? halting : 0)
  :                        min(weight, unload) × effectiveBookingRate
```

> **BR-26 — three distinct cases.** A fixed rate is the trip total. When the **consignee** pays, the party is billed on the **full loaded** weight and any halting payable to the truck is **passed through** into what the party owes. Otherwise the party is billed on the **lesser of loaded and delivered** weight — the business does not bill for goods that did not arrive.

```
toPayConfirmed = toPayFreightReceived ∈ {'Yes', 'No'}
bookingFreight = toPayConfirmed ? toPayAmountReceived : bookingFreightExpected
toPayBalanceDue= toPayConfirmed ? max(0, bookingFreightExpected − bookingFreight) : 0
```

> **BR-27/BR-28.** Before POD, full collection is assumed and every figure is an **estimate**. After POD, the **confirmed** amount drives everything, the expected figure is retained for side-by-side display, and the shortfall becomes the balance still due from the party.

```
truckGross = truckFreight
           + adjustments
           − (skipShortageForConsignee ? 0 : shortageAmount)
           − tdsAmount

netPayableToDriver = truckGross − bookingFreight
```

> **BR-09.** When the consignee settles and there is a shortage, the shortage is **not** deducted from the truck — the consignee and the truck resolve it at the delivery point, and deducting it again would recover the same loss twice.

```
truckFreightFinal = (truckFreight + adjustments − shortageAmount − tdsAmount) − advancesTotal
```

> **Reference figure only.** Note it subtracts the shortage **unconditionally** (ignoring the consignee rule) and subtracts advances. It feeds no other figure. **BR-18: on a To-Pay trip, advances deliberately do not reduce the recovery.** Owner-confirmed. Do not "correct" it.

```
amountToRecover = netPayableToDriver < 0 ? −netPayableToDriver : 0
extraPayable    = netPayableToDriver > 0 ?  netPayableToDriver : 0

recoveriesTotal       = Σ recoveries[].amount
recoverableBalance    = max(0, amountToRecover − recoveriesTotal)

extraPayablePaidTotal = Σ payments[].amount
extraPayableBalance   = max(0, extraPayable − extraPayablePaidTotal)

isSettled = extraPayable > 0
              ? extraPayableBalance <= 0
              : (amountToRecover <= 0 OR recoverableBalance <= 0)
```

> **BR-22/BR-23 — the reversal, and the defect to avoid.** A negative net means recover from the driver; a **positive** net means the business **owes the driver**. In that case the trip's ledger flips to `payments[]` and settlement must track `extraPayableBalance` to zero. An implementation whose `isSettled` only consults `amountToRecover` / `recoverableBalance` will report such a trip settled the instant it is created — both are always zero when `extraPayable > 0` — leaving no way to record the money actually paid. **This was a real bug in the reference implementation. Test Scenarios B and B2 explicitly.**

**Returns:** `mode:'recover'`, `weight`, `unload`, `shortage`, `shortageAmount`, `skipShortageForConsignee`, `truckFreight`, `bookingFreight`, `bookingFreightExpected`, `toPayConfirmed`, `toPayBalanceDue`, `chargeRows`, `commission`, `tdsAmount`, `dieselAmt`, `advanceRows`, `advancesTotal`, `truckGross`, `netPayableToDriver`, `truckFreightFinal`, `amountToRecover`, `extraPayable`, `recoveriesTotal`, `recoverableBalance`, `extraPayablePaidTotal`, `extraPayableBalance`, `isSettled`, `billOverrideActive`, `effectiveBookingRate`, `effectiveBookingFixed`.

#### 7.2.3 Branch 2 — `Paid` or `To-Be-Billed` → result mode `pay`

The business pays the truck itself. **The party side is deliberately out of scope here** — see Section 13.2.

```
netPayableToTruck   = truckFreight + adjustments − shortageAmount − tdsAmount
finalPayableToTruck = netPayableToTruck − advancesTotal
paymentsTotal       = Σ payments[].amount
balance             = max(0, finalPayableToTruck − paymentsTotal)
isSettled           = balance <= 0
```

> **BR-17 — and the second defect to avoid.** Here advances **do** reduce what remains payable. This branch originally ignored advances entirely, so a driver already given cash still showed the full freight as payable — risking a double payment. **Test Scenario D.** Note also that the consignee shortage exemption does **not** apply in this branch.

**Returns:** `mode:'pay'`, `weight`, `unload`, `shortage`, `shortageAmount`, `truckFreight`, `chargeRows`, `commission`, `tdsAmount`, `dieselAmt`, `advanceRows`, `advancesTotal`, `netPayableToTruck`, `finalPayableToTruck`, `paymentsTotal`, `balance`, `isSettled`.

### 7.3 Percentage commission

When a commission charge is entered as a percentage, the amount is computed **at entry time** against the then-current truck freight and the **resulting amount** is stored (with the mode and percentage retained for display):

```
amount = round( (commissionPct / 100) × truckFreight )
```

Note the contrast with TDS, which is recomputed live on every read. **This asymmetry is as-built.** See Section 13.5.

### 7.4 Derived presentation direction

A shared helper computes, from the settlement result, how the trip should be *labelled and which ledger it writes to*. Both the detail screen and the WhatsApp statement use it, so they can never drift apart.

```
isExtraPayableSettlement = mode == 'recover' AND extraPayable > 0
ledgerDirection          = mode == 'recover'
                             ? (isExtraPayableSettlement ? 'pay' : 'recover')
                             : 'pay'
```

| | `mode = recover`, normal | `mode = recover`, extra payable | `mode = pay` |
|---|---|---|---|
| Gross label | Truck Gross (after Charges/Shortage/TDS) | *same* | Net Payable to Truck (after Charges/Shortage/TDS) |
| Gross amount | `truckGross` | `truckGross` | `netPayableToTruck` |
| Net label | **Net to Recover from Driver** | **Extra Payable to Driver** | **Final Payable to Truck** |
| Net amount | `amountToRecover` | `extraPayable` | `max(0, finalPayableToTruck)` |
| Balance label | Balance to Recover | Balance Still Payable to Driver | Balance Payable |
| Balance amount | `recoverableBalance` | `extraPayableBalance` | `balance` |
| Ledger written | `recoveries[]` | `payments[]` | `payments[]` |

Line phrasings used in both the detail view and the statement:

- Charge: `"Cut from Freight — {type}: ₹{amount}"` when Recoverable, `"Added to Freight — {type}: ₹{amount}"` when Payable, with an appended `" — {note}"` where present.
- Advance: `"Advance already given — ₹{amount} ({mode}, {date})"`.
- Ledger: `"Received on {date}: ₹{amount} ({mode})"` when recovering, `"Paid on {date}: ₹{amount} ({mode})"` when paying.

### 7.5 Commission status label

```
if a Commission charge exists  → "Already netted via Commission charge"
else if commissionPaid == 'Yes' → "Paid" + " · {mode}" + " · {date}"  (each if present)
else if commissionPaid == 'No'  → "Not Paid"
else                            → "Not asked yet"
```

### 7.6 Other derived values

**Financial year** — April to March, formatted `YYYY-YY`. Computed and stored; never displayed.

**Loading challan status** — derived from its cancellation state and whether a bilty and/or freight challan is linked.

**Bilty status badge** — two states only: `Delivered` when `tripCompleted`, else `Booked`. *(A four-state Booked/Dispatched/Delivered/Closed field was removed deliberately.)*

**Expected To-Pay amount, for the POD form** — where a freight challan exists and is in `recover` mode, its `bookingFreightExpected`; where only a bilty exists, `freight + Σ bilty charges`; otherwise zero.

---

## 8. Cross-module automation

These rules exist because the records are created in an unpredictable order and data must not be lost or duplicated as a result. **Each one addresses a real, observed failure.**

### 8.1 Auto-fill only into blank fields

A universal convention: an automation fills a target field **only when it is empty**, never overwriting what a human typed. Applies to owner mobile/name on the loading challan when a truck is picked, the POD's received weight into the freight challan's unload weight, and others.

### 8.2 Bilty advances → freight challan advances

**The problem.** An advance logged on a bilty (₹10,000 cash at loading) once failed to reach the freight challan and silently dropped out of the settlement — a direct cash loss.

**The rule.** Bilty advances are mirrored onto the freight challan's advance list, both **at creation** and **on every subsequent bilty save**. Each mirrored entry is tagged `fromBilty` so a re-sync replaces only the mirrored subset and never disturbs an advance entered directly on the freight challan. Mapping: the bilty's `{mode, amount, date}` becomes `{mode, amount, date, givenBy:'', paidTo:'Driver', fromBilty:true}` — `paidTo` defaults to `Driver` because advances at loading are conventionally handed to the driver, and stays editable. Date falls back to the loading challan's date.

A `fromBilty` entry **cannot be deleted on the freight challan**; attempting it tells the user to remove it on the bilty instead. Deleting it locally would make it reappear on the next bilty save.

> **Why both moments matter.** Booking the truck, creating the freight challan, then remembering the advance is a completely normal sequence. Syncing only at creation time loses it.

### 8.3 POD → bilty and freight challan

Covered in 6.9. Saving or reverting a POD propagates to both linked documents; received weight transfers only into a blank unload weight.

### 8.4 Pre-existing POD → new freight challan

A POD may be recorded before its freight challan exists. At freight-challan creation, any POD already linked to the bilty is found and its received weight, status, photos, note, damage fields and To-Pay confirmation are carried in; the POD record is then updated with the new freight challan's id so future POD edits propagate correctly.

### 8.5 Silent masters

Locations and referred-by names are captured from free-text entry with no prompt. See 6.1.5.

### 8.6 Linkage integrity

Editing a loading challan must preserve `linkedBiltyId`, `linkedFreightChallanId` and the cancellation fields. Both were real defects: editing a challan silently un-linked its freight challan, and editing a cancelled challan silently un-cancelled it. Build the record field by field and carry these forward explicitly.

### 8.7 Rate carry-through

The loading challan's hire and booking rates and their type flags flow to the bilty and on to the freight challan. The bilty retains the hire rate so it *can* be disclosed if ever needed, and the booking rate so freight can keep recalculating when the weight is corrected.

---

## 9. Output formats

### 9.1 The printed Bilty

Rendered as a document, exportable as PDF, shareable to WhatsApp. Order:

1. **Letterhead** — logo, company name (large, bold), address, contact, and GSTIN / PAN where present. All from the Company Profile.
2. Truck number (prominent) with its type.
3. Bilty number · date · status badge, marked "(ref. no.)".
4. **Consignor and Consignee** side by side (name + city), and a Third Party block where one is linked.
5. Payment Type and **Freight Paid By**, resolved to the actual party name where possible.
6. Driver name.
7. **Goods table** — Product | Qty (+ unit) | Weight (MT).
8. **Disclosure-gated financial rows.** This is where BR-36 to BR-38 bind:

| Row | Shown when |
|---|---|
| Hire Rate (+ "(Fixed)" or "(₹/MT)") | `showHireRate = Yes` **and** a hire rate exists. **Default off — the internal buy price** |
| Rate row | `showBookingRate = Yes`. **Carries the rate actually in effect** — the bilty rate when the override is active, the booking rate otherwise — as **one row only** |
| Freight | `paymentType = To-Pay` **or** `showFreight = Yes` |
| Expenses, Total Freight | To-Pay **and** expenses exist |
| Advance (Payment Type) | with the freight block |
| **Final Freight to be Paid** (To-Pay) / **Balance Due** (otherwise) | with the freight block |
| Expenses breakdown + total | (To-Pay **or** `showExpenses = Yes`) **and** charges exist |
| Advance breakdown | `showAdvance = Yes` **and** advances exist |

> **BR-38 restated, because it was a real leak.** When the override is active and the booking rate is disclosed, the document must show **one** row carrying the **billed** rate. Previously it showed the internal booking rate *and* a second "Bilty Rate" row regardless of the toggle, exposing the business's internal rate to the party.

9. Remarks, where present.
10. Documents strip (E-way Bill, Invoice, Other), where present.
11. Actions: Export as PDF, Share via WhatsApp, the POD action, and the freight-challan action.

Totals: `totalFreight = To-Pay ? freight + Σ charges : freight`; the closing figure is `totalFreight − Σ advances`.

### 9.2 Loading slip WhatsApp message

Opens with a bold title, closes with the company's own name so the recipient knows who sent it.

```
*Loading Slip*

Truck Number: {truck}
Loading Date: {dd Mmm yyyy}
Route: {loadingPoint} → {unloadingPoint}
Freight: ₹{bookingRate fixed ? bookingRate : bookingRate × weight}
Booking Rate: ₹{rate}{'/MT' or ' (Fixed)'}
Weight: {weight} MT
Party Name: {party}
Driver Mobile: {mobile}
*Note 1: {note}*
*Note 2: {note}*

{Company Name}
```

Every line is individually selectable and omitted when unticked. **Notes are wrapped in WhatsApp bold markup** — a note usually flags something the party must actually notice. **The hire rate is not among the available fields at all.** Target: the challan's party number, falling back to the party master's contact.

### 9.3 Settlement statement WhatsApp message

Composed in **account-statement order**, with labels taken from the derived direction of Section 7.4 so it always reads correctly whichever way the trip settled.

```
Truck Number: {truck}
Payment Type: {To-Pay | Paid | To-Be-Billed}
Truck Freight: ₹{truckFreight}
Cut from Freight — Commission: ₹{amount}            ← per charge, each unticked by default
Added to Freight — Halting Charges: ₹{amount}
{grossLabel}: ₹{grossAmt}
{netLabel}: ₹{netAmt}
Advance already given — ₹{amount} ({mode}, {date})  ← per advance, unticked by default
Received on {date}: ₹{amount} ({mode})              ← per ledger entry, unticked by default
{balanceLabel}: ₹{balanceAmt}
POD Status: {Received | Pending}
```

Ticked by default: truck number, payment type, truck freight, gross, net, balance, POD status. **Unticked by default: every individual charge, advance and ledger line.** Target: the freight challan's owner mobile, falling back to the driver mobile; the chooser states which of the two it will reach.

### 9.4 Formatting conventions

| Item | Convention |
|---|---|
| Money | `₹` + Indian digit grouping: `₹1,20,500` — **not** `₹120,500` |
| Dates (display) | `10 Nov 2026` |
| Dates (stored/inputs) | ISO `YYYY-MM-DD` |
| Weight | decimal MT, suffixed `MT` |
| Truck numbers | UPPER CASE |
| Names, cities, places | Title Case |
| Mobile for WhatsApp | `91XXXXXXXXXX` |

---

## 10. UI and UX conventions

### 10.1 Design tokens

| Token | Value | Use |
|---|---|---|
| Primary | `#4F46E5` | Primary actions, active tab, accents |
| Primary dark | `#3730A3` | Pressed states |
| Primary light | `#EEF0FF` | Tinted backgrounds |
| Background | `#F4F5F9` | Screen background |
| Card | `#FFFFFF` | Surfaces |
| Text | `#1F2333` | Primary text |
| Text muted | `#6B7280` | Secondary text, labels |
| Border | `#E5E7EB` | Dividers, inputs |
| Green / bg | `#16A34A` / `#DCFCE7` | Active, delivered, settled |
| Amber / bg | `#B45309` / `#FEF3C7` | Pending, warning |
| Red / bg | `#DC2626` / `#FEE2E2` | Destructive, overdue |
| Gray / bg | `#6B7280` / `#F1F2F6` | Neutral badges |

System font stack. Mobile-first single-column layout.

### 10.2 Component vocabulary

**List item** — title, one or two subtitle lines, a trailing badge or amount; the whole row is the tap target.
**Card** — a grouped read-only block of label/value rows.
**Stat card** — a large number over a small label, in a grid.
**Badge** — small pill, coloured by semantic state.
**Field label** — above its input, with a red asterisk when required.
**Field hint** — small muted text below an input, for guidance and live computed hints.
**Segmented toggle** — two or three mutually exclusive options as adjacent tappable cells. **This is the standard for every Yes/No and short enumeration — never a dropdown.**
**Dropdown** — only for genuinely long lists that are not searchable (e.g. Truck Type).
**Autocomplete** — per 6.1.3.
**Inline mini-form** — the standard for adding a repeatable sub-record (charge, advance, ledger entry, goods line, note): a disclosure button expands a small form in place, saving appends to the list and collapses it. **Never a modal, never a native dialog.**
**Modal overlay** — used only for the share checklists.
**Toast** — transient confirmation and validation messages.
**Section title** — small muted uppercase heading grouping a form or detail region.
**Upload chip** — a labelled chip per uploaded document, with removal.
**Empty state** — a large glyph plus one line of text.

### 10.3 Interaction principles

**Validation names what is missing** — "Please fill: Truck Number, Loading Point, Truck Hire Rate", never "please fill all required fields".

**Mini-form state is isolated per screen.** The bilty's and freight challan's charge/advance mini-forms keep entirely separate state so the two screens can never interfere.

**Destructive and irreversible actions use in-app forms**, capturing a reason where one is meaningful (challan cancellation, mark fully settled).

**Buttons state the current reality** — "Create Bilty" becomes "View Bilty"; "Mark POD Received" becomes "View/Edit POD (Received)".

**Blocked actions are explained, not hidden** — a pre-POD settlement says the figures are an estimate and that money entry awaits POD.

---

## 11. Non-functional requirements

| # | Requirement |
|---|---|
| **NFR-1** | **Platform:** Android and iOS. |
| **NFR-2** | **Offline-first — architectural, Phase 1.** Records must be creatable and editable offline, syncing automatically on reconnection. **Decide the data layer before development begins** — this cannot be retrofitted onto a server-round-trip design without a rewrite. |
| **NFR-3** | **Multi-account isolation.** Every query scoped to the active account. No cross-account read path. |
| **NFR-4** | **Decimal money.** Never floating point. Weights carry decimals (0.5 MT is normal). |
| **NFR-5** | **Indian locale** throughout — digit grouping, date format, mobile normalisation. |
| **NFR-6** | **Role structure present from Phase 1** (Admin/Staff records) so enforcement is an addition, not a rework. |
| **NFR-7** | **Computed figures are never stored.** Settlement, balances and report totals are derived on read, so correcting an input corrects everything downstream. |
| **NFR-8** | **Usability:** a loading challan raisable in under a minute on a phone, one-handed, from a standing start, with only a truck number and a party name. |
| **NFR-9** | **Document retention:** multiple photos/PDFs per truck, driver and POD, retrievable with thumbnails and zoom. |
| **NFR-10** | **Share integrity:** no outbound message may include an unticked field. |
| **NFR-11** | **No native dialogs** anywhere. |
| **NFR-12** | **Numeric inputs** immune to accidental scroll changes. |
| **NFR-13** | **Scale:** tens of trips per month, 1–3 concurrent users. Optimise for correctness and single-record entry speed, not throughput. |

---

## 12. Acceptance criteria

### 12.1 Calculation engine — mandatory, build first

Implement all nine scenarios from **BRD Section 9** as automated tests. Expected values, from the reference engine:

| Scenario | Key expected values |
|---|---|
| **A** To-Pay, consignee, full collection | truckFreight 18,000 · expected 20,700 · gross 17,000 · net −3,700 · **recover 3,700** · truckFreightFinal 12,000 · settled false |
| **A0** same, pre-POD | toPayConfirmed false · bookingFreight 20,700 (assumed) · **recover 3,700 as estimate** |
| **B** To-Pay, collected 15,000 | expected 20,700 · collected 15,000 · **party balance due 5,700** · gross 17,000 · net +2,000 · recover 0 · **extraPayable 2,000** · settled false |
| **B2** B after paying 2,000 | extraPayable 2,000 · paid 2,000 · **balance 0 · settled true** |
| **C** consignor pays, shortage 0.5 MT (tol 0.2, cut 500) | **shortageAmount 650** · skip false · truckFreight 20,000 · expected 22,425 · gross 18,350 · **recover 4,075** |
| **C2** consignee pays, same shortage + 1,500 halting | shortageAmount 650 · **skip true** · expected **24,500** · gross 20,500 · **recover 4,000** |
| **D** To-Be-Billed, halting 1,500, TDS 1%, advance 10,000, paid 5,000 | mode pay · truckFreight 19,200 · **TDS 192** · netPayable 20,508 · **finalPayable 10,508** · **balance 5,508** |
| **E** To-Pay, bill rate override 1,250/MT | overrideActive true · effectiveRate 1,250 · expected 22,500 · **gross unchanged 17,000** · **recover 5,500** |
| **F** fixed hire 20,000 / fixed booking 24,000, 18→17 MT | truckFreight 20,000 · shortage 1 MT · **shortageAmount 0** · expected 24,000 · **recover 5,000** |

### 12.2 Lifecycle

- Path 2 completes: loading challan → bilty → freight challan → POD → ledger → settled.
- Path 1 completes, with the Start screen demanding payment type and freight payer.
- A second attempt to create a freight challan from the same loading challan **opens the existing one** — no duplicate.
- A POD raised from the bilty and then opened from the freight challan resolves to the **same** record.
- A POD recorded **before** its freight challan has its data carried forward at creation and is retroactively linked.

### 12.3 Automation

- An advance added to a bilty **after** its freight challan exists appears on the freight challan on bilty save, tagged as bilty-sourced.
- A bilty-sourced advance cannot be deleted on the freight challan; the user is directed to the bilty.
- An advance added directly on the freight challan survives a bilty re-sync.
- POD received weight fills a **blank** unload weight and leaves a populated one untouched.
- Editing a loading challan preserves its bilty link, freight challan link, and cancellation state and reason.

### 12.4 Disclosure

- A To-Pay bilty always shows the freight amount.
- A Paid bilty hides the freight amount by default.
- The hire rate is absent unless explicitly switched on.
- With the override active and the booking rate disclosed, **exactly one** rate row appears, carrying the **billed** rate.
- Charges, advances and ledger lines are unticked in every share checklist by default.
- With no valid number on file, the composed message reaches the clipboard.

### 12.5 Gating and validation

- Pre-POD, ledger entry and "mark fully settled" are blocked and the figures labelled an estimate.
- A To-Pay POD cannot be saved without the collection question answered, and either answer requires an amount.
- A POD cannot be saved with no photo or no date.
- A bilty cannot be saved without master-resolved consignor, consignee, truck and driver plus a freight amount.
- A loading challan **can** be saved with only a free-text truck number and party name (plus route, both rates, and bilty-needed).
- Validation messages name the specific missing fields.

### 12.6 Masters and data hygiene

- Quick-add from any reference field returns to the in-progress form with everything intact and the new record selected.
- Backing out of a quick-add without saving also restores the in-progress form.
- Names save in Title Case, truck numbers in upper case, **spelling unaltered**.
- A location typed once is suggested thereafter without any prompt.
- Switching accounts preserves both accounts' data with no leakage, and resets all transient state.

### 12.7 Reports

Pending recoveries lists exactly the unsettled `recover`-mode trips and totals their recovery balances; pending payables does the same for unsettled `pay`-mode trips; both reconcile to the trips they link to.

---

## 13. Recommended improvements

**Everything above this line describes the system as built. Everything below is a recommendation and is not part of the current system.** Each item states the current behaviour, the problem, and a suggested direction. Implement at the receiving team's and the owner's discretion.

### 13.1 Link truck owners to driver records *(designed, approved in principle, not built)*

**Current.** A truck's owner details (`ownerName`, `ownerMobile`, `ownerPan`, `ownerAadhar`, `ownerAddress`) are plain free text on the truck record, with **no autocomplete and no link to Driver Master**. This is the only remaining unlinked reference in the application.

**Problem.** Owners are frequently also drivers — in the seeded data, "Jainul Ansari" exists as both a truck owner and a driver, as two unrelated records. The database cannot express "driver X owns truck Y" or "truck Y's owner is currently driving truck Z", and the same person's details are maintained twice.

**Recommended design** (worked out with the owner; awaiting his go-ahead):

1. Make Owner Name an **autocomplete against Driver Master**, using the existing pattern from 6.1.3 — so an existing driver of that name surfaces as a pickable suggestion.
2. Add **one optional field** to the truck: `ownerDriverId`. Set only when a suggestion is picked. Owner Mobile auto-fills from the driver's contact on linking (still overridable, per 8.1). The free-text name remains for the unlinked case, so nothing is forced.
3. **Derive both reverse views; store nothing extra.** On Driver Detail, "Also Owner of: {trucks where ownerDriverId = this driver}". On Truck Detail, "Owner also driving: {trucks whose trips carry this driver as driverId}" — computed from existing bilty and freight-challan `driverId` values, so it stays correct automatically as trips happen.

**Open questions for the owner.** Whether a non-matching owner name should offer "➕ Add as new Driver" (consistent with every other quick-add, but it creates more driver records); whether existing name matches should be auto-linked retroactively or only going forward; and whether a mistaken link should be removable.

### 13.2 Party-side receivables ledger — the largest gap, and the natural integration point

**Current.** The freight challan tracks the **truck side** of each trip in full depth. The party side — what the customer owes — is visible on the bilty but has **no ledger, no ageing and no statement**. On a `Paid` / `To-Be-Billed` trip the specification explicitly declares the party side out of scope.

**Problem.** The business cannot answer "what does party X owe us, and how old is it?" The data to answer it is all present (bilty freight, To-Pay collection confirmations, `toPayBalanceDue`, standalone payment entries) but is not assembled anywhere.

**Recommendation.** This is the **single highest-value extension**, and the most natural place for the parent application to add value — most parent systems of this kind already have a receivables or customer-ledger module. Suggested shape: a receivable posted per bilty at the booking/bilty rate; To-Pay collections confirmed at POD posted as part-settlements; the `toPayBalanceDue` figure carried as a live receivable; standalone `Receipt In` entries applied against it; and a party statement plus an ageing report. Reuse the existing settlement-statement WhatsApp composer for party statements.

### 13.3 Real persistence, authentication and audit

**Current.** The reference implementation is entirely in memory — reloading loses everything. User records exist with roles but **no authentication and no permission enforcement**.

**Recommendation.** Three things the production build needs on day one: a real persistent store with an offline-first sync layer (NFR-2); actual login with the Admin/Staff distinction enforced; and a **change-audit log** recording every create and edit with the field changed, the previous and new value, the timestamp and the user. The owner explicitly deferred the audit log to Phase 2, but it is substantially cheaper to build into the data layer from the start than to add later.

### 13.4 Document storage

**Current.** Photo and document uploads are **placeholder labels only** (`"Photo 1"`, `"POD Photo 2"`) — no actual files are stored. Every upload affordance across trucks, drivers, bilty documents and POD is a stub.

**Recommendation.** Real file storage with thumbnail generation, client-side compression before upload (these are phone-camera photographs over patchy mobile data), offline queueing of pending uploads, and inclusion of actual images in PDF exports and WhatsApp shares. **PDF export is also currently a stub** — it shows a confirmation toast and generates nothing.

### 13.5 Consistency clean-ups

Each of these is a small, safe improvement that the as-built system does not make.

- **Percentage commission freezes; percentage TDS does not.** A percentage commission computes its amount at entry time and stores it, so a later correction to weight or hire rate leaves it stale. TDS recomputes live. Recommend making commission live too — with the caveat that the stored amount may have been deliberately rounded or renegotiated, so confirm with the owner first.
- **Legacy fields.** `bilty.advance` (always `'0'`, superseded by `advances[]`) and `freightChallan.commStatus` (superseded by `commissionPaid`) should be dropped rather than migrated.
- **Loose id comparison.** The reference source compares ids with `==` in places and `===` in others. A production build should type ids consistently and compare strictly.
- **Money as strings.** Amounts are stored as strings and coerced on read. Use a decimal type.
- **`truckFreightFinal`.** A reference-only figure that subtracts the shortage unconditionally, ignoring the consignee exemption that the main path honours. It feeds nothing, but it can be displayed. Either align it or label it unambiguously.
- **No list pagination or server-side search.** Fine at tens of records per month; revisit if volume grows.

### 13.6 Payment Management scope

**Current.** The module's stated intent is to show freight-challan ledger entries *alongside* standalone entries in one combined view. **The implemented list shows only standalone entries** — the combined view was never built.

**Recommendation.** Either build the combined read-only view as intended, or restate the module's scope as "non-trip money movements only". The current state is a mismatch between stated intent and behaviour, and should be resolved explicitly rather than inherited.

### 13.7 Deferred features from the business roadmap

Expense & fuel management (fuel, tyres, maintenance, driver salary, office costs, with per-truck profitability); the remaining reports (trip register, truck-wise and driver-wise revenue, monthly revenue versus expense); GST fields, **only** if the business registers; a party-facing document portal, should the owner ever want parties to self-serve rather than receive WhatsApp messages.

---

## 14. Appendix — using the reference prototype

`logitrack_prototype.html` is a single self-contained file requiring no build step, no server and no network. Open it in any modern browser; it renders at mobile viewport width with a simulated device frame.

**What it is good for.** Seeing exactly how every screen behaves, how the navigation and autocomplete feel, what each form contains, and — most importantly — **running the real calculation engine against your own inputs** to resolve any ambiguity in Section 7.

**What it is not.** It has **no persistence whatsoever** — reloading resets everything to seed data. It stores no real files. Its PDF export is a stub. It is a single 4,500-line file with global mutable state and string-template rendering: a design and behaviour reference, **not an architectural model**.

**Where to look.** The settlement engine is `computeFreightChallanCalc`. The presentation-direction helper is `freightStatementCtx`. The autocomplete pattern is `attachAutocomplete`. The bilty-advance sync is `syncBiltyAdvancesToFreightChallan`. The POD sync is `syncPodToLinkedDocs`. The screen dispatcher is `render`, which maps every screen id from Section 5.4 to its render function.

**Seed data.** Five trucks (including the owner's real 22-wheelers), six drivers, eight parties, four products, two locations, three bilties, one loading challan, and a second demo account for testing account switching. The seeded records are the owner's own fleet, parties and material, and make useful realistic test fixtures.

**To extract and test the engine yourself:** pull the `<script>` block, isolate `computeFreightChallanCalc`, stub an empty `bilties` array, and call it with a freight-challan object. This is exactly how the Section 12.1 expected values were produced.

---

*End of specification. Business rationale, the domain glossary and the worked scenarios are in `LogiTrack_BRD.md`. The working reference implementation is `logitrack_prototype.html`.*
