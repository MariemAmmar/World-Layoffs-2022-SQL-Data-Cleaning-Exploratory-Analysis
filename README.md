# 🏢 World Layoffs 2022 — SQL Data Cleaning & Exploratory Analysis

An end-to-end SQL project that cleans and explores a global dataset of tech and startup layoffs, uncovering which companies, industries, and countries were hit hardest — using advanced MySQL techniques including CTEs, window functions, and rolling aggregations.

---

## 📌 Project Overview

This project analyzes real-world layoff data sourced from [Kaggle](https://www.kaggle.com/datasets/swaptr/layoffs-2022). The raw dataset contains messy, duplicate, and inconsistent records that are first cleaned into a production-ready staging table, then explored to identify layoff trends by company, industry, country, and time period. The project follows a professional data cleaning workflow before diving into exploratory analysis — mirroring real analyst workflows.

---

## 📁 Repository Structure

```
├── Portfolio_Project_-_Data_Cleaning.sql   # Data cleaning queries
└── Portfolio_Project_-_EDA.sql             # Exploratory data analysis queries
```

> **Dataset:** [Layoffs 2022 — Kaggle](https://www.kaggle.com/datasets/swaptr/layoffs-2022)
> Import as `world_layoffs.layoffs` before running the scripts.

---

## 🗄️ Dataset

**Source Table:** `world_layoffs.layoffs`

| Field | Description |
|---|---|
| `company` | Company name |
| `location` | City/location of the company |
| `industry` | Industry sector |
| `total_laid_off` | Total number of employees laid off |
| `percentage_laid_off` | Percentage of workforce laid off (1 = 100%) |
| `date` | Date of the layoff event |
| `stage` | Company funding stage (Series A, B, Post-IPO, etc.) |
| `country` | Country of the company |
| `funds_raised_millions` | Total funds raised (in millions USD) |

---

## 🧹 Part 1 — Data Cleaning

**File:** `Portfolio_Project_-_Data_Cleaning.sql`

### Step 1 — Create Staging Table
- Created `layoffs_staging` as an exact copy of the raw table using `CREATE TABLE ... LIKE`
- All cleaning work is done on the staging table — raw data is preserved untouched
- Best practice: never modify the original source data directly

### Step 2 — Remove Duplicates
- First pass: used `ROW_NUMBER() OVER (PARTITION BY company, industry, total_laid_off, date)` to flag potential duplicates
- Manually verified flagged rows (e.g. Oda) — confirmed some were legitimate separate entries
- Second pass: expanded `PARTITION BY` to all 9 columns for a precise duplicate check
- Created `layoffs_staging2` with an added `row_num` column to safely delete rows where `row_num >= 2`
- Final `ALTER TABLE` drops the helper `row_num` column after cleaning

### Step 3 — Standardize Data (5 fixes applied)

| Issue | Fix |
|---|---|
| Blank `industry` values | Converted blanks to `NULL` first, then populated via self-join on company name |
| Crypto industry inconsistency | Consolidated `'Crypto Currency'` and `'CryptoCurrency'` → `'Crypto'` |
| Country name trailing period | `TRIM(TRAILING '.' FROM country)` fixed `'United States.'` → `'United States'` |
| Date stored as text | `STR_TO_DATE(date, '%m/%d/%Y')` converted string to proper DATE format |
| Date column type | `ALTER TABLE MODIFY COLUMN date DATE` enforced correct data type |

### Step 4 — Handle Null Values
- Reviewed NULL values in `total_laid_off`, `percentage_laid_off`, and `funds_raised_millions`
- Intentionally retained meaningful NULLs — they represent missing data, not errors
- Deleted rows where **both** `total_laid_off` AND `percentage_laid_off` are NULL — these rows carry no analytical value

---

## 🔍 Part 2 — Exploratory Data Analysis

**File:** `Portfolio_Project_-_EDA.sql`

### Basic Exploration
- Retrieved `MAX(total_laid_off)` to identify the single largest layoff event
- Checked `MAX` and `MIN` of `percentage_laid_off` to understand layoff severity range
- Filtered `percentage_laid_off = 1` to find companies that **laid off 100% of their workforce** — predominantly startups that went out of business
- Sorted those companies by `funds_raised_millions DESC` to reveal high-profile failures (e.g. Quibi raised ~$2B and shut down completely)

### Group-By Analysis
- **By company:** `SUM(total_laid_off)` grouped by company to find top 10 hardest-hit companies
- **By location:** Top 10 cities by total layoffs
- **By country:** Total layoffs per country ranked descending
- **By year:** Annual layoff totals using `YEAR(date)` — reveals which years were worst
- **By industry:** Total layoffs per sector to identify most-affected industries
- **By funding stage:** Layoffs by company stage (Seed, Series A–E, Post-IPO, etc.)

### Advanced Analysis — CTEs & Window Functions

**Top 3 Companies per Year**
```sql
WITH Company_Year AS (
  SELECT company, YEAR(date) AS years, SUM(total_laid_off) AS total_laid_off
  FROM layoffs_staging2
  GROUP BY company, YEAR(date)
),
Company_Year_Rank AS (
  SELECT company, years, total_laid_off,
    DENSE_RANK() OVER (PARTITION BY years ORDER BY total_laid_off DESC) AS ranking
  FROM Company_Year
)
SELECT company, years, total_laid_off, ranking
FROM Company_Year_Rank
WHERE ranking <= 3 AND years IS NOT NULL
ORDER BY years ASC, total_laid_off DESC;
```
- Uses **two chained CTEs** + `DENSE_RANK()` window function
- Returns the top 3 companies by layoffs for each year in the dataset

**Rolling Monthly Layoff Total**
```sql
WITH DATE_CTE AS (
  SELECT SUBSTRING(date,1,7) AS dates, SUM(total_laid_off) AS total_laid_off
  FROM layoffs_staging2
  GROUP BY dates
)
SELECT dates, SUM(total_laid_off) OVER (ORDER BY dates ASC) AS rolling_total_layoffs
FROM DATE_CTE
ORDER BY dates ASC;
```
- Computes a **running/cumulative total** of layoffs month by month
- Uses `SUM() OVER (ORDER BY dates)` window function for the rolling calculation
- Reveals the pace and acceleration of layoffs over time

---

## 🛠️ Tools & Technologies

- **Database:** MySQL 8.0+
- **Language:** SQL
- **Techniques:** Staging tables, `ROW_NUMBER()` deduplication, CTEs, `DENSE_RANK()`, rolling window functions (`SUM() OVER`), self-joins for NULL imputation, `STR_TO_DATE()`, `TRIM()`, `ALTER TABLE`, `GROUP BY`, `HAVING`, subqueries

---

## 💡 Key Findings

- Some companies laid off **100% of their workforce** — mostly startups that shut down entirely, including one that had raised ~$2 billion in funding
- **Crypto industry** had inconsistent naming across records — consolidated into a single label during cleaning
- The dataset reveals clear **year-over-year acceleration** in layoffs, visible through the rolling monthly total
- **Post-IPO companies** account for a surprisingly high share of total layoffs — larger companies shedding headcount, not just failing startups
- The **United States** dominates total layoff counts globally by a significant margin
- Rolling totals expose specific months where layoffs **spiked sharply** — correlating with macro events like rate hikes and market corrections

---

## 🚀 Getting Started

1. Clone this repository
2. Download the dataset from [Kaggle](https://www.kaggle.com/datasets/swaptr/layoffs-2022) and import as `world_layoffs.layoffs`
3. Run the cleaning script: `Portfolio_Project_-_Data_Cleaning.sql`
4. Run the EDA script: `Portfolio_Project_-_EDA.sql`

> Requires **MySQL 8.0+** for window functions (`ROW_NUMBER`, `DENSE_RANK`, `SUM OVER`).

---

## 📄 License

This project is open source and available under the [MIT License](LICENSE).
