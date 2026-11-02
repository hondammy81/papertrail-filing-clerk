# PaperTrail taxonomy

Nine categories. Every filed document gets exactly one. Signals are the
cues the Classifier Agent weighs; examples are illustrative placeholders.

| # | Category | Typical signals | Extracted metadata emphasis |
|---|---|---|---|
| 1 | Invoice | "Invoice", invoice number, line items, total due, payment terms | issuer, invoice date, due date, total, currency |
| 2 | Receipt | "Receipt", card last digits, itemized purchases, tax line | merchant, date, total, currency |
| 3 | Contract | signatures, parties, term/effective dates, "Agreement" | parties, effective date, end/renewal date |
| 4 | Warranty & Manual | "Warranty", model/serial numbers, coverage period, product guides | product, purchase date, warranty expiry |
| 5 | Tax | tax authority wording, tax year, forms (e.g. W-2, 1099), assessments | authority, tax year, amount owed/refund |
| 6 | Insurance | policy number, coverage, premium, renewal date | insurer, policy number, renewal date, premium |
| 7 | Medical | provider/clinic names, patient statements, explanation of benefits | provider, service date, amount |
| 8 | Personal & ID | certificates, licences, official letters, school records | issuer, document date, expiry if any |
| 9 | Other | none of the above signals are strong | best-effort vendor/date; usually Needs Review |

Rules:
- One category per document — the strongest signal set wins; ties lower the
  confidence score and usually route the item to Needs Review.
- A document whose primary purpose is a payment demand is an Invoice even
  if it also looks like a statement; a proof of a completed payment is a
  Receipt.
- Category names are fixed. Users may add subfolders under a category in
  the library tree, but PaperTrail never invents new top-level categories.
