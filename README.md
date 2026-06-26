# Data Cleaning agent skill

Data cleaning co-pilot. Produces a Jupyter notebook audit trail and reusable Python pipeline.

## Usage

```
User provides dataset path → Load & profile → Iterate through cleaning issues → Export clean_pipeline.py
```

Works along with Jupyter notebook agent skills and Pandas docs as context. 

## Output

- `output/data-cleaning/<dataset>-cleaning.ipynb` — Full exploration trace
- `output/data-cleaning/clean_<dataset>.py` — Reusable cleaning pipeline with `clean_data(input_path, output_path)` function

## Reference

- `references/cleaning-checklist.md` — Data quality categories to check
- `references/pandas-cleaning-patterns.md` — Common pandas cleaning snippets