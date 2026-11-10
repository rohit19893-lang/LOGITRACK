# LogiTrack — Handoff Brief

**Paste the text below into the first message of the receiving Claude account, and attach the three files it names.**

---

## Copy from here

I'm handing you a complete specification for an existing application called **LogiTrack**, and I want you to build it (or integrate it) into a parent application I already have.

**Attached files:**

1. **`LogiTrack_BRD.md`** — Business Requirements Document. Business context, stakeholders, the domain glossary, the end-to-end lifecycle, nine worked numeric scenarios, and a catalogue of 53 numbered business rules.
2. **`LogiTrack_Functional_Spec.md`** — the build specification. Complete data model, all 39 screens, every field and validation, the full calculation engine as pseudocode, output formats, UI conventions, non-functional requirements, and acceptance criteria.
3. **`logitrack_prototype.html`** — a working reference prototype. A single self-contained file; open it in a browser, no build or server needed. Every screen and every calculation described in the spec is live in it.

**What LogiTrack is.** A mobile operations app for an Indian road-freight brokerage. It digitises the business's master records (trucks, drivers, parties, materials), its document chain (loading challan → bilty/consignment note → freight challan → proof of delivery), and — the hard part — the per-trip money settlement with each truck. It is used by the owner plus one or two office staff. No driver, customer or other external party ever logs in; they receive documents by WhatsApp.

**Please read in this order:**

1. **`LogiTrack_BRD.md` Section 7, the glossary — first and non-optionally.** This is Indian road-freight trade vocabulary (bilty, freight challan, To-Pay, munshiyana, kanta, dala, shortage cutting). The failure mode if you skip it is plausible-looking software that computes the wrong numbers.
2. **`LogiTrack_BRD.md` Sections 8 and 9** — the two lifecycle paths and the nine worked scenarios with exact figures.
3. **`LogiTrack_Functional_Spec.md`** Sections 3, 4 and 7 — reference data, data model, calculation engine.
4. The rest of the spec per module as you build.

**Please start by building the calculation engine, as a pure function, with the nine scenarios in BRD Section 9 as automated tests — before any UI work.** Those figures were produced by running the reference prototype's actual settlement function, so they are exact, not illustrative. Spec Section 12.1 tabulates the expected values. The engine is where the business's money lives, and it is genuinely subtle: the arithmetic **reverses direction** depending on who pays the freight, and there is an edge case where a partial collection flips the business from recovering money from the driver to owing money to the driver. Spec Section 7 flags, inline, the two real defects this caused in the reference implementation — please make sure your version does not reproduce them.

**A few things to know about the specification:**

- It is deliberately **technology-agnostic** — it says what to build, not how. Pick whatever fits the parent application. The prototype is a design and behaviour reference, **not** an architectural model: it has no persistence at all and holds everything in memory.
- Spec **Sections 2–12 are as-built** and authoritative. **Section 13 is clearly separated and contains recommended improvements only** — known gaps and upgrades, none of which exist in the current system. Treat them as optional, and check with me before building any of them.
- **Offline-first is an architectural requirement, not an enhancement** (Spec NFR-2). Staff work from phones at mines, depots and roadsides with poor connectivity. This needs deciding before you choose a data layer; it cannot be retrofitted.
- Several design decisions look like oddities but are **deliberate, owner-confirmed business rules** — the shortage exemption when the consignee pays, advances deliberately not reducing a To-Pay recovery, and the mixed-axis truck-type list. The spec marks each one. Please don't "fix" them.
- The out-of-scope list in BRD Section 5.2 was each considered and rejected on purpose: no driver app, no GPS tracking, no payment gateway, no GST, no customer portal.

**My plan is to integrate this into a parent application I already have.** Spec Section 13.2 identifies the natural integration point: LogiTrack tracks the *truck* side of every trip in full depth, but the *party* side — what customers owe us — has no ledger, ageing or statement, even though all the data to build one is already captured. If the parent application has a receivables or customer-ledger module, that is where the two should meet.

**Before you start building, please tell me:** what you'd use to build it, how you'd structure the data layer given the offline-first requirement, and anything in the specification you find ambiguous or think is wrong. I'd rather resolve that up front than after the fact.

## Copy to here

---

## Notes for Mohit (not part of the pasted text)

**What's in the package.** Three documents plus the prototype. The BRD is the "why", the Functional Spec is the "what to build", the prototype is the "here's it working". Between them, another Claude account should be able to build this with no further explanation from you — which was the goal.

**The most important thing in the package** is the nine worked scenarios with exact rupee figures. I ran every one of them through the prototype's real settlement function rather than working them out by hand, so they're exactly what your app produces today. Any rebuild that reproduces all nine is arithmetically correct; one that doesn't, isn't. I've written them up as the first acceptance test so whoever builds it has to get the money right before they get to the screens.

**On the improvements section.** You asked for as-built plus a separate improvements list, so Section 13 of the Functional Spec is walled off and clearly marked as not-yet-built. It covers: the owner-is-also-a-driver linking we designed but haven't built; the party-side receivables ledger (the biggest real gap, and the most likely place your parent app adds value); real persistence, login and an audit log; actual file storage, since every photo upload in the prototype is a placeholder; and a handful of small consistency clean-ups. Nothing there has been built into the app.

**One thing worth flagging.** While documenting the Payment Management module I found a mismatch: its stated purpose is to show the freight-challan ledger entries alongside standalone entries in one combined view, but the screen only ever shows standalone entries — the combined view was never actually built. It's noted as Section 13.6. Worth deciding which way you want it before the other team inherits the ambiguity.

**Still pending from before this.** The owner-is-also-a-driver design is waiting on your okay, plus three small decisions: whether an unmatched owner name should offer "add as new Driver"; whether to retro-link the existing name matches in your data; and whether a wrong link should be removable. Say the word and I'll build it.
