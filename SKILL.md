---
name: papertrail-filing-clerk
description: "PaperTrail filing clerk for a UGREEN AI NAS: paperwork dropped into an intake folder is read on the device, classified into a nine-category taxonomy, renamed, filed into a structured library, and recorded in a reversible audit manifest, and expiry and due-date digests are compiled from the filed records. Use when the user asks to file new documents, asks what expires or falls due soon, asks where a filed document went, or asks to undo a filing run."
---

# Goal

The user drops scans, receipts, invoices, contracts, and photos of paper
documents into one intake folder and never files them. PaperTrail turns
that pile into a tidy, searchable library: every document is read on the
device, classified into the nine-category taxonomy in
[references/taxonomy.md](references/taxonomy.md), renamed and filed under
the convention in
[references/naming-and-metadata.md](references/naming-and-metadata.md),
and recorded in an audit manifest that makes every filing reversible.
Upcoming expiry and due dates are compiled into a digest from the filed
records before they become problems. Anything the Skill is not confident
about waits in a Needs Review folder with its original name, untouched,
instead of being guessed at. The safety and permission model is in
[references/safety-and-permissions.md](references/safety-and-permissions.md).

Typical triggers:

- "File my documents" — run an intake pass over new files.
- "What expires soon?" — compile the date digest from filed records.
- "Where did that document go?" — answer from the audit manifest.
- "Undo the last filing run" — reverse one recorded run.

# Required input

Collect only what this parent workflow needs. Real directories are
resolved by the responsible file expert from the user scope; the parent
does not pre-collect private parameters owned by a child expert.

| Input | Default | Notes |
|---|---|---|
| Intake folder | none — user confirms | The folder the user drops documents into. Read-only for this Skill. |
| Library folder | none — user confirms | Filing destination, organised by category and year, holding the Needs Review folder and the audit manifest. |
| Confidence threshold | 0.80 | At or above it, a document is filed automatically. Below it, the document goes to Needs Review. |
| Alert lead time | 30 days | How far ahead expiry and due dates are surfaced in a digest. |

No credentials, accounts, or external services are required or accepted.

# Generation-time capability evidence

- `ug-filesystem` → `file-agent`
- `ug-filemgr` → `file-agent`
- `file-workflow` → `file-agent`
- `ug-document` → `document-agent`
- `ug-photo` → `photo-agent`
- `ug-knowledge` → `knowledge-agent`

Feasibility notes, confirmed against the public capability catalog
version 2.0.0 before writing this Skill:

- `ug-filesystem` publishes directory listing and search, file status,
  content hashing (`sha256sum`), reading (`cat`), appending records
  (`echo`), and the state-changing `mkdir`, `rename`, `cpmv`, and
  `restore` operations. Reads carry no approval; state changes carry
  gateway approval.
- `ug-filemgr` publishes duplicate scanning (`scan-dedupe`) and the tag
  read and write operations used to record category and metadata on
  filed documents. Tag writes carry gateway or conditional approval.
- `file-workflow` publishes user directory resolution (`get-user-dir`),
  workflow planning (`start`) and execution (`run`), and `batch-move`.
  Execution carries workflow approval.
- `ug-document` publishes text and field extraction (`extract`),
  document `metadata`, `page-count`, and `qa`. All are read-only.
- `ug-photo` publishes OCR (`vision-ocr`) and visual analysis
  (`vision-vlm`) for images. Both are read-only.
- `ug-knowledge` publishes evidence collection (`collect-evidence`) and
  text saving (`save-text`, which returns a verified saved-file
  reference). Saving carries workflow approval.

# Orchestration

This Skill is the planner. It delegates every NAS operation to the
single registered expert for the CLI that owns it, in the order below.
Downstream steps use only the real result returned by the upstream
delegation; values are never invented or assumed. Classification itself
is the planner reasoning over the real text and fields the reading
experts return: it assigns one taxonomy category with a confidence
score and applies the confidence gate before anything is filed.

1. Intake — collect the batch and detect duplicates (file-agent):

