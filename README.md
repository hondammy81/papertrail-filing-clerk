# PaperTrail Filing Clerk

A skill for UGREEN AI NAS, built for the **UGREEN Hackathon 2026 on AI NAS** (Round 1, Automate Your Workflow track). Entrant: Dammy Okunlola (solo).

## What it does

PaperTrail is a direct record of the facts. It reads with AI on the device itself and exists in the NAS, and it can see when folders receive new documents. It then sorts and renames the files, organizes them, and notifies about renewals, warranties, and past due dates too.

## Layout

- `SKILL.md` — the skill itself: goal, workflow, orchestration, and the safety and completion rules for the UGREEN skill runtime.
- `references/taxonomy.md` — the nine categories every filed document is classified into, and the signals for each.
- `references/naming-and-metadata.md` — the library layout, the file naming convention, the metadata, and the reversible audit manifest format.
- `references/safety-and-permissions.md` — the safety and permission model: what the skill may read and write, and what it never deletes.

## Round 1 status

This repository holds the Round 1 source: the skill package as submitted with the Round 1 proposal. The package passes the official checker in the Uliya Skill Kit with a static-pass — 0 errors, 0 warnings. If the project is selected as a finalist, live testing happens on an assigned UGREEN AI NAS during the finalist testing window, and the skill is refined from those results for Round 2.

## License

MIT — see [LICENSE](LICENSE).
