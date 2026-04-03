# xlsform-reader

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that parses XLSForm survey definition spreadsheets (`.xlsx`) and gives Claude Code deep understanding of your survey's structure, variables, skip logic, and choice lists.

Works with **SurveyCTO**, **ODK**, **KoBoToolbox**, and **ONA** — any platform that uses the [XLSForm standard](https://xlsform.org/).

## What it does

Once installed, Claude Code can:

- **Summarize a survey instrument** — sections, field counts, flow, repeat groups
- **Generate codebooks / data dictionaries** — variable names, labels, types, value labels, skip conditions
- **Write analysis code (Stata / Python)** using correct variable names and value labels from the form
- **Build high-frequency check (HFC) scripts** *(coming soon)* — outlier detection, skip logic violations, completeness checks, enumerator performance
- **Generate Stata `.do` files** with `label define` / `label values` for all `select_one` and `select_multiple` variables

The skill uses a **data-first workflow**: when both a form definition and collected data are available, it reads the data columns first and left-joins against the form — so it never generates code for variables that don't exist in the data.

## How it works

XLSForm `.xlsx` files can't be read directly by Claude Code. The skill includes a Python script (`parse_form.py`) that converts the three sheets — `survey`, `choices`, and `settings` — into plain-text CSVs and a human-readable summary. Claude Code then reads these files alongside a bundled reference doc that explains XLSForm structure, field types, expression syntax, and implications for Stata/Python analysis.

## Installation

### 1. Install the skill (user-level — available in all sessions)

```bash
git clone https://github.com/<your-username>/xlsform-reader.git
cp -r xlsform-reader/surveycto-form-reader ~/.claude/skills/
```

Or manually: copy the `surveycto-form-reader/` folder into `~/.claude/skills/`.

### 2. Install the slash command (optional)

```bash
cp xlsform-reader/commands/parse-forms.md ~/.claude/commands/
```

This gives you a `/parse-forms` command that scans the current directory for all XLSForm `.xlsx` files and generates a markdown summary for each one.

### 3. Verify

Start a new Claude Code session and ask:

```
What skills do you have available?
```

You should see `surveycto-form-reader` in the list.

## Usage

The skill triggers automatically when you mention SurveyCTO forms, ODK forms, XLSForm files, survey instruments, or ask about variable names, choice lists, or survey structure.

**Summarize a survey:**
```
Read the SurveyCTO form definition survey.xlsx and give me a summary of the survey.
```

**Generate a codebook:**
```
Read survey.xlsx and generate a codebook as a markdown file.
```

**Create Stata value labels:**
```
Using survey.xlsx and data.csv, generate a Stata .do file that labels all select_one variables.
```

**Run high-frequency checks** *(coming soon)***:**
```
Using survey.xlsx and data.csv, run basic HFC checks: survey duration, outliers, and completeness. Summarize by enumerator.
```

**Scan all forms in a folder (with slash command installed):**
```
/parse-forms
```

## Slash commands

The repo includes optional slash commands — saved prompts you can invoke with a single shortcut in Claude Code.

| Command | What it does |
|---------|-------------|
| `/parse-forms` | Scans the current directory for all XLSForm `.xlsx` files and generates a markdown summary for each one |
| `/label-stata` | Reads form definitions and data files in the current directory, then generates a Stata `.do` file with value labels for all `select_one` and `select_multiple` variables that exist in the data |

### Installing commands

```bash
cp xlsform-reader/commands/*.md ~/.claude/commands/
```

Commands are installed at the user level (`~/.claude/commands/`) so they work in any Claude Code session.

## Repo structure

```
xlsform-reader/
├── README.md
├── LICENSE
├── surveycto-form-reader/                # The skill — copy this folder to ~/.claude/skills/
│   ├── SKILL.md                          # Main instructions
│   ├── scripts/
│   │   └── parse_form.py                 # Converts .xlsx → CSVs + summary
│   └── references/
│       └── surveycto-form-spec.md        # XLSForm reference (field types,
│                                         #   expression syntax, export format,
│                                         #   Stata/Python implications)
└── commands/                             # Slash commands — copy .md files to ~/.claude/commands/
    ├── parse-forms.md                    # /parse-forms — summarize all forms in current directory
    └── label-stata.md                    # /label-stata — generate Stata value labels from form + data
```

## Key design decisions

- **Data-first workflow**: when data is available, the skill reads actual data columns before consulting the form — preventing references to disabled, missing, or renamed variables.
- **Stata 32-character limit**: the skill knows that Stata truncates variable names to 32 characters and matches accordingly. It will never try to rename a variable to a name longer than 32 characters.
- **Bundled reference docs**: SurveyCTO/XLSForm documentation is curated into a single markdown file (~240 lines) that loads efficiently without network access. No live fetching required.
- **String vs. numeric dtypes**: the skill warns about SurveyCTO's tendency to store numeric-looking values as strings and includes guidance for `destring` (Stata) and `pd.to_numeric` (Python).

## XLSForm compatibility

This skill was built and tested with SurveyCTO forms but works with any XLSForm-based platform because they all share the same core structure:

| Platform | Status |
|----------|--------|
| SurveyCTO | ✅ Full support (including SurveyCTO-specific extensions) |
| ODK (Open Data Kit) | ✅ Supported |
| KoBoToolbox | ✅ Supported |
| ONA | ✅ Supported |

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) (Pro, Max, Team, or Enterprise plan)
- Python 3.7+ (for `parse_form.py`)
- `openpyxl` (auto-installed by the script if missing)

## License

[MIT](LICENSE)