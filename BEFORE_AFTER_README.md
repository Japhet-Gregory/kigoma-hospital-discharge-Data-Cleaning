# Before & After — Kigoma Hospital Discharge Dataset (January)

This document shows the raw → cleaned transformation for every column, using real (de-identified) records from the dataset. Patient names have been removed from both versions shown here — see [Privacy note](#privacy-note) at the bottom.

**Raw:** 907 rows × 17 columns → **Cleaned:** 905 rows × 13 columns

---

## Structural changes (before any column-level cleaning)

| Change | Detail |
|---|---|
| Removed 1 exact duplicate row | Same patient, same admission, identical in every field |
| Removed 1 test record | `Patient Name = "Test"`, District = `Temeke`, Region = `Dar es salaam` — a placeholder entry, not a real patient. Its removal is also why `Temeke` and `Dar es salaam` no longer appear anywhere in the cleaned dataset. |
| Removed 5 identifying columns | `Admitted By Doctor`, `Admitted By`, `Discharged By`, `Discharged By Doctor`, `Next of Kin` — real staff names and a broken nested field, not needed for ward/district/outcome analysis |
| Consolidated 5 ward sheets → 1 | One sheet turned out to be a full hospital-wide master export, not a single ward — used as the sole source instead of appending duplicates |

---

## Column-by-column: before → after

### Age
| Patient ID | Raw `Age` | Cleaned `Age (Yrs)` |
|---|---|---|
| 103251 | `11 months` | `0.9166...` *(displays as `11/12`)* |
| 61937 | `77 years` | `77.0` |

Months converted to a fraction of a year rather than a rounded decimal, so infant ages stay precise. A custom display format (`# ??/12`) keeps the fraction denominator fixed at 12, so e.g. `9/12` doesn't silently simplify to `3/4` and lose its "months" meaning.

**Bug caught via raw cross-check:** an earlier version of the cleaned file had `Age` shifted by one row for the last ~50 patients — each affected row showed the *previous* patient's age. This was invisible to null/format checks since the column was internally consistent, just misaligned. Caught only by validating against the raw source, matched on Patient ID + Date Discharge. Fixed by rebuilding `Age` directly from raw for every row.

### District
| Raw value | Cleaned value |
|---|---|
| `Manisipaa ya kigoma ujiji` *(hidden whitespace/characters)* | `Kigoma Ujiji Mc` |
| `Buhingwe` / `Buhigwe` *(two spellings, same district)* | `Buhigwe` |

`KG` and `KGG` were deliberately left unmapped — best guess is they're abbreviations for `Kigoma Ujiji Mc`, but this wasn't confirmed against the source system, so they weren't merged.

### Ward
| Raw value | Cleaned value |
|---|---|
| `NEONATAL WARD (WARD 2B)` | `Neonatal` |
| `MALE SURGICAL & ORTHOPEDIC (WARD) (7)` | `Male Surg & Orthopedic` |
| `MALE MEDICAL WARD (WARD 2A)` | `Male Medical` |
| `ICU` | `ICU` *(kept as a proper acronym, not lowercased)* |

11 verbose raw labels standardized into short, consistent categories via substring matching, with more specific terms checked before broader ones to avoid mismatches (e.g. "FEMALE SURGICAL" before "FEMALE WARD").

### Date Admitted / Date Discharge
| Patient ID | Raw value | Cleaned value |
|---|---|---|
| 103251 | `2025-04-16 23:30:10` *(unformatted serial)* | `2025-04-16 23:30:10` *(proper timestamp)* |
| 127430 | `46029.0217...` *(raw Excel serial, inconsistent with rest of column)* | `2026-01-07 00:31:17` |

Two rows had `Date Admitted` stored as raw numeric serials while every other row was text — an earlier processing step misconverted these into `1970-01-01`. The true timestamps were recovered directly from the raw serial values (not estimated), then verified against `Admission Days` with zero mismatches.

### Admission Days
Recomputed `Date Discharge − Date Admitted` for all 905 rows and compared against the stored value. **0 mismatches** — the column is trustworthy once the two date corrections above were applied.

### Sponsor
One value had a leading space (`" JKT BULOMBOLA"`), which would have silently split it into a separate category from any clean occurrence of the same sponsor. Trimmed.

### Patient ID
Not unique — 21 IDs repeat. Confirmed these are legitimate readmissions (same patient, different admission), not duplicate errors. Used as a join key, not a primary key; a surrogate key will be used at the database level.

### Gender, Discharge Reason, Region
No formatting issues found in any of these columns.

---

## Summary

| Check | Result |
|---|---|
| Rows | 905 (from 907 raw) |
| Duplicate rows | 0 |
| Nulls | 0 |
| Age mismatches vs. raw | 0 |
| Date / Admission Days mismatches | 0 |
| Identifying columns removed | 5 |

## Open items

- `KG` / `KGG` district codes unresolved — needs source-system confirmation
- 2 admission timestamps recovered from raw serials rather than the column's own text format — flagged for awareness

## Privacy note

The raw export originally included real names in `Patient Name`, `Admitted By Doctor`, `Admitted By`, `Discharged By`, and `Discharged By Doctor`, plus a broken `Next of Kin` field. All five have been removed from the published raw file (`Patient Number` renamed to `Patient ID` as the row identifier); none are needed for ward-, district-, or outcome-level analysis.