delegate --target file-agent --instruction 'Resolve the user document directories, then inspect the intake folder the user confirmed for this Skill. List every file in the intake folder with its real path, size and format, compute a SHA-256 hash for each file, and scan the filing library for files whose hash matches, so exact duplicates are known before any reading starts. Return the new files with their paths, hashes, sizes and formats, list the duplicates separately, and state plainly when the intake folder is empty or holds no new files.'

2. Reader, documents — extract text and fields (document-agent):

delegate --target document-agent --instruction 'Extract the full text and the structured fields from every document file in the list of real paths supplied with this instruction. For each file return the extracted text, the document metadata, the page count, the issuer or vendor, the document date, any total amount with its currency, and any due date or expiry date the document states. Mark a file as unreadable when its text cannot be extracted, and never invent content for such a file.'

3. Reader, images — OCR scans and photos (photo-agent). Used for image
   files, and for any item step 2 marked unreadable:

delegate --target photo-agent --instruction 'Run OCR on every scanned image and every photo of a paper document in the list of real paths supplied with this instruction, and return the recognised text for each image. When the OCR text is too sparse to identify the document, add a visual reading of the apparent document type and of any visible issuer, date and amount. Mark an image as unreadable when no usable text can be recognised, and never invent content for such an image.'

4. Classifier and gate — the planner assigns one category from
   [references/taxonomy.md](references/taxonomy.md) and extracts the
   metadata fields defined in
   [references/naming-and-metadata.md](references/naming-and-metadata.md)
   from the real results of steps 2 and 3, scores its confidence, and
   builds a filing plan: file at or above the confidence threshold,
   route below it (or when unreadable, or when extracted fields
   conflict) to Needs Review. There is no third outcome and no silent
   guessing.

5. Archivist — carry out the filing plan (file-agent). The same
   delegation, with a plan built from the user decision, also files
   items the user has confirmed or corrected in Needs Review:

delegate --target file-agent --instruction 'Carry out the filing plan supplied with this instruction. Create any missing category and year folders inside the library, rename each approved document to its planned library name, move it into its category folder, and record the category plus the extracted metadata as tags on the filed document. Move each document the plan marks for review into the Needs Review folder with its original name and content unchanged. Append one audit record per action to the audit manifest stored inside the library, covering the original path, the filed path, the hash, the category and the metadata. Never delete a file, never overwrite an existing library file, and on a name collision use the short hash suffix given in the plan.'

6. Sentinel — gather the date and digest facts (file-agent):

delegate --target file-agent --instruction 'Read the audit manifest stored inside the library together with the tags of the filed documents, and return every filed document whose due date or expiry date falls within 30 days of the run date, with its category, issuer, amount and the date itself. Also return the counts of filed, review and duplicate items and the entries of the most recent run, so a digest can be compiled from real records only.'

   The planner compiles the digest from that real result, using the
   user alert lead time when it differs from the 30-day default, and
   has it saved as the run digest record (knowledge-agent):

delegate --target knowledge-agent --instruction 'Save the digest text supplied with this instruction as the PaperTrail digest record for this run, and return the verified saved file reference. The digest contains only counts, categories, issuers, amounts and dates, and no document body text.'

7. Undo — reverse one recorded run (file-agent), only on user request:

delegate --target file-agent --instruction 'Reverse the filing run named in the manifest records supplied with this instruction. For each record of that run, move the filed document back to its original path under its original name, then append one undo record per restored file to the audit manifest. Leave any document the user has confirmed after review where it is, unless this instruction names that document explicitly. Never delete a file.'

State-changing delegations surface the catalog approval attached to
their operations (gateway approval for file changes, workflow approval
for workflow execution and text saving). While such a step is waiting
for approval the run pauses; nothing proceeds around it.

# Capability gaps and honest limits

Checked against catalog version 2.0.0. The following PaperTrail
behaviours have no matching published capability. They are stated here
as unsupported or planned rather than claimed:

- Standing intake watch. No published capability watches a folder or
  fires when new files arrive. A run starts when the user asks, or when
  the host runs this Skill. A true arrival trigger requires NAS
  confirmation.
