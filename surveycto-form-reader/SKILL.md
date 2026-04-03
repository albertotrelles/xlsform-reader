---
name: surveycto-form-reader
description: >
  Parse and understand SurveyCTO/ODK form definition spreadsheets (.xlsx).
  Use this skill whenever the user mentions a SurveyCTO form, ODK form,
  XLSForm, form definition, survey instrument, questionnaire .xlsx, or
  asks you to understand variable names, choice lists, skip logic, or
  survey structure from a form file. Also trigger when the user asks to
  summarize a survey, generate a codebook or data dictionary, write
  analysis code (Stata/Python) that needs variable names and value labels
  from the form, or build high-frequency checks based on the form design.
  Even if the user just says "read the form" or "look at the questionnaire",
  use this skill. If a .xlsx file is uploaded and you suspect it might be
  a SurveyCTO/ODK form definition (look for sheets named "survey",
  "choices", "settings"), use this skill.
---

# SurveyCTO Form Reader

## Purpose

This skill lets you parse a SurveyCTO/ODK form definition spreadsheet
(.xlsx) into plain-text CSVs you can read, and gives you the domain
knowledge to understand the form's structure, variables, skip logic,
choice lists, and repeat groups. With this understanding you can:

- Summarize the survey instrument (sections, field counts, flow)
- Generate codebooks / data dictionaries
- Write analysis code (Stata, Python) using correct variable names and value labels
- Build data cleaning and high-frequency check scripts informed by form logic
- Explain what specific variables mean, including their skip conditions

## Step 1: Parse the form definition

The form .xlsx cannot be read directly. Run the parser script:

```bash
python3 /path/to/this/skill/scripts/parse_form.py "<path_to_form.xlsx>" --outdir /tmp/form_parsed
```

This produces CSV files in the output directory:
- `survey.csv` — all fields, groups, and their properties
- `choices.csv` — all choice lists with list_name, value, label
- `settings.csv` — form title, form ID, default language, version
- `summary.txt` — a human-readable overview of the form

**Read `summary.txt` first** for a quick overview, then consult the CSVs
as needed for details.

If the script is not at the expected path (e.g., the user installed the
skill elsewhere), look for `parse_form.py` inside a `scripts/` folder
adjacent to this SKILL.md.

## Step 2: Understand the form structure

After parsing, read `references/surveycto-form-spec.md` (adjacent to
this SKILL.md) to understand how to interpret the parsed data. Key
concepts:

- **survey sheet**: each row is a field. The `type` column determines
  what kind of field it is. The `name` column is the variable name in
  exported data. `label` is the question text. `relevance` is skip logic.
  `constraint` restricts valid entries.
- **choices sheet**: defines value labels for `select_one` and
  `select_multiple` fields. Linked by `list_name`.
- **Groups**: `begin group` / `end group` rows define sections.
  `begin repeat` / `end repeat` define roster/loop structures.
- **Field types**: `text`, `integer`, `decimal`, `select_one listname`,
  `select_multiple listname`, `date`, `datetime`, `geopoint`,
  `calculate`, `note`, and more.

## Step 3: Apply the knowledge

### Summarizing the survey
Walk through the survey sheet group by group. Report:
- Form title and ID (from settings)
- Number of sections (top-level groups)
- For each section: name, label, number of visible fields, key variables
- Repeat groups and what they represent (e.g., household roster)
- Total field count by type

### Writing analysis code (Stata / Python)

**CRITICAL: Always use a data-first approach.** When both a form
definition (.xlsx) and collected data (.csv / .dta) are available,
follow this workflow:

1. **Read the data file first.** Inspect the actual column headers /
   variable names present in the data. This is the ground truth —
   only these variables exist and can be referenced in code.
2. **Parse the form definition.** Run `parse_form.py` to get the
   survey and choices CSVs.
3. **Left-join from data to form.** For each variable in the data,
   look up its match in the parsed survey sheet's `name` column.
   This tells you the field type, label, skip logic, and (for
   select fields) which choice list to use.
4. **Generate code only for matched variables.** Never create label
   assignments, renames, or references for variables that are not
   in the data — even if they appear in the form. Fields may be
   absent from the data because they are disabled (`disabled` = "yes"),
   are note/group types that collect no data, or were added after
   data collection started.

**Stata-specific rules:**
- Stata truncates variable names to **32 characters** on import.
  When matching data columns to form names, if a form `name` exceeds
  32 characters, match against its first 32 characters. Use the
  truncated name throughout the .do file — do NOT attempt to rename
  variables to names longer than 32 characters, as Stata will reject
  them.
- For `select_one` fields: the variable stores a single numeric or
  string value from the `choices` sheet `value` column. Generate
  `label define` from the choices sheet, then `label values`.
- For `select_multiple` fields: the variable stores a **space-separated
  list** of selected values. In wide-format exports, SurveyCTO also
  creates binary indicator columns named `{fieldname}_{value}`.
- SurveyCTO data frequently stores numeric-looking values as strings
  (e.g., `"0"` / `"1"`, `"-99"`). After import, use `destring` with
  the `replace` option on numeric fields before labeling.
- Respect `relevance` conditions: variables behind skip logic will have
  missing values for observations that didn't meet the condition.

**Python-specific rules:**
- Column names in pandas match the `name` column from the survey sheet.
- `select_multiple` raw values are strings like `"1 3 5"` — split on
  space to get individual selections.
- Map values to labels using the choices sheet as a dictionary.
- Repeat group CSVs: `pd.merge()` on `KEY` / `PARENT_KEY`.

### Building high-frequency checks
Use the form to generate checks for:
- **Outliers**: `integer` and `decimal` fields; use `constraint` values
  as valid ranges if available.
- **Skip logic violations**: cross-check that variables behind
  `relevance` conditions are missing when the condition isn't met.
- **Completeness**: required fields (`required` = "yes") should not be
  missing.
- **Duplicates**: identify key ID fields (often near the top of the form).
- **Survey duration**: use metadata fields like `starttime`, `endtime`,
  `duration`.
- **Enumerator performance**: group checks by enumerator ID field.

### Generating a codebook
For each non-metadata, non-group field:
- Variable name (`name`)
- Question text (`label`)
- Type (simplified: text, numeric, single-choice, multiple-choice, date, etc.)
- Value labels (from choices sheet, if applicable)
- Skip condition (from `relevance`, translated to plain language)
- Constraints (from `constraint`, if any)
- Required (yes/no)

## Important notes

- The parser handles multilingual forms: columns like `label::English`,
  `label::Spanish` are all preserved. Use whichever language column the
  user needs.
- Metadata fields at the top of the form (deviceid, starttime, endtime,
  etc.) are hidden system fields — note them but don't treat them as
  survey questions.
- `calculate` and `calculate_here` fields are hidden computed fields —
  they appear in exported data but not in the survey UI.
- `note` fields display text but collect no data.
- Repeat groups produce separate data files in long-format exports,
  linked by KEY / PARENT_KEY.
