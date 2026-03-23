# Walmart E-Commerce Data Pipeline

A Python-based ETL data pipeline built to analyze Walmart's e-commerce supply and demand patterns around public holidays. This project merges sales data from a PostgreSQL database with complementary feature data, transforms and aggregates it, and exports clean CSV outputs.

---

## Project Background

Walmart is the biggest retail store in the United States. By end of 2022, e-commerce represented **$80 billion in sales** — 13% of total company revenue. Public holidays like the Super Bowl, Labour Day, Thanksgiving, and Christmas significantly impact weekly sales.

This pipeline was built to support analysis of those holiday-driven patterns.

---

## Project Structure

```
walmart-data-pipeline/
│
├── notebook.ipynb          # Main Jupyter notebook (10 cells)
├── extra_data.parquet      # Complementary features dataset
├── clean_data.csv          # Output: cleaned & transformed data
├── agg_data.csv            # Output: average monthly sales
└── README.md
```

---

## Data Sources

### `grocery_sales` (PostgreSQL table)
| Column | Description |
|--------|-------------|
| `index` | Unique row ID |
| `Store_ID` | Store number |
| `Date` | Week of sales |
| `Weekly_Sales` | Sales for that store/week |

### `extra_data.parquet`
| Column | Description |
|--------|-------------|
| `IsHoliday` | 1 if week contains a public holiday, 0 if not |
| `Temperature` | Temperature on the day of sale |
| `Fuel_Price` | Cost of fuel in the region |
| `CPI` | Consumer Price Index |
| `Unemployment` | Prevailing unemployment rate |
| `MarkDown1–4` | Number of promotional markdowns |
| `Dept` | Department number in each store |
| `Size` | Size of the store |
| `Type` | Type of store (based on Size) |

---

## Pipeline Steps

The pipeline is implemented across **10 notebook cells**:

### Cell 1 — SQL Query
Fetches all data from the `grocery_sales` PostgreSQL table.
```sql
SELECT * FROM grocery_sales
```

### Cell 2 — `extract()`
Reads `extra_data.parquet` and merges it with `grocery_sales` on the `index` column.
```python
def extract(store_data, extra_data):
    extra_df = pd.read_parquet(extra_data)
    merged_df = store_data.merge(extra_df, on="index")
    return merged_df

merged_df = extract(grocery_sales, "extra_data.parquet")
```

### Cell 3 — `transform()` (defined)
Cleans and reshapes the merged data:
- Fills missing numerical values with the **column median**
- Adds a `Month` column extracted from the `Date` field
- Keeps only rows where `Weekly_Sales > 10,000`
- Selects only the 7 required columns
```python
def transform(raw_data):
    raw_data.fillna(raw_data.select_dtypes(include='number').median(), inplace=True)
    raw_data["Month"] = pd.to_datetime(raw_data["Date"]).dt.month
    raw_data = raw_data[raw_data["Weekly_Sales"] > 10000]
    raw_data = raw_data[["Store_ID", "Month", "Dept", "IsHoliday",
                          "Weekly_Sales", "CPI", "Unemployment"]]
    return raw_data
```

### Cell 4 — `transform()` (called)
```python
clean_data = transform(merged_df)
```

### Cell 5 — `avg_weekly_sales_per_month()` (defined)
Calculates average weekly sales per calendar month using a pandas method chain.
```python
def avg_weekly_sales_per_month(clean_data):
    agg_data = (clean_data[["Month", "Weekly_Sales"]]
                .groupby("Month")
                .agg(Avg_Sales=("Weekly_Sales", "mean"))
                .reset_index()
                .round(2))
    return agg_data
```

### Cell 6 — `avg_weekly_sales_per_month()` (called)
```python
agg_data = avg_weekly_sales_per_month(clean_data)
```

### Cell 7 — `load()` (defined)
Saves both DataFrames as CSV files without the row index.
```python
def load(full_data, full_data_file_path, agg_data, agg_data_file_path):
    full_data.to_csv(full_data_file_path, index=False)
    agg_data.to_csv(agg_data_file_path, index=False)
```

### Cell 8 — `load()` (called)
```python
load(clean_data, "clean_data.csv", agg_data, "agg_data.csv")
```

### Cell 9 — `validation()` (defined)
Verifies that output files exist in the current working directory.
```python
def validation(file_path):
    return os.path.exists(file_path)
```

### Cell 10 — `validation()` (called)
```python
validation("clean_data.csv")   # Expected: True
validation("agg_data.csv")     # Expected: True
```

---

## Output Schemas

### `clean_data.csv`
| Column | Description |
|--------|-------------|
| `Store_ID` | Unique store identifier |
| `Month` | Calendar month (1–12) |
| `Dept` | Department number |
| `IsHoliday` | Holiday week flag |
| `Weekly_Sales` | Weekly sales (filtered > $10,000) |
| `CPI` | Consumer Price Index |
| `Unemployment` | Unemployment rate |

### `agg_data.csv`
| Column | Description |
|--------|-------------|
| `Month` | Calendar month (1–12) |
| `Avg_Sales` | Average weekly sales for that month (2 decimal places) |

Sample output:
| Month | Avg_Sales |
|-------|-----------|
| 1.0 | 33174.18 |
| 2.0 | 34333.33 |
| ... | ... |

---

## Tech Stack

| Tool | Purpose |
|------|---------|
| Python 3 | Core programming language |
| Pandas | Data manipulation and transformation |
| SQL / PostgreSQL | Source data extraction |
| Parquet | Efficient columnar data storage |
| os | File system validation |
| Jupyter Notebook | Interactive development environment |

---

## How to Run

1. Clone the repository:
   ```bash
   git clone https://github.com/your-username/walmart-data-pipeline.git
   cd walmart-data-pipeline
   ```

2. Install dependencies:
   ```bash
   pip install pandas pyarrow psycopg2-binary jupyter
   ```

3. Open the notebook:
   ```bash
   jupyter notebook notebook.ipynb
   ```

4. Run all cells in order (Cell 1 → Cell 10).

5. Check outputs:
   ```
   clean_data.csv
   agg_data.csv
   ```

## 📝 License

This project was completed as part of a DataCamp data engineering learning path.
