# New Bill — AI Accountant

A single-file, working prototype of the **New Bill** screen: create a purchase bill,
upload a supplier invoice, and let the system extract the line detail, work out which
item and ledger masters it belongs to — proposing the ones that do not exist yet — and
predict the GST and TDS treatment.

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

**Master matching.** The sample bill carries only what a supplier actually prints — their
wording, sometimes an HSN, the quantities and the rate. Which item or ledger master a line
belongs to is worked out, three ways with three standards of proof:

| | |
|---|---|
| **History** | this vendor has billed these words before and we know how it was posted. A precedent, not a guess, so it wins outright — and how many times carries the confidence |
| **Name** | the words resemble a master we hold. Scored as `0.6 × word overlap + 0.4 × character overlap`, with bonuses when the printed HSN agrees or the specifications match |
| **Neither** | nothing in the books resembles it, so the master does not exist yet and is proposed — which means predicting the HSN or SAC, the rate, the group, and for a ledger the **nature of payment**, so the TDS engine picks it up the moment you accept |

**Every one of those answers is applied.** A prediction that leaves the field empty is not a
prediction — it is a blank the user now has to fill without being told what to fill it with.
So the third case does not sit there asking: it mints the master it proposed, marked
provisional, and points the line at it. The field reads back what will be created, the GST
and TDS engines see it immediately, and the master becomes a real record only when the bill
posts. Discard the bill and it goes with it; point the line somewhere else and it is dropped.

Two thresholds say how much of a claim is being made. Above **0.82** the row says nothing —
the filled field is the whole message. Between **0.55 and 0.82** it is marked *Confirm*.
Below, it says what it invented: the field is dashed rather than solid and the mark reads
*New item* or *New ledger*. On a seven-line bill that is one word on three rows and nothing
anywhere else.

The mark opens a popover: the proposal with its reasoning and an editable name, any
candidates with their scores, and *Keep unmapped*. Predicted HSN and rate are stated there
but edited in the row's own HSN and Tax cells — one number editable in two places is one
place too many. Accepting anything writes a mapping, so the same words on the next bill
match on history rather than asking twice. Nothing blocks allocation; a line left unmapped
posts on its description and is named in the posting summary.

A proposed item is named in the vendor's words, because the wording *is* the product. A
proposed ledger is named for the kind of expense — `E-Waste Disposal & Recycling`, not
"…of the units replaced in July" — because that name belongs to the heading, and the heading
knows it.

It is a local heuristic over the tables in this file — `MASTERS.mappings` for the history and
`HSN_KB` for the classification. No model is called and no document leaves the page; the
scoring is arithmetic you can read.

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

The bundled sample is a ₹8,51,747.60 invoice of seven lines from a vendor already
₹49,20,000 into the financial year. Four lines match outright, one asks to be confirmed,
and two have no master — so two are proposed:

| | |
|---|---|
| Sub total | ₹7,21,820.00 |
| CGST 9% / SGST 9% | ₹64,963.80 each |
| **Invoice total** | **₹8,51,747.60** |
| TDS — 194C ₹1,270, 194J ₹1,920, 194Q ₹482.32 | −₹3,672.32 |
| **Net payable** | **₹8,48,075.28** |

The invoice totals ₹8,51,747.60; the payable is ₹8,48,075.28. The difference is the
predicted deduction — which is the whole point.

The 194C figure is where the proposal earns its keep. **E-Waste Disposal & Recycling** is
classified under SAC 999432, which carries 194C, so the ₹18,500 line joins the contractor
base the moment it is proposed: ₹45,000 becomes ₹63,500. Reject that ledger in the popover
and the deduction falls back to ₹3,302.32 in front of you. A proposed master is not
paperwork — it changes what the bill posts.

Discarding or starting a new bill removes any masters created for it, so the next run
begins where the last one did.

## Notes

- Fluid layout down to mobile; wide tables scroll inside their own container rather than
  the page. The document pane is 50% wide and drag-resizable (25–70%, double-click to
  reset, arrow keys when focused).
- All icons are the exported Figma assets, inlined as an SVG sprite.
- Master data, the sample bill and the tax logic are all near the top of the `<script>`
  block and are meant to be edited. So are `MASTERS.mappings`, `HSN_KB` and the `MATCH`
  thresholds, which sit with them — moving a threshold changes what the demo does.
