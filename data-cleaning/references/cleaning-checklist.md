# Data Cleaning Checklist

Systematic categories of data quality issues to check, in suggested order. For each category, detect the issue, present findings, discuss with the user, and apply only on confirmation.

---

## 1. Missing Values

Detect:
```python
df.isna().sum()
df.isna().mean().round(4)  # fraction per column
```
For each column with missing values, show the count, percentage, and a few example rows. Discuss:
- Drop rows (if few, random pattern)
- Fill with constant, mean, median, mode (explain the bias trade-off)
- Fill with forward/backward fill (for time series)
- Mark as a sentinel value (`-1`, `"Unknown"`)
- Leave as-is and note the reason

---

## 2. Duplicates

Detect exact duplicates:
```python
df.duplicated().sum()
df[df.duplicated(keep='first')]
```
For near-duplicates (fuzzy matching), flag but ask before deduplicating — it's riskier.

Discuss:
- Drop duplicates (keep first / last / none)
- Keep all if legitimate (e.g., timestamps differ, transaction IDs)
- Flag duplicates for review rather than dropping

---

## 3. Type Coercion

Check inferred vs intended dtypes:
```python
df.dtypes
```
Common problems:
- Numeric column stored as `object` (contains strings like `"$1,000"`, `"N/A"`)
- Date column stored as `object` or `int`
- Boolean column stored as `object` or `float`

For each mismatch:
```python
pd.to_numeric(..., errors='coerce')
pd.to_datetime(..., errors='coerce')
df['col'].astype('int64')
```
Show which values fail conversion (the `coerce` turns them to NaN) so the user can decide how to handle them.

---

## 4. Outliers

For numeric columns, detect potential outliers.

**IQR method:**
```python
Q1 = df['col'].quantile(0.25)
Q3 = df['col'].quantile(0.75)
IQR = Q3 - Q1
outliers = df[(df['col'] < Q1 - 1.5*IQR) | (df['col'] > Q3 + 1.5*IQR)]
```

**Z-score method (for roughly normal distributions):**
```python
from scipy import stats
z = np.abs(stats.zscore(df['col'].dropna()))
outliers = df[z > 3]
```

Show the outlier rows and their values. Discuss:
- Drop if clearly erroneous (sensor glitch, typo)
- Cap/clip at reasonable bounds (winsorize)
- Keep if legitimate (income data, extreme but real values)
- Investigate further — could indicate a data entry issue

---

## 5. Inconsistent Formatting

Common categories to check:

**String casing / whitespace:**
```python
df['col'].str.strip().str.lower()  # normalise
```
Check for trailing whitespace, mixed case, leading zeros that should be strings.

**Date formats:**
```python
# Check for mixed formats in a single column
df['date_col'].apply(type).value_counts()
# Check unique patterns
df['date_col'].astype(str).str[:10].value_counts()
```
Dates may arrive as `2024-01-15`, `01/15/2024`, `Jan 15 2024` in the same column.

**Categorical values:**
```python
df['col'].value_counts()
```
Look for near-identical categories: `"Yes"` / `"yes"` / `"Y"` / `"YES"`, or `"NY"` / `"New York"`.

---

## 6. Value Constraints

Check column-specific business rules:

- **Negative values in positive-only columns** (age, price, quantity)
- **Out-of-range values** (age > 120, percentage > 100, date in the future for historical data)
- **Cardinality surprises** — column expected to have few unique values has hundreds (or vice versa)
- **Zero values** — are they valid or missing-data placeholders?

```python
df[df['age'] < 0]
df[df['age'] > 120]
df[df['price'] < 0]
df['category'].nunique()
```

---

## 7. Cross-Field Validation

Check relationships between columns that should be consistent:

- **Start date ≤ end date** (project timelines, contracts)
- **City ↔ Country** consistency (or city ↔ zip code)
- **Subtotals summing to total** (invoice line items)
- **Conditional required fields** (if `order_shipped=True`, then `ship_date` must not be null)
- **Unique constraint violations** — composite key that should be unique isn't

```python
# Example: start > end
df[df['start_date'] > df['end_date']]

# Example: shipped but no ship date
df[df['shipped'] & df['ship_date'].isna()]
```

---

## Guiding principles

- **Show before you fix.** Always print the affected rows so the user can judge.
- **One category at a time.** Don't skip ahead. The user decides per category.
- **Skip is a valid answer.** Note the skip in the notebook and move on.
- **If none found, say so.** *"No duplicates detected in this dataset."* is still useful information.
- **Ask about domain knowledge.** A value that looks like an outlier might be normal in the user's domain.
