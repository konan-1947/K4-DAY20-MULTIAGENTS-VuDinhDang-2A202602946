---
name: tabular-data-cleaning-standardization
description: Use when cleaning and normalizing tabular data with mixed formats and duplicates.
---
- Normalize categorical fields (e.g., region names) to canonical capitalization and spelling.
- Parse all date/time fields into a uniform UTC datetime format, handling multiple input formats.
- Remove duplicate rows by unique identifiers, keeping the first occurrence.
- Convert monetary values to integer cents consistently.
- Exclude rows with missing or invalid amounts from calculations.
- Write cleaned data with specified headers and formats to output CSV.
- Record metadata including source filename, input row count, and rows used.