- Scheduling. No published capability provides timed or recurring
  runs, so the weekly digest is a cadence the user invokes; the Skill
  keeps no clock of its own. Unsupported in the catalog.
- Notification delivery. No published capability delivers push
  notifications. Digests and alerts are compiled from real manifest
  records, saved as a digest record, and reported to the user in the
  session. Push delivery requires NAS confirmation.
- Taxonomy classification and confidence scores. No published Action
  assigns PaperTrail categories or confidence values. Classification is
  the planner reasoning over text and fields returned by the document
  and photo experts, and the result is recorded as tags on the filed
  document.
- A raw on-device language model endpoint. The catalog exposes
  document extraction and question answering (`ug-document`) and visual
  analysis (`ug-photo`), not a general model service. PaperTrail phrases
  all model work through those published capabilities.
- Handwriting and damaged scans lower OCR confidence and usually route
  to Needs Review. Accuracy depends on the on-device models of the NAS.

# Completion and failure

Delegation states are handled explicitly, and only a `completed`
delegation is ever reported as a business result:

- `completed` — the step result is used and reported: a run completes
  when every intake item is filed, routed to Needs Review, or recorded
  as a duplicate, the manifest holds a record for every action, and the
  digest has been compiled and saved.
- `failed` — the failure and the expert reason are reported as a
  failure, never as success. Partial results already returned are
  preserved in the manifest, and dependent steps stop.
- `cancelled` — the run stops. Actions already completed stay recorded
  in the audit manifest, so the partial state is exactly known and can
  be undone like any other run.
- `waiting_user` — the run pauses and the open question is surfaced,
  for example confirming the intake or library folder, or deciding a
  Needs Review item. Dependent steps wait for the answer.
- `waiting_approval` — the run pauses at the state-changing step that
  needs gateway or workflow approval and says which step it is.
  Nothing proceeds without that approval.

Empty and partial results are reported as they are. If an intake scan
returns an empty result, the run reports that there are no results to
file and ends; it does not widen the scope or switch to another folder.
If only some items could be read or filed, the digest lists the filed,
review, duplicate, and failed counts separately.

| Failure | Behaviour |
|---|---|
| Extraction fails or text is near-empty | Route the item to Needs Review as unreadable, record the reason, never guess a category. |
| Exact duplicate of an already filed document | Do not file it again; note it in the run summary. |
| Name collision in the library | Append the short hash suffix from the plan; never overwrite an existing file. |
| Password-protected or corrupt file | Leave it in the intake folder, flag it in the summary with the reason, do not retry in a loop. |
| A move or rename fails | Stop that item, keep the original in place, record the error in the manifest, surface it in the summary, and continue with the other items. |
| No usable reading from any expert | Route items to Needs Review on name and format signals only; never auto-file from heuristics. |

# Disclosures

- **Third-party components and open-source libraries:** none. PaperTrail
  uses only the published UGOS capabilities listed in the capability
  evidence section, through the registered experts. No external
  services, no external data processors, and therefore no third-party
  licenses apply.
- **Principal functionality developed for this Hackathon:** the
  end-to-end filing workflow, the expert orchestration, the
  nine-category taxonomy, the naming and metadata schema, the
  confidence-gate and review-queue model, the audit-manifest and undo
  design, and the expiry and due-date digest. All were newly developed
  during the Hackathon submission period.
- **AI tools used, and their principal uses:** Muse (Meta AI assistant,
  "Ember") assisted the entrant with workflow design and drafting of
  this SKILL.md, its references, and the proposal draft. The entrant
  wrote the final proposal answers in his own words and reviewed and
  approved all submitted material.

## Version

PaperTrail Filing Clerk v0.2 — Round 1 pre-test package, aligned to the
Uliya Skill Development Standard and capability catalog 2.0.0
(uliya-skill-kit 0.7.3) and checked locally with the bundled static
checker. A static check proves only that the files were written to the
standard; it does not prove upload, installation, compatibility, or
runtime execution on a NAS. Live binding and on-device validation are
the Round 2 step.
