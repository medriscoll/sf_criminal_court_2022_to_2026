# San Francisco Criminal Court Data

Linked criminal-court records for San Francisco County, spanning approximately 4 years of case history. Combines scraped Superior Court docket data, District Attorney open-data feeds, and a charge-disposition spreadsheet produced by the Court under California Rules of Court rule 10.500.

**License:** CC-BY-NC-4.0 (non-commercial use only)

---

## Parquet Files

### `cases.parquet` — 77,406 rows
Case-level records, one row per case. Contains public case numbers, internal court case IDs, defendant names, and filing dates. This is the primary dimension table that most other files join to via `case_number`.

---

### `register_of_actions.parquet` — 776,728 rows
Full docket entries for each case — every proceeding, filing, and event logged in the court's register of actions. Includes filer names, entry dates, and docket text. The most granular longitudinal record of what happened in each case.

---

### `calendar.parquet` — 318,993 rows
Scheduled and completed hearing entries. Contains hearing dates, hearing types, courtroom/department identifiers, and case references. One row per hearing event.

---

### `calendar_with_judicial_assignments.parquet` — 318,993 rows
`calendar.parquet` left-joined to `judicial_department_assignments` using calendar department and hearing date falling within the published assignment effective range. Adds the publicly assigned judge for each hearing where a match exists. Includes join-confidence fields:
- `deterministic_to_published_department_assignment` — exact department/date match to a single assigned judge
- `published_assignment_month_only_date` — joined to a source with month/year precision only (date basis flagged)
- Null — no matching public assignment source available for that row

---

### `attorneys.parquet` — 72,289 rows
Attorney appearances on each case. One row per attorney-case pairing. Includes attorney name, bar number (where available), and role (e.g., defense, prosecution).

---

### `da_arrests.parquet` — 176,130 rows
San Francisco District Attorney arrest feed. One row per arrest incident. Fields include incident number, court number, arrest date, arresting agency, booked charges, booked case type, crime type, domestic violence flag, DA action taken, and data freshness timestamps.

---

### `da_prosecuted.parquet` — 120,827 rows
DA prosecuted-cases feed. One row per prosecuted case incident. Fields include incident number, court number, arrest date, filed charges, filed case type, crime type, domestic violence flag, DA action taken, and case status.

---

### `sfsc_case_matches.parquet` — 13,790 rows
Deterministic-inferred matches linking anonymized Court-provided case IDs (from the rule 10.500 spreadsheet) back to public case numbers. Match key is `filed_date + normalized charge multiset`, accepted only when the composite key is unique on both sides. Labeled "deterministic-inferred" because the Court declined to supply the public case number directly, requiring inference of a join that should have been given outright.

---

### `sfsc_charge_dispositions.parquet` — 44,029 rows
Charge-level dispositions, sentences, and outcomes — the most detailed sentencing data in the dataset. Sourced from the rule 10.500 spreadsheet and enriched with public case numbers where a strict match succeeded. Fields include:
- Statute descriptions
- Initial and disposition charge types
- Disposition date and type
- Diversion type
- Sentence type and date
- Probation days, jail days, prison days

---

### `judicial_officers.parquet` — 77 rows
Canonical judge/officer dimension table. Maps raw extracted name variants (OCR misspellings, missing middle initials, ordering differences) to a single canonical key per judge. Used to merge name variants across sources while preserving auditability of the source strings.

---

### `judicial_assignments.parquet` — 804 rows
Raw extracted assignment rows from each source document, typically one row per department/judge/function. Preserves the source representation before normalization.

---

### `judicial_department_assignments.parquet` — 777 rows
Normalized one-row-per-source/date/department table. Intended for joining to calendar data by `department + hearing date` falling within an effective date range. Built from public SF Superior Court assignment memos, department PDFs (2022–2025), and the current judicial assignments web page.

---

### `judicial_assignment_sources.parquet` — 14 rows
Source document registry. One row per source document or page set used to build the judicial assignment tables. Fields include URL, local file path, effective start/end dates, extraction method (PDF parse vs. OCR), and date-basis notes. Text-based PDFs were parsed with `pdftotext`; scanned/image PDFs were OCR'd with `glm-ocr` via Ollama.

---

## Key Joins

| Join | Keys |
|------|------|
| `cases` → `register_of_actions` | `case_number` |
| `cases` → `calendar` | `case_number` |
| `cases` → `attorneys` | `case_number` |
| `cases` → `sfsc_charge_dispositions` | `case_number` |
| `calendar` → `judicial_department_assignments` | `department` + hearing date within effective range |
| `sfsc_case_matches` → `sfsc_charge_dispositions` | anonymized court ID |
| `da_arrests` / `da_prosecuted` → `cases` | `court_number` / `case_number` |

---

## Data Sources

- **Superior Court scrape** — `cases`, `calendar`, `attorneys`, `register_of_actions`
- **DA open-data feeds** — `da_arrests`, `da_prosecuted`
- **Rule 10.500 records request** — `sfsc_case_matches`, `sfsc_charge_dispositions` (Court supplied anonymized IDs rather than public case numbers; matches were inferred)
- **Public assignment documents** — `judicial_*` tables (assignment memos, department PDFs, current assignments page)
