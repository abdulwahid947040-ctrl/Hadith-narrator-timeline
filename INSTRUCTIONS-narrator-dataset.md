# Narrator Dataset — Verification & Upload Instructions

**Purpose.** This file documents (a) how to reassemble and upload the split archives,
and (b) the verification rules, known bugs, and open items extracted from the
Claude working session, so work can continue on the chronological narrator dataset.

> Source of section 2: the shared Claude conversation
> <https://claude.ai/share/58b69639-c7e0-4dbd-8a93-a7e6d3b34bf6>.
> Content there is a snapshot and may contain unverified material.

---

## 1. The split archives (25 MB upload limit)

Two source zips exceed the upload ceiling. Each has been split into fixed **20 MiB
parts (max part = 20,971,520 bytes ≈ 21.0 MB)** to stay safely under a 25 MB limit.
Splitting is **lossless byte concatenation** — no recompression, no entry is
altered, nothing dropped.

| Archive | Original size | Parts | Part naming |
|---|---|---|---|
| `Itqan-1.8.0.zip` | 62,522,804 B (62.5 MB) | 3 | `Itqan-1.8.0.zip.part01..03` |
| `narrator-dataset.zip` | 43,236,763 B (43.2 MB) | 3 | `narrator-dataset.zip.part01..03` |

### Reassemble (Linux / macOS)

```bash
cat Itqan-1.8.0.zip.part* > Itqan-1.8.0.zip
cat narrator-dataset.zip.part* > narrator-dataset.zip
```

The `part*` glob depends on zero-padded, ascending names, so it orders correctly.
If your shell sorts oddly, list them explicitly: `part01 part02 part03`.

### Reassemble (Windows PowerShell)

```powershell
cmd /c copy /b Itqan-1.8.0.zip.part01+Itqan-1.8.0.zip.part02+Itqan-1.8.0.zip.part03 Itqan-1.8.0.zip
cmd /c copy /b narrator-dataset.zip.part01+narrator-dataset.zip.part02+narrator-dataset.zip.part03 narrator-dataset.zip
```

### Verify the reassembled zips

```bash
sha256sum Itqan-1.8.0.zip narrator-dataset.zip
```

Expected (must match exactly):

```
Itqan-1.8.0.zip       c4d946548f0fa3fa611a742b61a3de71be1d6fbdf7ecfb4cd0dc1569d86c6050
narrator-dataset.zip  a51a041467f301a162d16c6933bd3732255d55a01aa7c93791354028bdc5a422
```

Integrity also confirmed by extracting and CRC-checking every entry
(`Itqan`: 3693 entries; `narrator-dataset`: 6 entries; all CRCs OK).

### Verify individual parts

Use `split_zips/SHA256SUMS-parts.txt`:

```bash
sha256sum -c SHA256SUMS-parts.txt
```

Per-part digests:

```
e6e454dc023b428658721c6dde491585443476276454545405f53fa8f3dd79c4  Itqan-1.8.0.zip.part01
33775a165ec677a49b5849f75a525e78d53550420ba7463cb484cbef17d0f481  Itqan-1.8.0.zip.part02
cf737ffbd54f877ea531cdd297ee171402731ddcf75d9d25fb31121e01019e44  Itqan-1.8.0.zip.part03
cfaf275cc0cb7d009adedc1733e64eef51e678111c3cb7aac3cf1ca2f7730183  narrator-dataset.zip.part01
22c71f15840eaf8091bfcc6eb61df1b8861c71a39d91574fdd1da75c3e57afdc  narrator-dataset.zip.part02
370dc6488aa475fbd7268e33e31ca3887e6a126c0e40f1860f9232a28051567b  narrator-dataset.zip.part03
```

### Format note

The zips use **LZMA (ZIP method 14)**, so they open natively in 7-Zip, WinRAR,
PeaZip, Python, and macOS Archive Utility, but **not** older Windows Explorer.
After reassembly, any tool that reads LZMA-zip will extract them normally.

---

## 2. Verification rules (from the Claude session)

### 2.1 The three accepted statuses

Only three statuses may enter the dataset. Nothing enters unlabeled.

1. **exact** — a single, directly cited death year.
2. **disputed** — multiple cited years. The **most verified** value is the primary
   field; the alternates go in a detail field.
3. **extended** — a range or "until X". The **earliest** year is the primary field;
   the full range is stated in the detail field.

Entries that cannot be confirmed/disputed/extended are **excluded**, never guessed.

### 2.2 Precision labels observed (v10 baseline)

| precision | count |
|---|---|
| exact | 34,043 |
| disputed | 4,102 |
| word-form | 1,795 |
| extended | 305 |
| hf_recovered | 16 |
| digit_form_unconfirmed_possible_truncation | 15 |
| externally_verified | 6 |
| century_elided_corrected | 5 |

Note: this arithmetic totals 40,287, not 41,354 — the missing 1,067 is the
unlabeled group below.

### 2.3 Provenance / sourcing rules

- Do **not** trust Wikipedia. Verify against book-derived sources only
  (hadith.com, الجامع / aljam3.com, shamela.ws, trajm, sunnah).
- Prefer sources enriched with primary-text data. For a specific passage, pull the
  exact book page (e.g. al-Muntazam vol. 3 p. 83) rather than a summary.
