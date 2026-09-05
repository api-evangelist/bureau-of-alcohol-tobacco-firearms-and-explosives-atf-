---
name: track-atf-cfr-amendment
description: >-
  Find what changed in an ATF firearms or explosives regulation and cite the Federal
  Register notice that changed it. Use when someone asks what 27 CFR 478/479/555/447/646/771
  says now, when it last changed, or what a specific rulemaking amended.
api: ATF eRegulations API
base_url: https://regulations.atf.gov/api
auth: none
operations:
  - listPartVersions
  - getRegulationTree
  - getNotice
  - getVersionDiff
---

# Track an ATF CFR amendment

ATF's eRegulations platform identifies each state of a regulation by the **Federal Register
document number that produced it**. That single fact makes this flow work: the version id and the
notice id are the same string, so a text lookup and a "why did this change" lookup join for free.

## Steps

1. **Pick the part.** Only six exist: `447` (importation of arms), `478` (commerce in firearms and
   ammunition), `479` (machine guns, destructive devices, NFA firearms), `555` (commerce in
   explosives), `646` (contraband cigarettes), `771` (rules of practice in explosive licence
   proceedings). Anything else is not in this system.

2. **List its versions** — `listPartVersions`:
   `GET /api/regulation/478`
   Returns `versions[]` of `{version, by_date, regulation}`, newest first. `by_date` is the
   effective date. If the newest `by_date` is years old, say so to the user rather than implying
   currency — 27 CFR 646 has not been reloaded since 2014-08-11 while 478 is current to 2026-01-22.

3. **Get the text** — `getRegulationTree`:
   `GET /api/regulation/478/2026-01141`
   Returns a recursive node tree. Each node has `label` (an ARRAY, e.g. `["478","11"]`), `text`,
   `node_type`, `title`, `lft` and `children`. Walk `children` and sort by `lft` to recover
   document order. To quote § 478.11, find the node whose `label` joins to `478-11`.

4. **Explain the change** — `getNotice`, using the version id as the document number:
   `GET /api/notice/2026-01141`
   Returns `title`, `publication_date`, `effective_on`, `fr_url`, `dockets`,
   `regulation_id_numbers` and — the part that matters — `amendments[]`, each with the verbatim
   `instruction` text and the `authority` citation. Quote the instruction; do not paraphrase a
   legal amendment.

5. **Diff two versions** if asked — `getVersionDiff`:
   `GET /api/diff/478/2025-04872/2026-01141`
   Returns a node-keyed map. An **empty object `{}` means no diff is stored**, not that nothing
   changed. Fall back to step 4 and compare the amendment instructions.

## Rules

- **Branch on Content-Type, never on the status code.** `/api/regulation/999` returns HTTP **200**
  with an HTML web page. So does an unknown notice id. Treat a response as data only when
  `Content-Type` starts with `application/json`; otherwise report "not found" — never hand the
  HTML to a model as regulatory text.
- **Always cite `fr_url`.** It is the canonical federalregister.gov URL for the notice. A regulatory
  answer without its citation is not usable.
- **Never assert the current law from a stale version.** State the `by_date` alongside any quote.
- No authentication is needed and no rate limit is published; still cache, because this text changes
  at Federal Register cadence, not at request cadence.
