# Electronics Sales Analysis

A small Python project that reads a messy electronics sales CSV, cleans the numeric columns, and reports revenue figures in Kenya shillings (KES). The results are cross-checked against using pandas to clean the data.

## What it does

1. **Reads** `data/electronics_sales.csv` with the `csv` module (every value arrives as a string).
2. **Cleans** `quantity` and `unit_price_kes`, repairing what can be repaired safely and logging every row it skips, with a reason.
3. **Calculates**
   - total revenue
   - revenue by product, store and payment method (plain-dict grouping)
   - top 3 products
   - high-value sales (KES 50,000 or more), with a count per store
   - mean, median, minimum and maximum revenue per transaction (hand-written, no `statistics` module)
4. **Verifies** the headline numbers against an independent pandas calculation and prints `MATCH` / `MISMATCH` for each.

## Project layout

```
.
├── data/
│   └── electronics_sales.csv    # input
├── sales_analysis.ipynb         # all code
├── output/
│   └── skipped_rows.csv         # created on each run
└── README.md
```

Expected CSV columns: `sale_id`, `sale_date`, `store`, `product`, `quantity`, `unit_price_kes`, `payment_method`.

## How to run

### In a Jupyter notebook (.ipynb)
1. Open the console and run the command below to add all the needed packages for the program.
   ```python
   pip install -r requirements.txt
   ```
2. Put the imports, constants and functions from `sales_analysis.py` in one or more cells and run them.

3. In a new cell, call the entry point directly:

   ```python
   main()                                   # default path
   main(Path("data/electronics_sales.csv")) # or an explicit path
   ```

---
## Data cleaning decisions

The guiding rule: **repair a value only when the intended number is unambiguous; otherwise skip the row and say why.** Guessing silently risks corrupting the revenue figures, while skipping is visible and reversible.

| Raw value | Example | Decision | Why |
|---|---|---|---|
| Surrounding quotes / whitespace | `"38,999"`, ` 12 ` | **Repair** | The number is unambiguous. (Quotes from the CSV format are already removed by the `csv` module; extra quotes are stripped as a safety net.) |
| `KES` prefix | `KES 18999` | **Repair** | Currency label only; case-insensitive. |
| Thousands comma | `38,999` | **Repair** | Only when commas form proper groups of three digits. |
| Missing / empty | `""` | **Skip** | No value to repair. |
| Placeholders | `N/A` | **Skip** | Explicitly "no data". |
| Text | `two` | **Skip** | Not numeric (see assumptions below on `two`). |

A row is skipped if *either* `quantity` or `unit_price_kes` fails. The reason names the first failing field, and every skipped row is recorded with its CSV line number (header = line 1), its `sale_id`, and the reason. The log is printed and written to `output/skipped_rows.csv`.

### Other design choices

- **Revenue** for a row is `quantity × unit_price_kes`.
- **Tie-breaking in rankings:** `top_n` sorts by `(-revenue, name)`, so equal revenues are ordered alphabetically. The ranking never depends on row order in the file.
- **High-value threshold** is inclusive (`>=` KES 50,000). The per-store counts use a dict comprehension over *all* stores, so a store with zero qualifying sales shows `0` rather than disappearing.
- **Statistics** functions raise a clear `ValueError` on an empty list. `median` sorts a copy, so the caller's list isn't reordered; `minimum_maximum` does a single pass without `min()` / `max()`.
- **Verification** re-reads the file with `dtype=str, keep_default_na=False`, so pandas doesn't convert `N/A` or blanks on its own, and applies the same cleaning rules with  string operations.

## Assumptions

- Column names are exactly `quantity` and `unit_price_kes`, and all prices are in KES.
- Each CSV row is one transaction since in the data provided the `sale_id` values are unique.
- Skipped rows are excluded from every total calculation.
- `two` (written as a word) is treated as invalid, although it is can be considered recoverable.

## Results on the supplied file

50 rows read, 46 used, 4 skipped.

| Line | sale_id | Reason |
|---|---|---|
| 7 | S-3006 | quantity is not a valid number (`'two'`) |
| 31 | S-3030 | unit_price_kes is a placeholder (`'N/A'`) |
| 36 | S-3035 | missing quantity |
| 40 | S-3039 | missing unit_price_kes |

Two rows were repaired: S-3017 (`"38,999"` → 38999.0) and S-3029 (`KES 18999` → 18999.0).

| Metric | Value |
|---|---|
| Total revenue | KES 1,310,812.00 |
| Top product | HP 15 Laptop (KES 389,994.00) |
| Top store | Kisumu Mega Plaza (KES 449,678.00) |
| Top payment method | M-Pesa (KES 738,568.00) |
| Sales of KES 50,000 or more | 5 |
| Mean revenue per sale | KES 28,495.91 |
| Median revenue per sale | KES 11,097.00 |
| Smallest / largest sale | KES 1,499.00 / KES 259,996.00 |

The mean is more than double the median because a few large sales (laptops, TVs, tablets) pull the average up, so the median is the better measure of a typical sale. All 10 pandas verification checks report `MATCH`.
