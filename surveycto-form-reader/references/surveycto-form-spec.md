# SurveyCTO / XLSForm — Form Definition Reference

This reference covers everything you need to interpret a parsed SurveyCTO
form definition. Consult this when working with the CSVs produced by
`parse_form.py`.

---

## 1. Workbook structure

A SurveyCTO form definition is an .xlsx workbook with three key sheets:

| Sheet      | Purpose |
|------------|---------|
| **survey** | Every field, group, and repeat in the form — one row per element |
| **choices** | Value labels for all multiple-choice fields |
| **settings** | Form-wide metadata: title, ID, version, default language |

---

## 2. The survey sheet

### Core columns

| Column | Description |
|--------|-------------|
| `type` | Field type (see §3). For select fields, includes the list name: `select_one yesno` |
| `name` | Variable name — **this becomes the column header in exported data** |
| `label` | Question text shown to the enumerator. Multilingual forms have `label::English`, `label::Spanish`, etc. |
| `hint` | Help text shown below the question |
| `relevance` | Skip logic expression. Field only appears when this evaluates to true. Uses `${fieldname}` syntax to reference other fields. Example: `${consent} = 1` |
| `constraint` | Validation rule. Example: `. >= 0 and . <= 125` (age between 0 and 125) |
| `constraint_message` | Error text shown when constraint fails |
| `required` | `yes` or `1` = field must be answered |
| `appearance` | Controls display: `minimal`, `field-list`, `likert`, `quick`, etc. |
| `default` | Pre-filled value |
| `calculation` | Expression for `calculate` and `calculate_here` fields |
| `repeat_count` | For `begin repeat`: how many iterations (can be an expression) |
| `choice_filter` | Expression to dynamically filter choices |
| `disabled` | `yes` to disable a field |
| `read_only` | `yes` to make field non-editable |
| `response_note` | Note shown after field is answered |

### Common metadata fields (top of form)

These are hidden system fields — they collect device/timing info automatically:

| type | name | What it captures |
|------|------|------------------|
| `start` | `starttime` | Timestamp when form was opened |
| `end` | `endtime` | Timestamp when form was finalized |
| `today` | `today` | Date of form submission |
| `deviceid` | `deviceid` | Unique device identifier |
| `subscriberid` | `subscriberid` | SIM subscriber ID |
| `simserial` | `simserial` | SIM serial number |
| `phonenumber` | `phonenumber` | Phone number if available |
| `username` | `username` | SurveyCTO username of enumerator |
| `caseid` | `caseid` | Auto-generated unique submission ID |
| `audit` | `audit` | Tracks field-by-field timing |

### Groups and repeats

- `begin group` / `end group` — Define a section. The group's `name`
  is used as a nesting prefix in some export formats.
- `begin repeat` / `end repeat` — Define a loop (e.g., household roster).
  In **wide-format** exports, repeated fields become `{name}_1`,
  `{name}_2`, etc. In **long-format** exports, each repeat instance is
  a separate row in a separate CSV, linked by `KEY` / `PARENT_KEY`.

---

## 3. Field types

### Visible (data-collecting) fields

| Type | Data stored | Notes |
|------|-------------|-------|
| `text` | String | Free text entry |
| `integer` | Integer | Max 9 digits; use text with `numbers` appearance for more |
| `decimal` | Decimal number | |
| `select_one listname` | Single value from choice list | Stored as the `value` from choices sheet |
| `select_multiple listname` | Space-separated values | E.g., `"1 3 5"`. Wide exports also create binary columns `{name}_{value}` |
| `date` | Date string | Format: YYYY-MM-DD |
| `datetime` | Date+time string | |
| `time` | Time string | |
| `geopoint` | Lat, lon, altitude, accuracy | Space-separated: `"-12.05 -77.03 0 5"` |
| `geotrace` | Series of geopoints | |
| `geoshape` | Polygon of geopoints | |
| `image` | Filename | Photo capture |
| `audio` | Filename | Audio recording |
| `video` | Filename | Video recording |
| `barcode` | String | Barcode/QR scan |
| `note` | *No data collected* | Display-only text |
| `acknowledge` | `"OK"` or empty | Confirmation screen |
| `rank listname` | Ordered list of values | Ranking of choices |
| `file` | Filename | Generic file upload |

### Hidden (computed) fields

| Type | Behavior |
|------|----------|
| `calculate` | Auto-computed using `calculation` column. Recalculates whenever dependencies change. |
| `calculate_here` | Like `calculate`, but only computed when the enumerator reaches this point in the form. Useful for `once(duration())` or `once(now())`. |

---

## 4. The choices sheet

| Column | Description |
|--------|-------------|
| `list_name` | Identifier linking to `select_one listname` / `select_multiple listname` in survey sheet |
| `value` | The stored data value (what appears in exported data). Best practice: use numeric values for Stata compatibility. |
| `label` | Display text shown to enumerator. Multilingual: `label::English`, etc. |
| `image` | Optional image filename for the choice |

### Example

