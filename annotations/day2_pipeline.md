# Day 2 — Medicaid ITS Pipeline Annotations
_(Anchored to `Day 1 Clean.py` — Sep 2024 cap, T-MSIS reporting-lag exclusion)_

## Pipeline in my own words

The pipeline behind analysing the impact of ending of Medicaid's continuous enrollment provision in 2023 upon chronic healthcare utilization begins with loading up a more efficient parquet file format of HHS Medicaid Provider Spending (T-MSIS) using a polars lazy pipeline. The file is then lazy scanned and filtered for HCPCS codes (codes that point towards specific clinical procedures and tests) and claim months. After grouping by HCPCS codes and months using .group_by, .agg sums up the total patients and total claim lines for each of these codes and months before commencing with the .collect method. A verification that there is no inclusion of data past 2024-09 is done to ensure a complete data set, since there is claims reporting lag with T-MSIS in the last quarter (late-filed claims for Oct–Dec 2024 haven't arrived yet, so including those months would falsely show a utilization drop that is actually missing data, not missing care). The KFF enrollment data is used per month to enable per Medicaid enrollee analysis through a rate per 1,000. This allows for understanding if the decrease in chronic care utilization is an effect of just pure decrease in enrollment or true decrease per enrollee. This data is then fitted with a segmented OLS interrupted time series to estimate the level shift and slope change at April 2023, the first month states could begin disenrollments.

---

## 1. pathlib.Path (lines 66, 75–87)
**What:** Provides streamlined syntax to follow same path as established in line 75 with path method designated as BASE_DIR
**Idiom:** object-oriented filesystem paths
**Why (vs. string paths + os.path.join):** Path has methods innately attached to it and also has filesystem operations that are automated based on the system. 

## 2. Polars lazy pipeline (lines 149–162)
**What:** Lazy scanner that uses polars library to filter down to parquet reader itself based on the specific logic asked (i.e., specific HCPCS codes we have in TRACER_CODES)
**Idiom:** lazy query with predicate pushdown + terminal .collect()
**Why (vs. read_parquet + eager filter):** This lazy scanner enables a much more efficient and less memory intensive method of pulling data we need, instead of loading up the multiple GB dataset 
**Timing from Supplement A:** lazy: 0.36s, shape (84, 2)

## 3. Polars → pandas handoff (line 170)
**What:** Conversion from polars to pandas structure since heavy lift of extracting necessary data from huge dataset is done, allowing for easier data analysis ecosystem with pandas
**Idiom:** hybrid data pipeline (fast tool for heavy lift, ecosystem-rich tool for analysis)
**Why (vs. staying in polars end-to-end):** Pandas is a more data analysis friendly environment that can now be used after polars data extraction from large dataset is done 

## 4. Boolean masking + .copy() (line 225)
**What:** Filters through boolean indexing the rows with True if the location does match a state and then makes an independent copy via .copy that serves as new reference file
**Idiom:** defensive-copy boolean filter
**Why (vs. implicit view):** The .copy enables a new raw file that can be mutated without touching the original raw file. Boolean indexing ensures only true marked rows where the location matches established states

## 5. melt(): wide → long (lines 230–231)
**What:** Takes KFF table where the months are column names in wide format to long format where headers are state, month, and enrollment. Also enables this is done for "__Medicaid Enrollment" and not including "__CHIP Enrollment"
**Idiom:** wide-to-long reshape (a.k.a. "tidying")
**Why (vs. keeping it wide and iterating over columns):** The new long format ensures months is its own column and can be grouped, counted and plotted against for enrollment rather than iterating over each column 

## 6. The .str accessor chain (lines 239–245)
**What:** Takes the "enrollment" row and removes all commas, strips out leading or trailing white space and replaces any "N/A"s with np.nan which are then dropped later. After all this it changes the enrollment strings to floats to be able to use mathematically
**Idiom:** vectorized string-cleanup chain (accessor chaining)
**Why (vs. `.apply(lambda x: ...)` row-by-row):** .str removes the commas in a single pass across whole column rather than row by row running of code done by lambda

## 7. Cache-and-skip pattern (lines 147, 218, 297)
**What:** Looks for if the CSV file is loaded up or not before running phase so as to not redundantly load it up again and instead use existing loaded up CSV. 
**Idiom:** file-based memoization (a.k.a. idempotent materialization)
**Why (vs. re-running everything every time):** Makes the process much more efficient when running code rather than waiting multiple minutes for each run despite already having a CSV loaded in

---

## Questions for Day 3

1. **`os.path.join` vs `pathlib`:** How does `os.path.join` actually work, and what specifically does `pathlib.Path` give me that string concatenation + `os.path.join` doesn't? What does "cross-platform path handling" mean in practice (Windows vs macOS slashes, etc.)?

2. **Joining tables of different shapes to compute a derived rate:** What are the patterns for combining two tables of different shapes (e.g., a long-format monthly time series joined to per-month aggregates) to compute a derived per-capita metric? How do I choose between `pd.merge`, `.map(dict)`, and `.map(Series)` in future projects? When does each one win on clarity, speed, or memory?

3. **View vs copy in pandas:** What's the actual difference between a view and a copy in pandas? When does an operation return a view vs a copy, and how does `SettingWithCopyWarning` connect to the defensive `.copy()` pattern from focal point 4? What can silently break if I forget the `.copy()`?

4. **Predicate pushdown at the parquet level:** What does predicate pushdown actually do inside the parquet reader — is it filtering rows after the file is read, or filtering during the read itself? How does parquet's columnar + row-group structure (with min/max statistics per row group) make this possible? This is the underlying reason Supplement A clocked the lazy query at 0.36s on a multi-GB file — I want to understand the file-format mechanics, not just the polars API.

---

**Note for Claude:** Use the understanding conveyed in this .md file to calibrate future explanations. 