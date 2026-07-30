# AR Reconciliation — Full Rule Coverage Test Data

Upload `Customers.csv`, `SL.csv`, `GL.csv`, and `Bank_Statement.csv` (in that order) via Data Hub, then run the AR Reconciliation. Every rule currently implemented fires at least once; verified end-to-end.

## Customer Identification (Phase 1A)

| Rule | Customer | How it fires |
|---|---|---|
| Duplicate transaction reference check | Some Vendor Co | BANK-015 and BANK-016 share reference `UTR-DUPX-015` |
| Expected UTR match | Acme Industries (CUST-001) | BANK-001's reference is exactly `UTR-ADV-7001` |
| Payer account + IFSC match | Bright Textiles (CUST-002) | BANK-002/003/004/014 all carry the customer's exact account + IFSC |
| UPI handle match | Nimbus Traders (CUST-003) | BANK-005/006 narration contains `nimbus@okhdfc` |
| Customer code in narration | Kestrel Freight (CUST-004) | BANK-007/008 narration contains `KEST04` |
| GSTIN / PAN extraction | Solace Pharma (CUST-005) | BANK-009 narration contains the customer's GSTIN |
| Fuzzy company name match | Vantage Retail Solutions (CUST-006) | BANK-010 payer name is `Vantage Retail Solution` (one letter off) |

## Candidate Pool (Phase 1B — only reached if 1A misses)

| Rule | Customer | How it fires |
|---|---|---|
| Masked account suffix match | Meridian Freight (CUST-008) | BANK-012's account ends `5544`, same as the customer's, full account differs |
| Token-based narration match | Halcyon Foods (CUST-007) | BANK-011 narration contains the token `HALCYON`; payer name is unrelated |

## Scoped Invoice Allocation (Phase 2)

| Rule | Invoice(s) | How it fires |
|---|---|---|
| Exact invoice number in narration | INV-2026-105, INV-2026-107 | Full invoice number present in BANK-005 / BANK-008 narration |
| Invoice suffix / truncated number | INV-2026-1046 | Only the last 4 digits (`1046`) appear in BANK-007 narration |
| Exact amount match | INV-101, INV-111, INV-112, INV-113 | Payment exactly equals the open balance |
| TDS match | INV-102 (10% TDS) | BANK-002 pays ₹13,500 = ₹15,000 − 10% |
| Subset sum (many-to-many) | INV-109 + INV-110 | One payment (BANK-009, ₹12,000) exactly covers both |
| Bank fee / minor variance | INV-104 | BANK-004 is short by exactly its `explicit_fee` (₹20) |
| Small balance write-off | INV-118 | BANK-014 is short by ₹2 — within the ₹5 materiality threshold |
| Overpayment → On Account | INV-103 | BANK-003 pays ₹9,500 against a ₹9,000 balance |
| Universal partial payment | INV-117 | BANK-006 pays ₹2,500 against a ₹4,000 balance, no other rule claims it |

## Exceptions

| Type | Source |
|---|---|
| Duplicate Transaction | BANK-015 / BANK-016 (shared reference) |
| Standalone Bank Charge | BANK-017, `is_bank_charge = true` |
| Suspense | BANK-018 — payer matches no customer by any signal |
| Scoped Exception | BANK-020 vs. Coral Living's two identical ₹4,500 invoices (INV-120/121) — ambiguous, never guessed |
| Double Collision | BANK-013 — account suffix `7788` matches both Silverline Traders (CUST-009) and Silverline Exports (CUST-010), each with an open ₹18,000 invoice |
| Short-Pay | INV-107 (Kestrel, short by ₹2,000), INV-117 (Nimbus's partial payment) |
| No Payment Received | INV-108 (Kestrel) and any invoice left unclaimed by the Scoped Exception / Double Collision above |
| GL Control Mismatch | `GL.csv` control balance (₹50,000) deliberately doesn't match the computed sub-ledger balance |

## Notes

- `explicit_fee` and `is_bank_charge` are new optional columns on Bank Statement uploads — added specifically so this file could exercise Standalone Bank Charge and the explicit-fee path of Bank fee / minor variance via CSV (previously only possible in the hand-authored seed data).
- Rules Studio settings (match mode, source/field selection, confidence thresholds) are all at their defaults for this run. Changing them will change which rows land where — that's expected.