- **Shia entries are allowed** — do not auto-remove them (a later instruction
  reversed the earlier "Sunni-only" rule). Do correct status for known Shia figures.

---

## 3. Known bugs found and their logic (apply dataset-wide)

Fault-finding must identify **why** a year is wrong and then apply the same
correction to every case of that pattern.

1. **HF cross-reference ~34% error rate.**
   The `hf_exact_crossref_correction` (v5) step linked the wrong person by name in
   ~260 of 763 checkable records (e.g. raw "91 or 100 AH" → field 392).
   Itqan's *native* parser is clean. Fix: revert to Itqan's own text; keep bug
   contained to the HF step.

2. **Zero-year placeholder.**
   24 records had `death_year_h: 0` tagged "exact"; raw text was literally "0 هـ".
   No year 0 exists in the Hijri calendar. Fix: move out of exact → flagged/excluded.

3. **Compound "and two-hundred" (ومائتين) truncated.**
   "twenty-six and two-hundred" (226) parsed as just 26 — exactly 200 too low.
   Also found with the informal spelling **ومائه**. Fix: sweep all magnitudes
   (100/200/300/400/500) and both spellings; correct +200 / +100 / +300 as found.
   The truncation exists **in Itqan's raw source text itself**, not only our parser.

4. **Disputed primary ignored explicit preference.**
   When text says "X, or Y, and the correct one is Y" (الصحيح/الصواب), the parser
   used the *first* value. Fix: prefer the explicitly-flagged correct value.

5. **"Siyar-only source" records all sitting at a suspicious year (e.g. 4 AH).**
   7 records sourced only from Siyar's index shared the same wrong value.
   Fix: flag/correct; use the record's own `death_gregorian` field as a cross-check
   (e.g. al-Tirmidhi's raw Gregorian "892" reveals 279 AH vs the wrong "4 هـ").

6. **Name-collision false positives (methodological, affects every source).**
   Arabic descendants are named after distant ancestors, so name matching alone
   yields false pairs centuries apart (1,702 of 2,763 hawramani "matches").
   Fix: require the match to cover most of the *full* name, not appear as a
   substring of a longer name; then confirm era/genuinely.

7. **Grading vs era inconsistency.**
   Jarh-wa-ta'dil grading essentially did not exist before the 2nd century AH, so a
   graded (reliable/weak/fabricator) entry at a very early year is suspect.
   Keep the ~500 such records as a **watchlist**, verify individually; do not bulk-fix
   on weak evidence.

### 3.1 Example corrections already made

| Record | Was | Corrected to | Basis |
|---|---|---|---|
| Imam al-Bukhari | 167 | 256 AH | famous-figure check |
| Al-Darimi | 50 | 255 AH | famous-figure check |
| Abu Hatim al-Razi | 77 | 277 AH | century elision |
| Imam Muslim | 219 | 261 AH | OpenITI/KITAB |
| Muhammad al-Shaybani | 233 | 189 AH | OpenITI/KITAB |
| Khalifa ibn Khayyat | 40 | 240 AH | OpenITI/KITAB |
| Al-Bayhaqi | 145 | 458 AH | OpenITI/KITAB |
| Abu Dawud al-Sijistani | — | added | hawramani + OpenITI agreement |

---

## 4. Open items

- **1,058 records with no precision label** (1,017 blank Itqan text, HF-only;
  suggested label `hf_only_low_confidence`) — need a real category, not silence.
- **9 `inferred-from-event`** entries — decide whether they count as confirmed.
- **783 excluded era-only records** ("in the caliphate of Uthman"), no digits.
  Resolving = per-person historical research; a project, not a sweep.
- **726 same-era multi-source alternates** — candidates to fold into `disputed`.
- **~500 graded-at-early-year watchlist** — verify individually.
- **1,283 pending-review** backlog remaining.

---

## 5. Sources to use

- **Itqan** — <https://github.com/R3GENESI5/Itqan> (Zenodo DOI). Builds rijal DB from
  KASHAF (17,093), AR-Sanad 280K (18,298, has birth/death + Ibn Hajar & al-Dhahabi
  ranks), hatemben/hadithdb (1,524), and 22 OpenITI-parsed rijal texts.
- **AR-Sanad 280K** — <https://github.com/somaia02/Narrator-Disambiguatio>
- **hawramani (hadithtransmitters)** — 100,915 narrators, classical-text excerpts
- **Muslimscholars.info** — 25,247 scholar profiles
- **OpenITI / KITAB** — `OpenITI/kitab-metadata-automation` (5,419 authors w/ death years);
  <https://kitab-project.org/metadata/>
- **الجامع** — <https://aljam3.com> (63,500-book library; use direct book-page URLs)
- **shamela.ws** — browsable Ikmal Tahdhib al-Kamal, etc.
- **emadjumaah/hadith-kg** — 49,819 narrators

---

## 6. Working rules recap

1. Confirm → disputed → extended only. No guessing; exclude what can't be sourced.
2. Find the *reason* a value is wrong, then fix every case of that pattern.
3. Never bulk-apply on weak evidence; flag instead.
4. Guard against name-collision false positives.
5. Combined-number and placeholder bugs must be swept dataset-wide, both spellings.
6. Do not trust Wikipedia; use book-derived sources.
7. Shia entries stay in.
