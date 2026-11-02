# PaperTrail safety and permissions model

## Principles

1. **Nothing leaves the NAS.** OCR, classification, and extraction run on
   the device's own models. PaperTrail configures no external endpoints
   and requires no API keys or accounts.
2. **No deletion, ever.** Filing is a move. The only destructive-sounding
   operation a user can request — undo — is itself a move back, and it is
   logged like everything else.
3. **No silent guessing.** Confidence gates every automatic action.
   Anything uncertain waits for the user in Needs Review, with its
   original filename intact.
4. **Everything is auditable.** Every action (file, review, undo,
   duplicate-skip, failure) appends one record to
   `_papertrail/manifest.jsonl` before the run reports success.

## Permission scope

| Scope | Access | Why |
|---|---|---|
| `intake_path` | read, list | Find new documents; originals stay put until filed. |
| `library_root` (incl. `Needs Review/`, `_papertrail/`) | read, write, move within | Filing, digests, manifest. |
| Everything else on the NAS | none | PaperTrail has no reason to touch it. |

The Skill requests only these scopes at install time and refuses to run
against paths outside them (e.g. a user-supplied `library_root` of `/`).

## Data handling

- Document text exists in memory only for the current item's pipeline
  steps; it is not copied anywhere except the filed document itself.
- Extracted metadata lives in the manifest, inside the library the user
  owns — the same trust boundary as the documents themselves.
- Digests contain counts, categories, vendors, and dates — never document
  body text. A digest is saved inside the library as the run digest
  record and reported to the user in the session. The public capability
  catalog contains no notification delivery capability, so PaperTrail
  does not claim push delivery of digests or alerts.

## Undo

"Undo last run" (or a chosen `run_id`) replays that run's manifest records
in reverse: each filed document returns to its original path and name.
Undo never crosses a user review decision: items the user has since
confirmed stay filed unless the user names them explicitly.
