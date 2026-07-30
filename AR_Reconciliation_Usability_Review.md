# AR Reconciliation — Prototype Usability Review

**Scope:** How usable is the current prototype (`index copy.html`, product name "Stack Books") for real‑world B2B AR reconciliation, measured against `AR_Reconciliation_Proposal.docx` (Three‑Tier Resolution Workflow) and `AR_DB_Schema.docx` (Schema v6.0).
**Focus (as requested):** (1) UX gaps around the Rules Studio, (2) logical gaps in the Rules Studio.
**Date:** 2026‑07‑29

---

## 1. Executive summary

The prototype is **substantially more faithful to the proposal than a typical demo**. All six phases exist as *named, cascading, first‑match‑wins* rules in a real engine (`arEngine`): Phase 1A customer‑lock (dup‑UTR, expected‑UTR, account+IFSC, UPI, customer‑code, GSTIN/PAN, fuzzy‑name), Phase 1B candidate pooling, Phase 2 scoped allocation (rules 2.1–2.9), plus short‑pay / unapplied‑cash / GL‑control threshold checks. It genuinely handles candidate‑pool resolution, the "Double Collision" exception, TDS, explicit‑fee decoupling, small‑balance write‑off, overpayment→on‑account, partial payments, and a sub‑ledger↔GL control proof. Exceptions route to a real inbox; the Reports module gives a read‑only archive and audit trail.

**Verdict by use‑case:**

| Use it as… | Usable? | Notes |
|---|---|---|
| **Design / stakeholder validation prototype** | ✅ High (~80% conceptual coverage) | The workflow, rule taxonomy, and exception vocabulary are all demonstrable end‑to‑end on seeded + uploaded CSVs. |
| **UX reference for the real build** | ✅ Good, with the gaps below | The Rules Studio is the right shape but has explainability/feedback gaps that matter for a rule‑tuning surface. |
| **Operational AR tool** | ❌ Not yet | localStorage only, no DB/auth/integrations, no JE posting, no cryptographic run hash, no multi‑currency, and several engine simplifications (below). |

The rest of this document is the *gap list* — the strengths above are real, but the point of the review is what still blocks it.

---

## 2. Gaps in the UX around the Rules Studio

> The Rules Studio (`arReconRules` → `arRuleCard` → `arRuleEditor`) lets you add/edit/reorder/enable rules per phase with a live per‑rule match count. These are the friction points that would surface with a real reconciler.

**U1 — No aggregate simulation feedback (only per‑rule counts).** *[High]*
The generic engine's Rules Studio has a match‑rate ring + "vs last run" delta; the **AR** Rules Studio does not. Editing a rule updates each card's "N matched" count, but there's no headline "match rate went 71% → 84%, breaks 40 → 22" summary. A reconciler tuning tolerances can't see the *net* effect of a change without leaving for the Overview/Matches tabs.

**U2 — No near‑miss / "why didn't this match" explainability.** *[High]*
A rule card shows how many rows it matched, but never **which** rows, and — more importantly — never the rows it *just missed* (e.g., "3 payments were ₹40–₹90 outside your ±₹100 fee tolerance"). Tolerance tuning is guesswork without near‑miss visibility. This is the single biggest missing affordance for a rules studio.

**U3 — No dry‑run / test‑a‑row panel.** *[Med]*
You can't paste a sample narration/amount and see which customer it would lock and which invoice it would allocate to. Every real matching‑rules product ships a "test against this record" box to build trust before enabling a rule.

**U4 — Rules can't be deleted, only disabled — but *can* be duplicated.** *[Med]*
By design, "Add rule" can add a second "Exact amount" (or any kind) to a phase, yet there is no delete — only enable/disable and "Reset to defaults" (which nukes *all* edits globally). Result: clutter accumulates with no clean way to remove it, and there's no per‑rule revert.

**U5 — Confidence slider is shown even where confidence does nothing.** *[Med]*
Every card renders a `confBar`/confidence slider (min 50, max 100). For some kinds it's load‑bearing (`fuzzy-name` → match threshold %; short‑pay/unapplied/GL → tolerance), for others it's purely decorative (`dup-utr`, `partial-payment`, `overpayment`). Showing a tunable control that has no effect trains users to distrust the panel. The semantics ("confidence score" vs "match threshold" vs "tolerance") are conflated in one widget.

**U6 — Phase 2.0 guardrails are invisible and non‑configurable.** *[High]*
The proposal makes the **accounting‑period cutoff** (Phase 2.0a) and **credit/debit net‑off** (Phase 2.0b) explicit pre‑filters. In the engine they're hardcoded: `periodEnd = todayISO()` and `effectiveBalance = total − creditNotes + debitNotes`. The Rules Studio exposes *no* control to set the reconciliation period‑end date or govern net‑off behaviour — two settings a controller must own each cycle. They should be surfaced as a "Phase 2.0 guardrails" block at the top of the studio.

