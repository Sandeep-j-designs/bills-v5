# New Bill — AI Accountant

A single-file, working prototype of the **New Bill** screen: create a purchase bill,
upload a supplier invoice, and let the system extract the line detail and predict the
GST and TDS treatment.

Built from the Karbon — AI Accountant Figma frame `23201:32379`, then made functional.
Everything lives in `index.html` — no build step, no dependencies. Open it in a browser.

```
open index.html
```

## Try it

Click **Upload Bill** and pick any file, drop a file on the left pane, or click
**use a sample bill** to skip the dialog. A five-stage pipeline runs — upload, OCR,
field extraction, master matching, tax computation — and the form fills in.

> **The extraction is simulated.** No file leaves your machine and no OCR runs. The
> pipeline is timed against a bundled sample bill so the flow works offline. Drop a real
> PDF or image and you will see its actual preview; otherwise a facsimile invoice is
> rendered from the same figures the form is using, so document and form always agree.

## What it actually does

**GST.** Source state vs destination state decides the split: same state gives CGST +
SGST at half the rate each, different states give IGST at the full rate. Taxable value
is aggregated per slab, so a bill mixing 12% and 18% produces separate ledger rows per
slab. GST treatment matters — an unregistered supplier switches the bill to reverse
charge, composition and overseas suppliers attract no GST.

**TDS.** Predicted, never read from the document — a supplier invoice does not carry the
buyer's deduction. It is derived from the expense ledger's nature of payment, the
vendor's PAN and deductee type, and their year-to-date figures:

| Section | Trigger |
|---|---|
| 194C | single payment ≥ ₹30,000 or ₹1,00,000 in the year |
| 194J | ₹30,000 FY aggregate; 2% technical, 10% professional |
| 194I | rent, ₹2,40,000 FY aggregate |
| 194Q | goods above the ₹50,00,000 FY threshold, on the excess only |

Deduction is on taxable value with GST excluded (CBDT circular 23/2017). No PAN on file
forces 20% under s.206AA. GST and TDS share one Taxes table, because a TDS deduction is
just another tax line on the bill.

**Everything is overridable.** Line amounts, tax rates, tax amounts, the voucher number,
the vendor's address, GSTIN and place of supply. An auto-computed row that you edit is
pinned and will not be recalculated over; untouched rows keep following the engine.

**On allocation** the bill gets a voucher number (`PUR/26-27/185` — prefix, Indian FY,
sequence) and posts a double entry: purchase and expense debits, input GST debits, TDS
payable credits per section, round off, and the net vendor credit — with a balance check.
ITC marked ineligible is charged to expense rather than to the input credit ledger.

## Worked example

The bundled sample is a ₹7,77,525.60 invoice from a vendor already ₹49,20,000 into the
financial year:

| | |
|---|---|
| Sub total | ₹6,58,920.00 |
| CGST 9% / SGST 9% | ₹59,302.80 each |
| TDS — 194C, 194J, 194Q | −₹3,257.92 |
| Round off | ₹0.32 |
| **Grand total** | **₹7,74,268.00** |

The invoice totals ₹7,77,525.60; the payable is ₹7,74,268.00. The difference is the
predicted deduction — which is the whole point.

## Notes

- Fluid layout down to mobile; wide tables scroll inside their own container rather than
  the page. The document pane is 50% wide and drag-resizable (25–70%, double-click to
  reset, arrow keys when focused).
- All icons are the exported Figma assets, inlined as an SVG sprite.
- Master data, the sample bill and the tax logic are all near the top of the `<script>`
  block and are meant to be edited.
