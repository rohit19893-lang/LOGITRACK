# LogiTrack

A mobile operations app for an Indian road-freight brokerage. It digitises the business's
master records, its booking-to-delivery document chain, and — the hard part — the per-trip
money settlement with each truck.

Used by the business owner plus one or two office staff. No driver, customer or other
external party ever logs in; they receive documents by WhatsApp.

## Repository contents

```
logitrack_prototype.html   The entire application — one self-contained file
docs/                      Full specification set
README.md
```

### The application

`logitrack_prototype.html` is the whole application: all HTML, CSS and JavaScript in a
single document, roughly 4,530 lines. **No build step, no dependencies, no server.** Open
it in any browser and it runs, rendering at mobile viewport width in a simulated device
frame.

It covers 39 screens, seven master records, the four-document transaction chain, and the
complete settlement engine.

### The documentation

| File | What it is |
|---|---|
| `docs/LogiTrack_BRD.md` | **Business Requirements Document.** Business context, stakeholders, a 40-term domain glossary, the end-to-end lifecycle, **nine worked numeric scenarios**, and 53 numbered business rules. |
| `docs/LogiTrack_Functional_Spec.md` | **Build specification.** Technology-agnostic. Complete data model, all 39 screens, every field and validation, **the calculation engine as annotated pseudocode**, output formats, design tokens, non-functional requirements, acceptance criteria, and a separated recommendations section. |
| `docs/LogiTrack_Handoff_Brief.md` | Covering note for handing the package to another team or agent. |
| `docs/LogiTrack_PRD.md` | Product requirements, module by module, with the decision history. |
| `docs/LogiTrack_Build_Log.md` | Full change history, v0.1 through v24, with the reasoning behind every decision. |
| `docs/LogiTrack_ChatGPT_Comparison_Report.md` | A design comparison against an earlier brainstorming session. |

## Read this before building on it

**The prototype is a design and behaviour reference, not a production codebase.**

- It has **no persistence whatsoever** — reload and everything resets to seed data.
- Photo and document uploads are **placeholder labels**; no files are stored.
- PDF export is a **stub** that shows a confirmation and generates nothing.
- It is a single file with global mutable state and string-template rendering.

Do not branch off it and extend it. Read the specification, build properly in whatever
stack suits, and keep the prototype open beside you to check behaviour and to run the
settlement arithmetic against your own inputs.

## Suggested reading order

1. **`docs/LogiTrack_BRD.md` Section 7 — the glossary. First, and non-optionally.** This is
   Indian road-freight trade vocabulary: bilty, freight challan, To-Pay, munshiyana, kanta,
   dala, shortage cutting. The failure mode if you skip it is plausible-looking software
   that computes the wrong numbers.
2. `docs/LogiTrack_BRD.md` Sections 8 and 9 — the two lifecycle paths, and the nine worked
   scenarios with exact figures.
3. `docs/LogiTrack_Functional_Spec.md` Sections 3, 4 and 7 — reference data, data model,
   calculation engine.
4. The rest of the specification per module, as you build.

## Start with the money

Build the settlement engine first, as a pure function, with the nine scenarios from BRD
Section 9 as automated tests, **before any UI work**. Those figures were produced by
executing the prototype's own settlement function, so they are exact rather than
illustrative. Functional Spec Section 12.1 tabulates the expected values.

The engine is genuinely subtle. The arithmetic **reverses direction** depending on who pays
the freight, and there is an edge case where a partial collection at delivery flips the
business from recovering money from the driver to owing money to the driver. Functional
Spec Section 7 flags, inline, the two real defects this caused during development — make
sure a reimplementation does not reproduce them.

## Deliberate decisions, not oversights

Several things look odd and are intentional, owner-confirmed business rules. The
specification marks each one. Among them: the shortage deduction is waived when the
consignee settles the freight; advances deliberately do not reduce a To-Pay recovery; and
the truck-type list mixes wheel-count, length and body-type classes on purpose.

Out of scope by explicit decision: no driver app, no GPS tracking, no payment gateway, no
GST, no customer portal. See BRD Section 5.2 for the reasoning on each.