**U7 — Powerful field pickers with no schema‑aware validation.** *[Med]*
The Source/Field pickers (`bankField` / `secondSource` / `secondField`) are flexible enough to build a nonsensical mapping (e.g., compare `narration` to `amount`) with no warning. There's no inline hint of what a valid pairing looks like and no guard against type‑mismatched comparisons.

**U8 — No risk warnings on dangerous configurations.** *[Med]*
You can silently disable the **duplicate‑UTR check** (a fraud/double‑credit control), set a write‑off materiality far above the fee tolerance, or push subset‑sum combo size arbitrarily high. A rules studio for money movement should flag: "You've disabled a control that prevents double‑crediting," or "Write‑off threshold (₹5,000) exceeds your dispute tolerance."

**U9 — No navigation from a rule to its results.** *[Med]*
Clicking a rule's "N matched" count doesn't filter the Matches tab to that rule's matches (or the Exceptions tab to what it failed to clear). The Rules Studio and the results tabs are disconnected, so cause→effect requires manual cross‑referencing.

**U10 — Reordering suggests more power than it has (see L1).** *[High — UX symptom of a logic gap]*
Up/down arrows and the copy ("reorder rules to see matching change live … first match wins in priority order") imply that Phase 2 evaluation order is user‑controlled. For most of Phase 2 it is **not** (the engine runs specialised allocation kinds in a fixed pipeline — see **L1**). The UI promises control the engine doesn't honour, which is worse than not offering it.

**U11 — Only up/down reordering; no drag‑and‑drop.** *[Low]* Fine for short lists; tedious as phases grow.

---

## 3. Logical gaps in the Rules Studio

> These are behavioural divergences between the engine (`arEngine` and its matchers) and the proposal's specified logic. References are to the proposal's rule numbers.

**L1 — Phase 2 allocation runs in a *fixed code pipeline*, not the user's priority order.** *[Critical]*
The Studio presents Phase 2 as a reorderable, first‑match‑wins cascade. In reality the engine dispatches by rule **kind** in a hardcoded sequence: single‑row kinds 2.1–2.4 (in priority order among themselves), then `subset‑sum` → `bank‑fee` → `write‑off` → `overpayment` → `partial‑payment`, each found by `allocRules.find(r => r.kind === …)`. So reordering (e.g.) *write‑off* above *exact invoice number* changes `priority` but **not** evaluation order. This breaks the core "first match wins in priority order" contract for most of Phase 2 and makes the reorder UI misleading.

**L2 — Subset‑sum (Rule 2.5) does not apply FIFO and never flags the ambiguous‑subset case.** *[Critical]*
The proposal mandates a FIFO (oldest‑due‑first) chronological waterfall and, *if multiple valid subsets exist for the same due dates*, a fail‑safe **"Ambiguous Match" → Exception Queue**. `findExactSumSubset` returns the **first** subset found by array index (size 2 upward) — no due‑date ordering is passed in, and it stops at the first hit, so it **cannot detect competing subsets**. Consequently subset‑sum can silently pick a non‑FIFO combination and will never raise the ambiguity exception the "never guess" principle requires.

**L3 — Duplicate‑UTR check is within‑file only, not historical.** *[High]*
Proposal's pre‑phase security check compares `bank_tx_ref` against **historical reconciled transaction logs**. `findDuplicateUTRs` only counts duplicates **within the current run's** `bankStatements`. A bank/gateway that resends a settlement file the next day (the exact edge case cited) would pass undetected because the first instance is in a prior run. Needs a persistent reconciled‑UTR ledger.

**L4 — Candidate‑pool tie‑break uses *exact amount only*.** *[High]*
Proposal Phase 2 resolves a 1B pool via "exact invoice matches **or** mathematical sums." The engine resolves pools solely through the `exact-amount` rule (`payments.filter(p => !p.customerId && p.candidatePool.length)…`). A pooled payment that a locked customer *could* have resolved via invoice‑number‑in‑narration or subset‑sum is left in Suspense. Narrower than spec.

**L5 — Phase 2.0(b) net‑off ignores memo dates and the period cutoff.** *[High]*
`effectiveBalance` nets `creditNotes`/`debitNotes` as **scalar fields on the invoice**, applied unconditionally. The proposal requires memos to be **separate dated documents filtered by `document_date ≤ period_end`**. A credit note dated after the cutoff would still reduce the balance here — the exact period‑leakage Phase 2.0 exists to prevent, in the other direction. Also there's no "full‑pay preference / leave CN parked on‑account" branch.

