# Pandas Cleaning Patterns

Curated snippets for common data cleaning operations. Use these as a quick reference when implementing fixes in the notebook.

---

## Missing Values

```python
# Count nulls per column
df.isna().sum()

# Percentage of nulls
df.isna().mean().round(4) * 100

# Drop rows with any null
df.dropna()

# Drop rows where specific column is null
df.dropna(subset=['col1', 'col2'])

# Drop columns that are entirely null
df.dropna(axis=1, how='all')

# Fill with a constant
df['col'].fillna(0)
df['col'].fillna('Unknown')

# Fill with summary statistic
df['col'].fillna(df['col'].mean())
df['col'].fillna(df['col'].median())
df['col'].fillna(df['col'].mode()[0])

# Forward / backward fill (time series)
df['col'].fillna(method='ffill')
df['col'].fillna(method='bfill')

# Interpolate (time series, ordered)
df['col'].interpolate()

# Flag that a value was originally missing
df['col_missing'] = df['col'].isna()
```

---

## Duplicates

```python
# Count duplicates
df.duplicated().sum()

# View duplicate rows
df[df.duplicated(keep=False)].sort_values(by=list(df.columns))

# Drop duplicates (keep first occurrence)
df.drop_duplicates()

# Drop duplicates based on subset of columns
df.drop_duplicates(subset=['email', 'name'])

# Keep last occurrence
df.drop_duplicates(keep='last')

# Drop all rows that have any duplicate (keep=False removes all copies)
df.drop_duplicates(keep=False)
```

---

## Type Conversion

```python
# Convert column to numeric (coerce errors to NaN)
df['col'] = pd.to_numeric(df['col'], errors='coerce')

# Convert to datetime (coerce errors to NaT)
df['date'] = pd.to_datetime(df['date'], errors='coerce')

# Custom datetime format
df['date'] = pd.to_datetime(df['date'], format='%Y-%m-%d', errors='coerce')

# Convert to string
df['col'] = df['col'].astype(str)

# Convert to category (saves memory for low-cardinality strings)
df['col'] = df['col'].astype('category')

# Convert to boolean (handles True/False, 1/0, yes/no)
df['col'] = df['col'].map({'yes': True, 'no': False, 'y': True, 'n': False})

# Check inferred types
df.dtypes

# Find non-numeric values in object column
df[pd.to_numeric(df['col'], errors='coerce').isna() & df['col'].notna()]
```

---

## String Cleaning

```python
# Strip whitespace
df['col'] = df['col'].str.strip()

# Lowercase
df['col'] = df['col'].str.lower()

# Title case
df['col'] = df['col'].str.title()

# Replace substring
df['col'] = df['col'].str.replace(r'\s+', ' ', regex=True)  # collapse multiple spaces

# Remove specific characters
df['col'] = df['col'].str.replace(r'[^\w\s]', '', regex=True)  # remove punctuation

# Extract pattern
df['col'] = df['col'].str.extract(r'(\d+)', expand=False)  # extract digits

# Split into multiple columns
df[['first', 'last']] = df['full_name'].str.split(' ', n=1, expand=True)

# Check for leading/trailing whitespace
df['col'].str.match(r'^\s|\s$').sum()

# Find rows matching a pattern
df[df['col'].str.contains(r'\d{5}', na=False)]  # rows with 5-digit pattern (zip codes)
```

---

## Date Cleaning

```python
# Standardise date format in output
df['date'] = pd.to_datetime(df['date']).dt.strftime('%Y-%m-%d')

# Extract components
df['year'] = pd.to_datetime(df['date']).dt.year
df['month'] = pd.to_datetime(df['date']).dt.month
df['quarter'] = pd.to_datetime(df['date']).dt.quarter

# Find unparseable dates
mask = pd.to_datetime(df['date'], errors='coerce').isna()
df[mask & df['date'].notna()]

# Fix common date format issues
# "Jan 5, 2024" → standard
df['date'] = pd.to_datetime(df['date'].str.replace(r'(\w{3})\s+(\d{1,2}),?\s+(\d{4})',
                                                     r'\1 \2 \3', regex=True))

# Parse dates stored as YYYYMMDD integer
df['date'] = pd.to_datetime(df['date'].astype(str), format='%Y%m%d', errors='coerce')
```

---

## Outlier Detection

```python
# IQR method
Q1 = df['col'].quantile(0.25)
Q3 = df['col'].quantile(0.75)
IQR = Q3 - Q1
lower = Q1 - 1.5 * IQR
upper = Q3 + 1.5 * IQR
outliers = df[(df['col'] < lower) | (df['col'] > upper)]

# Z-score method (requires scipy)
from scipy import stats
z = np.abs(stats.zscore(df['col'].dropna()))
outliers_z = df.loc[df['col'].dropna().index[z > 3]]

# Winsorize / cap (replace extremes with threshold values)
df['col'] = df['col'].clip(lower=lower, upper=upper)

# Visual check via box plot
df[['col']].boxplot()

# Percentile-based capping (less sensitive to extremes)
lower_p = df['col'].quantile(0.01)
upper_p = df['col'].quantile(0.99)
df['col'] = df['col'].clip(lower=lower_p, upper=upper_p)
```

---

## Filtering and Selection

```python
# Filter rows by condition
df[df['col'] > 0]
df.query('col > 0')  # same, sometimes more readable

# Multiple conditions
df[(df['col1'] > 0) & (df['col2'] == 'active')]

# Filter by list
df[df['col'].isin(['a', 'b', 'c'])]

# Exclude rows matching condition
df[~df['col'].isna()]

# Select columns by dtype
df.select_dtypes(include='number')
df.select_dtypes(include='object')

# Sample for inspection
df.head(10)
df.sample(5)
```

---

## Renaming and Reordering

```python
# Rename specific columns
df.rename(columns={'old_name': 'new_name', 'Old Name': 'old_name'})

# Lowercase all column names
df.columns = df.columns.str.lower().str.replace(' ', '_')

# Reorder columns
df = df[['col_a', 'col_b', 'col_c']]

# Drop columns
df.drop(columns=['col_to_drop'])
```

---

## Combining Checklists

Many cleaning steps depend on each other. Suggested order of operations:

1. **Standardise column names** (so you can reference them reliably)
2. **Fix types** (so null detection and filtering work correctly)
3. **Handle duplicates** (so you don't analyse the same row twice)
4. **Handle missing values** (after types are correct)
5. **Fix formatting** (string normalisation, date standardisation)
6. **Handle outliers** (after missing values are addressed)
7. **Cross-field validation** (last — relies on everything above being correct)

This is a suggestion, not a rule. Deviate when the dataset or user preference calls for it.
