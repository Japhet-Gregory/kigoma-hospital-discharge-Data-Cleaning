# Before & After — Kigoma Hospital Discharge Dataset (January)

Column-by-column transformation from raw export to analysis-ready data, shown with real (de-identified) records and the exact Excel formulas used for each fix.

**Raw:** 907 rows × 17 columns → **Cleaned:** 905 rows × 13 columns

---

## Structural changes (before any column-level cleaning)

| Change | Detail |
|---|---|
| Removed 1 exact duplicate row | Same patient, same admission, identical in every field |
| Removed 1 test record | `Patient Name = "Test"`, District = `Temeke`, Region = `Dar es salaam` — a placeholder entry, not a real patient |
| Removed 5 identifying columns | `Admitted By Doctor`, `Admitted By`, `Discharged By`, `Discharged By Doctor`, `Next of Kin` — real staff names and a broken nested field, not needed for analysis |
| Consolidated 5 ward sheets → 1 | One sheet turned out to be a full hospital-wide master export, not a single ward — used as the sole source instead of appending duplicates |

---

## Age

![Before](screenshots/age_before.png)
![After](screenshots/age_after.png)

**Formula used** (converts text age to numeric years, months as a fraction):
```excel
=IF(ISNUMBER(SEARCH("month",[Age])), VALUE(LEFT([Age],FIND(" ",[Age])-1))/12, VALUE(LEFT([Age],FIND(" ",[Age])-1)))
```

**Display format applied** (keeps fractions at a fixed denominator instead of auto-simplifying):
```
Custom number format: # ??/12
```

**Bug caught via raw cross-check:** an earlier version had `Age` shifted by one row for the last ~50 patients — each row showed the *previous* patient's age. Invisible to null/format checks since the column was internally consistent, just misaligned. Fixed by rebuilding `Age` from the raw source, matched on Patient ID + Date Discharge (safe against duplicate Patient IDs from readmissions).

---

## District

![Before](screenshots/district_before.png)
![After](screenshots/district_after.png)

**Formula used** (removes hidden whitespace/non-printable characters before substituting):
```excel
=SUBSTITUTE(TRIM(CLEAN([@District])),"Manisipaa ya kigoma ujiji","Kigoma Ujiji Mc")
```

**Second pass**, merging the two spellings of the same district:
```excel
=SUBSTITUTE(SUBSTITUTE(TRIM(CLEAN([@District])),"Manisipaa ya kigoma ujiji","Kigoma Ujiji Mc"),"Buhingwe","Buhigwe")
```

`KG` and `KGG` were deliberately left unmapped — likely abbreviations for `Kigoma Ujiji Mc`, but not confirmed against the source system.

---

## Ward

![Before](screenshots/ward_before.png)
![After](screenshots/ward_after.png)

**Formula used** (standardizes 11 verbose raw labels into short categories via substring matching, most specific terms first to avoid mismatches):
```excel
=IFS(
ISNUMBER(SEARCH("NEONATAL",[@Ward])),"Neonatal",
ISNUMBER(SEARCH("LABOUR",[@Ward])),"Labour",
ISNUMBER(SEARCH("PAEDIATRIC",[@Ward])),"Paediatric",
ISNUMBER(SEARCH("FEMALE SURGICAL",[@Ward])),"Female Surg & Gyn",
ISNUMBER(SEARCH("PSYCHIATRIST",[@Ward])),"Psychiatrist",
ISNUMBER(SEARCH("GRADE 1",[@Ward])),"Grade 1",
ISNUMBER(SEARCH("MALE SURGICAL",[@Ward])),"Male Surg & Orthopedic",
ISNUMBER(SEARCH("MALE MEDICAL",[@Ward])),"Male Medical",
ISNUMBER(SEARCH("GRADE 2",[@Ward])),"Grade 2",
ISNUMBER(SEARCH("FEMALE WARD",[@Ward])),"Female Medical",
ISNUMBER(SEARCH("ICU",[@Ward])),"ICU",
TRUE,"UNMAPPED: "&[@Ward])
```

The `TRUE,"UNMAPPED: "&[@Ward]` fallback is deliberate — instead of erroring on any unrecognized value, it flags exactly which raw label slipped through, so nothing is silently miscategorized.

---

## Date Admitted

![Before](screenshots/date_before.png)
![After](screenshots/date_after.png)

**Format applied** to convert raw serial numbers into a real, readable timestamp:
```
Custom number format: YYYY-MM-DD HH:MM:SS
```

Two rows (highlighted above) had `Date Admitted` stored as raw numeric serials while every other row in the column was text — these converted to the invalid date `1970-01-01` in an earlier pass. The true timestamps were recovered directly from the raw serial values (not estimated), then verified against `Admission Days` with zero mismatches.

---

## Admission Days

**Validation formula used** to check the stored value against the real elapsed time:
```excel
=(Date Discharge - Date Admitted)  →  compared against stored Admission Days
```
0 mismatches across all 905 rows once the two date corrections above were applied.

---

## Sponsor

One value had a leading space (`" JKT BULOMBOLA"`), invisible on screen but enough to split it into a separate category. Fixed with the same `TRIM()` pattern used on District.

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

The raw export originally included real names in `Patient Name`, `Admitted By Doctor`, `Admitted By`, `Discharged By`, and `Discharged By Doctor`, plus a broken `Next of Kin` field. All five have been removed from the published raw file (`Patient Number` renamed to `Patient ID`); none are needed for ward-, district-, or outcome-level analysis. Screenshots above show Patient ID only, no names.