**L6 — Expected‑remittance is a single customer field, not a remittance‑communications table.** *[Med]*
Rule 1.1a matches against `customer.expected_utr` (one value per customer) rather than the schema's `expected_remittance_communications` table (many pending, dated, with `declared_amount`). It therefore can't handle multiple outstanding pre‑advices per customer, or use the declared amount/date as corroboration.

**L7 — TDS rate / `allowed_tds_amount` is not configurable and not clearly sourced.** *[Med]*
Rule 2.4 needs `payment == effective_balance − allowed_tds_amount`. The Studio exposes no field for the allowed TDS source or standard rate(s) (2% / 10%). Whether `matchTDS` reads a real per‑invoice `allowed_tds_amount` or infers a fixed rate should be made explicit and user‑governed; as shipped it's opaque.

**L8 — Standalone bank‑charge classification is a precomputed data flag, not a tunable rule.** *[Med]*
Standalone bank‑initiated fees (Exception §3) are detected via `bs.isBankCharge` set on the ingested row, then routed to a "Standalone Bank Charge" exception. There's no Rules‑Studio rule governing *what counts* as a standalone charge (narration patterns, zero‑customer rows), so a controller can't tune it, and the proposal's **direct GL JE** (Dr Bank Charges / Cr Cash Control) isn't posted — only flagged.

**L9 — `bank-fee` and `write-off` have identical matching shape and can shadow each other.** *[Med]*
Both are "payment short of invoice by `diff > 0 && diff ≤ tolerance`," differing only in intent/note. Because `bank-fee` runs before `write-off` in the fixed pipeline (see L1), an overlapping tolerance means the fee rule silently claims rows the user intended as write‑offs. No overlap detection or ordering control exists.

**L10 — Gateway‑fee decoupling (Stripe/Razorpay `gross/fee/gst_on_fee/net_settled`) is unimplemented.** *[Med]*
Explicit **bank** fee decoupling works (`bs.explicitFee`). The gateway‑settlement path (`gatewaySettlements`) has no sample data and no fee/GST decoupling logic — it only raises a generic "Gateway Variance" exception. Rule 2.6's gateway half is a stub.

**L11 — No multi‑currency normalization.** *[Med]*
Schema v6 carries `recon_amount` / `recon_balance_due` (INR‑normalised). The engine compares raw amounts and assumes a single currency; a USD invoice vs an INR receipt would mismatch. Out of the Studio's immediate scope but a real‑world blocker.

**L12 — Post‑reconciliation JE timing, SL↔GL proof export, and the cryptographic run hash / granular `old_state→new_state` audit are absent.** *[Med — out of Studio, in scope end‑to‑end]*
The GL control **variance** is computed and flagged, but no consolidated JEs are posted after exception resolution (Proposal §6), and the immutable audit trail lacks the `RUN‑…‑SHA256` cryptographic hash and per‑field before/after capture (Proposal §7 / schema table 11). The Reports "Audit Trails" tab is a good start but is event‑level, not tamper‑evident.

---

## 4. Priority recommendations

**Fix first (make the Studio honest & tunable):**
1. **L1 / U10** — Either make Phase 2 genuinely priority‑ordered, or change the UI to show Phase 2 as a *fixed pipeline* with per‑stage on/off (stop implying reordering works when it doesn't).
2. **L2** — Give `findExactSumSubset` FIFO ordering and multi‑subset detection → raise the Ambiguous Match exception.
3. **U1 + U2** — Add an aggregate match‑rate/breaks delta and a **near‑miss panel** per rule. This is what turns the Studio from a viewer into a tuning tool.
4. **U6 / L5** — Surface Phase 2.0 guardrails (period‑end date, dated credit/debit net‑off) as first‑class, editable controls.

**Fix next (correctness & trust):**
5. **L3** — Historical/cross‑run duplicate‑UTR ledger.
6. **L4** — Broaden candidate‑pool tie‑break to invoice‑number and subset‑sum.
7. **U4 / U5 / U8** — Allow rule delete + per‑rule revert; hide confidence where inert; warn on risky configs.

**Later (breadth):**
8. **L6, L7, L8, L10, L11, L12** — remittance‑communications table, configurable TDS, rule‑driven bank‑charge classification, gateway fee decoupling, multi‑currency, JE posting + cryptographic run hash.

---

## 5. One‑line takeaway

The prototype **proves the three‑tier workflow convincingly and is an excellent design/validation artifact**, but the Rules Studio currently *looks* more configurable than it *behaves* (fixed Phase‑2 pipeline, hidden guardrails, no near‑miss feedback) and the engine makes several "never guess"‑violating simplifications (subset‑sum FIFO/ambiguity, in‑file‑only dedup, unconditional memo net‑off). Closing L1, L2, U1/U2 and U6 would move it from "great demo" to "credible tuning surface."
