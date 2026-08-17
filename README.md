# Accounts Payable — AI Accountant

The accounts payable prototype. One screen so far, **New Bill**: create a purchase
bill, upload a supplier invoice, and let the system extract the line detail into the
item and ledger tables and work out the GST and TDS treatment.

Built from the Karbon — AI Accountant Figma frame `23201:32379`, then made functional.
Everything lives in `index.html` — no build step, no dependencies. Open it in a browser.

```
open index.html
```

## Try it

Click **Upload Bill** and pick any file, drop a file on the left pane, or click
**use a sample bill** to skip the dialog. A five-stage pipeline runs — upload, OCR,
field extraction, reading the item and ledger detail, tax computation — and the
form fills in.

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

**Line detail.** The extraction fills each line's description, HSN, quantity, rate,
discount and cost centre, and puts it in the table it belongs in — goods in **Item
Details**, services and charges in **Ledgers**. Where the bill names a master we hold,
the Item or Ledger cell arrives filled and brings its unit, godown and GST rate with
it. Where it names none, the cell arrives empty and you pick one; the row still posts
on its description in the meantime, and the posting summary names how many were left.

Nothing is inferred, scored or proposed. A line's master is either read off the
document or chosen by hand.

**Tax rates are carried, not chosen per row.** The rate comes from the item master or
the expense ledger the line posts to, so there is no Tax column in either table — the
GST breakdown below aggregates whatever the lines carry.

**TDS.** Worked out here, never read from the document — a supplier invoice does not
carry the buyer's deduction. It is derived from the expense ledger's nature of payment, the
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

The bundled sample is a ₹8,51,747.60 invoice of seven lines from a vendor already
₹49,20,000 into the financial year. Four land in **Item Details** and three in
**Ledgers**; two of the seven name no master and arrive with the cell empty:

| | |
|---|---|
| Sub total | ₹7,21,820.00 |
| CGST 9% / SGST 9% | ₹64,963.80 each |
| **Invoice total** | **₹8,51,747.60** |
| TDS — 194C ₹900, 194J ₹1,920, 194Q ₹482.32 | −₹3,302.32 |
| **Net payable** | **₹8,48,445.28** |

The invoice totals ₹8,51,747.60; the payable is ₹8,48,445.28. The difference is the
deduction the buyer withholds — which is the whole point.

The two lines with no master are where the ledger choice shows its weight. The ₹18,500
e-waste line posts to *Unallocated expense* and contributes no TDS until you name a
ledger for it; point it at a contracted-work ledger and the 194C base grows from
₹45,000 to ₹63,500, with the deduction following in front of you.

## Notes

- Fluid layout down to mobile; wide tables scroll inside their own container rather than
  the page. The document pane is 50% wide and drag-resizable (25–70%, double-click to
  reset, arrow keys when focused).
- All icons are the exported Figma assets, inlined as an SVG sprite.
- Master data, the sample bill and the tax logic are all near the top of the `<script>`
  block and are meant to be edited. Each line in `SAMPLE.lines` carries a `route` and an
  `item` or `ledger` id — that is what decides where the extraction puts it; leave the id
  empty and the row arrives asking for one.