| list_name | value | label |
|-----------|-------|-------|
| yesno | 1 | Yes |
| yesno | 0 | No |
| crops | 1 | Maize |
| crops | 2 | Rice |
| crops | 3 | Beans |
| crops | -88 | Other (specify) |
| crops | -99 | Don't know |

### Key behaviors

- For `select_one`: the exported variable contains a single value (e.g., `1`).
- For `select_multiple`: the exported variable contains space-separated
  values (e.g., `"1 3"`). In wide-format exports, SurveyCTO also
  generates binary indicator columns: `crops_1`, `crops_2`, `crops_3`, etc.
  where `1` = selected and `0` = not selected.
- Non-response codes are typically negative: `-88` for "Other",
  `-99` for "Don't know", `-77` for "Refused to answer". These
  conventions vary by project.

---

## 5. The settings sheet

| Column | Description |
|--------|-------------|
| `form_title` | Human-readable form name |
| `form_id` | Unique identifier (no spaces/punctuation) |
| `version` | Version string (must be lexically greater than previous) |
| `default_language` | Language used when no `::language` suffix specified |
| `public_key` | Encryption key (if form uses encryption) |
| `submission_url` | Custom submission endpoint (rare) |

---

## 6. Expression syntax (relevance, constraint, calculation)

SurveyCTO uses XPath-like expressions:

### Referencing fields
- `${fieldname}` — value of another field

### Operators
- Comparison: `=`, `!=`, `<`, `>`, `<=`, `>=`
- Logic: `and`, `or`, `not()`
- Math: `+`, `-`, `*`, `div`, `mod`

### Common functions
- `selected(${field}, 'value')` — true if value is selected in a
  select_one or select_multiple
- `count-selected(${field})` — number of selections in select_multiple
- `selected-at(${field}, N)` — Nth selected value (0-indexed)
- `if(condition, then, else)` — conditional
- `coalesce(a, b)` — first non-empty value
- `int(${field})` — convert to integer
- `string-length(${field})` — text length
- `concat(a, b, ...)` — concatenate strings
- `substr(${field}, start, end)` — substring
- `regex(${field}, 'pattern')` — regex match
- `pulldata('dataset', 'col', 'key_col', ${key})` — lookup from
  pre-loaded dataset
- `duration()` — seconds since form was opened
- `once(expr)` — evaluate only once (first time reached)
- `now()` — current datetime
- `today()` — current date
- `join(' ', ${repeated_field})` — join repeat values into a list

---

## 7. Exported data structure

### Standard columns added by SurveyCTO (not in form definition)
- `KEY` — unique submission identifier
- `instanceID` — duplicate of KEY
- `SubmissionDate` — when data was synced to server (NOT when survey
  was completed; use `starttime`/`endtime` for actual timing)
- `formdef_version` — form version used
- `review_status`, `review_quality` — if review workflow is enabled

### Repeat group exports
- **Wide format**: repeated fields become `{name}_1`, `{name}_2`, ...
  in the main CSV. Number of columns = max repeat count across all
  submissions.
- **Long format**: each repeat group exports as a **separate CSV**.
  Each row has its own `KEY` and a `PARENT_KEY` linking to the main
  submission. Use `PARENT_KEY` to merge back.

---

## 8. Implications for analysis code

### CRITICAL: Data-first workflow

When both a form definition and collected data are available, **always
start from the data, not the form.** Read the actual data file's column
headers first, then use the form as a lookup reference. This prevents:
- Referencing variables that don't exist in the data (disabled fields,
  note fields, fields added after collection)
- Name mismatches due to truncation (Stata's 32-char limit)
- Labeling variables that were never exported

Workflow:
1. Read data columns (CSV headers or Stata `describe`)
2. Parse form definition into CSVs
3. For each data column, find its match in the survey sheet `name` column
4. Only generate code for matched variables

### Stata
- Variable names = `name` column from survey sheet, **truncated to 32
  characters** by Stata on import. Never attempt to rename a variable
  to a name longer than 32 characters — Stata will reject it as an
  invalid name. When matching, compare the data's (possibly truncated)
  variable name against the first 32 characters of the form's `name`.
- Value labels: create `label define` from choices sheet, then
  `label values varname labelname`. Only assign labels to variables
  confirmed to exist in the data.
- Use `capture` before `label values` or `destring` commands to
  gracefully handle variables that may not exist.
- `select_multiple` in wide format: binary vars `{name}_{value}`,
  stored as 0/1. **Watch out**: sometimes stored as string "0"/"1"
  instead of numeric — use `destring, replace` before labeling.
- Repeat groups in long format: `merge` using `KEY` / `PARENT_KEY`
- Skip logic → expect missing values for fields behind `relevance`
- Check the `disabled` column in the survey sheet — disabled fields
  are not exported and should be skipped entirely.

### Python (pandas)
- Column names = `name` column from survey sheet (no truncation issue)
- `select_multiple` raw values are strings like `"1 3 5"` — split on
  space to get individual selections
- Map values to labels using the choices sheet as a dictionary
- Repeat group CSVs: `pd.merge()` on `KEY` / `PARENT_KEY`
- SurveyCTO data frequently stores numeric-looking values as strings;
  use `pd.to_numeric(col, errors='coerce')` before analysis
