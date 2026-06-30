---
name: "data-cleaning"
description: "Use when the user asks to clean, fix, or explore a dataset for data quality issues — for example, 'clean this CSV', 'find data quality issues in this file', 'explore and fix this dataset', 'prepare this data for analysis by handling nulls and duplicates', or 'help me clean up this messy Excel file'. Use this skill whenever the user mentions a dataset they want to clean, sanitise, or explore for quality problems, even if they don't explicitly say 'data cleaning'."
---

# Data Cleaning Skill

Explore, diagnose, and fix data quality issues in a user-provided dataset through conversational interaction. Cleaning steps are recorded in a Jupyter notebook (as an audit trail) and exported at the end as a reusable Python pipeline script.

## Overview

```
User provides dataset path
       │
       ▼
Load data → profile shape, dtypes, head
       │
       ▼
Iterate: spot issue → discuss with user → apply fix in notebook
       │  (guided by cleaning-checklist.md)
       │
       ▼
User confirms done → export pipeline .py + save .ipynb
```

## Workflow

### 1. Accept the dataset

The user provides a local path to their dataset. Accept any format pandas can read (CSV, Excel, Parquet, JSON, Feather, etc.). If the file extension is ambiguous, try `pd.read_csv` first, then fall back to other readers.

If the path doesn't exist or can't be read, explain why and ask the user to check the path.

This skill does NOT generate demo/synthetic data. The user always brings their own dataset.

### 2. Scaffold a notebook

Use the jupyter-notebook skill to create a new experiment notebook. The notebook is the live workspace where all exploration and cleaning happens.

Title the notebook after the dataset filename:
```bash
uv run .agents/skills/jupyter-notebook/scripts/new_notebook.py \
  --kind experiment \
  --title "Data cleaning: <filename>" \
  --out output/data-cleaning/<filename-slug>-cleaning.ipynb
```

### 3. Profile the data first, then iterate

Load the dataset in the first code cell. Print:
- Shape (rows, columns)
- Column names and dtypes
- `.head()` and `.sample(5)` for a glance
- `df.describe(include='all')` for summary stats

After the initial profile, enter the **explore-fix loop**:

**One issue at a time.** Use `references/cleaning-checklist.md` as your systematic guide. For each category:
- Detect the issue (write a code cell, show results)
- Explain what you found in plain language
- Propose 1–3 fix options with trade-offs
- Let the user decide (or suggest skipping)
- Apply the fix in a new code cell only after user confirms
- Annotate the fix with a markdown cell explaining what was done

**Do NOT batch multiple fixes silently** — every cleaning operation must be discussed first.

### 4. Handle uncertainty

If you're unsure about a fix (domain-specific logic, ambiguous values, risk of data loss), state the uncertainty clearly and ask the user. Show the relevant rows. Suggest options, but don't assume.

### 5. Know when to stop

The user signals when they're done. Before assuming the exploration is complete, ask explicitly: *"I've covered the checklist categories I can detect. Do you want me to check anything else, or are you ready to export the cleaning pipeline?"*

### 6. Export the pipeline

When the user confirms, extract every cleaning operation performed in the notebook into a standalone `.py` file.

Write a function with this signature:
```python
def clean_data(input_path: str, output_path: str | None = None) -> pd.DataFrame:
```

The function:
- Reads the dataset
- Applies all cleaning steps **in the same order** they were performed in the notebook
- Returns the cleaned DataFrame
- If `output_path` is provided, saves the cleaned data (infer format from extension, default CSV)

Add an `if __name__ == "__main__"` block that uses `argparse` for CLI usage:
```bash
python clean_<filename>.py input.csv -o output.csv
```

Save the pipeline as `output/data-cleaning/clean_<filename>.py`.

### 7. Save outputs

| File | Location | Purpose |
|------|----------|---------|
| `.ipynb` | `output/data-cleaning/<slug>-cleaning.ipynb` | Full trace of exploration and decisions |
| `clean_<filename>.py` | `output/data-cleaning/clean_<filename>.py` | Reusable automation pipeline |

### 8. Report summary

When done, tell the user what was saved and where. Briefly summarise what was found and fixed (2–3 sentences). Mention the pipeline function signature so they know how to call it.

## Conversation guidelines

- **Be specific.** Show actual values, counts, and example rows instead of vague statements. *"Column 'age' has 12 null values (2.4% of rows)"* not *"There are some missing values."*
- **Use pandas output.** Print DataFrames, Series, or scalar results so the user can see what you see.
- **Don't overwhelm.** Present one finding at a time. If a category has multiple sub-issues, show the most impactful first.
- **Offer trade-offs.** When suggesting a fix, briefly mention pros and cons. *"Dropping these 12 rows is the simplest option, but you'd lose data. Filling with the median preserves row count but may bias the distribution."*
- **Respect skip decisions.** If the user says skip, note it in a markdown cell and move on. Don't re-ask.

## Reference files
- `references/cleaning-checklist.md` — Systematic categories of data quality issues to check
- `references/pandas-cleaning-patterns.md` — Curated pandas snippets for common cleaning operations

Read both reference files before starting the exploration phase.
