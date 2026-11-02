# PaperTrail naming, metadata, and manifest formats

## Library layout

```
<library_root>/
  <Category>/<YYYY>/
  Needs Review/
  _papertrail/manifest.jsonl
  _papertrail/digests/<YYYY-MM-DD>.md
```

## File naming convention

`YYYY-MM-DD__Category__Vendor__Amount.ext`

- Date = document date (not filing date). Unknown date → `0000-00-00`.
- Vendor = issuer/merchant, spaces removed, unsafe characters stripped,
  max 32 chars. Unknown → `Unknown`.
- Amount = total with currency code, e.g. `USD123.45`. No amount → the
  segment is omitted rather than faked.
- Extension = original extension, lowercased. The original file content is
  never modified; filing changes name and location only.
- Collision → append `__h<first 6 of hash>`; never overwrite.

Example: `2026-10-14__Invoice__CityPower__USD86.20.pdf`

## Metadata record (stored in the manifest, one JSON object per action)

```json
{
  "run_id": "2026-11-02T08-00-00",
  "action": "file",
  "hash": "<sha256>",
  "original_path": "Intake/IMG_2041.pdf",
  "filed_path": "Library/Invoice/2026/2026-10-14__Invoice__CityPower__USD86.20.pdf",
  "category": "Invoice",
  "category_confidence": 0.93,
  "metadata": {
    "vendor": "CityPower",
    "document_date": "2026-10-14",
    "due_date": "2026-11-13",
    "expiry_date": null,
    "amount": 86.20,
    "currency": "USD"
  },
  "timestamp": "2026-11-02T08:00:11"
}
```

Review-queue and undo actions use the same record shape with
`action: "review"` or `action: "undo"`.

## Digest format (Markdown, saved as the run digest record and reported to the user)

```
PaperTrail digest — 2026-11-02
Filed: 6 · Needs review: 1 · Duplicates skipped: 2
Coming up: Warranty — HeatPump (expires 2026-11-28, in 26 days)
           Invoice — CityPower USD 86.20 (due 2026-11-13)
Needs review: 1 item in Library/Needs Review
```
