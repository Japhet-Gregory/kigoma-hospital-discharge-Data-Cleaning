# Hospital Admission & Discharge Records — Data Cleaning

A documented Excel-based data-cleaning workflow for a hospital admission/discharge dataset (Kigoma region). The workbook takes a messy raw export and turns it into an analysis-ready dataset, with every fix implemented as a visible, auditable formula rather than a silent manual edit.

## Contents

| Sheet | Description |
|---|---|
| `Raw Dataset` | Untouched source export — 907 rows × 18 columns |
| `Cleaning Process` | Raw columns paired with helper columns that validate or standardize each field, formula by formula |
| `Clean Dataset` | Final corrected data, condensed to 12 columns — 905 rows, ready for analysis |

## Dataset overview

- **Raw records:** 907 rows × 18 columns
- **Records carried through the cleaning workflow:** 906 rows analyzed
- **Final clean records:** 905 rows × 12 columns
- **Coverage:** admissions/discharges predominantly for facilities in the Kigoma region

**Final columns:** `Patient Name`, `Patient Number`, `Age (Yrs)`, `Gender`, `District`, `Ward`, `Sponsor`, `Date Admitted`, `Clean Discharge`, `Days`, `Discharge Reason`, `Region`

## Data quality issues identified

| Field | Issue Found | Detail |
|---|---|---|
| Patient Number | Duplicate IDs | 44 of 906 rows share a Patient Number with another row; 862 are unique |
| Age | Mixed units in one field | Stored as free text ("1 years", "11 months") — not usable for numeric analysis |
| Gender | Inconsistent labels | Checked for values outside "Male"/"Female"; all 906 rows passed as Valid |
| District | Spelling/naming variants | 11 raw spellings collapsed to 10 standardized names (e.g. "Manisipaa ya kigoma ujiji" → "Kigoma Ujiji Mc") |
| Ward | Free-text ward/room labels | Long raw labels mapped to 11 standardized ward categories |
| Sponsor | Unstandardized categories | 9 distinct payer categories inventoried (CASH, NHIF, NSSF, MHIS, FAST TRACK, CHF, OTHER INSURANCE, STRATEGIES, + one facility code) |
| Date Admitted | Text, not real dates | 904 of 906 rows stored the timestamp as text, blocking date math and sorting |
| Date Discharge | Mostly valid | Already numeric/date in nearly all rows; standardized into a "Clean Discharge" column |
| Days (length of stay) | Checked for errors | Row-by-row check: all 906 rows numeric and non-negative |
| Discharge Reason | Category inventory | 5 categories in use: Normal Discharge, Escape, DAMA, Death, Referred |
| Region | Outlier values | 3 distinct regions found: Kigoma (904 rows), Katavi (1), Tabora (1) |

## Cleaning methodology

Every field is cleaned with a dedicated formula in an adjacent helper column (headers highlighted in green), so raw and corrected values sit side by side. Two validation techniques are used:

- **Row-by-row checks** (Patient Name, Patient Number, Age, Gender, Days) — evaluate every row and return a per-row verdict (`OK` / `Text Error` / `Negative`, etc.).
- **Category-inventory checks** (Sponsor, Discharge Reason, Region) — a single `UNIQUE()` array formula placed once, which spills down to list every distinct value in that column, so unexpected categories stand out without scanning 900+ rows.

<details>
<summary><strong>Formula reference</strong> (click to expand)</summary>

| Field | Column | Formula |
|---|---|---|
| Patient Name (completeness) | B | `=IF(LEN(TRIM(A2))-LEN(SUBSTITUTE(TRIM(A2)," ",""))>=1,TRIM(A2),"⚠ Only 1 name: "&TRIM(A2))` |
| Patient Number (duplicate check) | D | `=IF(COUNTIF($C$2:$C$905,C2)>1,"Duplicate","Unique")` |
| Age (text → number) | F | `=IF(ISNUMBER(SEARCH("month",E2)), LEFT(E2,FIND(" ",E2)-1)&"/12", VALUE(LEFT(E2,FIND(" ",E2)-1)))` |
| Gender (validity) | H | `=IF(OR(TRIM(G2)="Male", TRIM(G2)="Female"), "Valid", "Inconsistent")` |
| District (standardization) | J | `=SUBSTITUTE(SUBSTITUTE(TRIM(CLEAN(I2)),"Manisipaa ya kigoma ujiji","Kigoma Ujiji Mc"),"Buhingwe","Buhigwe")` |
| Ward (keyword mapping) | L | `=IFS(ISNUMBER(SEARCH("ICU",K2)),"ICU", ISNUMBER(SEARCH("NEONATAL",K2)),PROPER("NEONATAL"), … TRUE,"Unmapped: "&K2)` |
| Sponsor (category inventory) | N | `=UNIQUE(M2:M907)` |
| Date Admitted (text-vs-date) | P | `=IF(ISNUMBER(O2), "Valid Date", "Text Error")` |
| Date Admitted, parsed (re-check) | R | `=IF(ISNUMBER(Q2), "Valid Date", "Text Error")` |
| Days (non-numeric / negative) | V | `=IF(NOT(ISNUMBER(U2)), "Text Error", IF(U2<0, "Negative", "OK"))` |
| Discharge Reason (category inventory) | X | `=UNIQUE(W2:W907)` |
| Region (category inventory) | Z | `=UNIQUE(Y2:Y907)` |

</details>

### Date column fix: Text to Columns

The broken `Date Admitted` column (text instead of real dates) was standardized using Excel's **Text to Columns** feature:

1. Select the entire Date column (Column O).
2. Go to the **Data** tab → **Text to Columns**.
3. Choose **Delimited** → Next.
4. Uncheck all delimiter boxes → Next.
5. Under **Column data format**, choose **Date** → **YMD**.
6. Click **Finish**.
7. Format the column as Short Date or a custom `YYYY-MM-DD HH:MM:SS` format (`Ctrl+1`).

## Validation summary

| Field | Outcome |
|---|---|
| **Date Admitted** | Confirmed broken (904/906 rows as text) → standardized via Text to Columns |
| **Days** | Confirmed valid — all 906 rows numeric and non-negative, no correction needed |
| **Discharge Reason** | Confirmed valid — all 5 categories are legitimate, expected values |
| **Region** | Confirmed valid — all 3 values are legitimate entries, not typos |
| **Patient Number (44 duplicates)** | Confirmed as **readmitted patients** (same patient, separate admission episode) — not data-entry duplicates; retained as distinct admission records |

## Recommendations

- Keep the 44 duplicate Patient Numbers as separate admission records — they're readmissions, not errors — but distinguish "patients" from "admissions" when counting.
- Re-apply the Text-to-Columns fix to `Date Discharge` if it's ever re-entered as text.
- Retain the standardized District, Ward, and Discharge Reason categories for consistent reporting going forward.
- The `Clean Dataset` sheet is ready to use as the analysis-ready source for downstream reporting.

## Repository structure

```
.
├── Data_Cleaning_process.xlsx                    # Source workbook (Raw Dataset, Cleaning Process, Clean Dataset)
├── Data_Cleaning_Process_Summary_Report.docx     # Full write-up with screenshots
└── README.md                                     # This file
```

## How to use

1. Open `Data_Cleaning_process.xlsx`.
2. Review `Raw Dataset` for the untouched source data.
3. Review `Cleaning Process` to see each validation/standardization formula next to its raw column.
4. Use `Clean Dataset` as the ready-to-analyze table.

## License

Add your preferred license here (e.g. MIT).
