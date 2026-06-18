# 📉 World Layoffs: Data Engineering & Exploratory Data Analysis Using MYSQL

## 📌 Project Overview
This project presents an end-to-end SQL workflow focused on **Data Engineering Pipelines** and **Exploratory Data Analysis (EDA)** using a comprehensive dataset of global tech layoffs since the COVID-19 pandemic. The lifecycle spans from engineering a raw, unformatted data dump into a production-ready analytical layer, followed by surfacing actionable macroeconomic insights about industry-wide contractions, funding impacts, and corporate workforce reductions.

* **Dataset Source:** [Kaggle World Layoffs Dataset (2020 - Present)](https://www.kaggle.com/datasets/swaptr/layoffs-2022)
* **Database Platform:** MySQL / MySQL Workbench
* **Core Competencies:** Advanced SQL, Data Pipelines, Window Functions, Common Table Expressions (CTEs), Schema Modification, Data Imputation.

---

## 📂 Repository Structure
To maintain production-grade data pipeline standards, the SQL scripts are modularly divided into separate structural layers:
```text
📦 World_Layoffs_Data_Engineering_and_EDA_using_MYSQL
 ┣ 📂 sql_scripts
 ┃ ┣ 📜 1_data_cleaning.sql            # Staging, Deduplication, and Schema Modification
 ┃ ┗ 📜 2_exploratory_data_analysis.sql # Trend Analysis, Aggregations, and Rank Partitions
 ┗ 📜 README.md                         # Project documentation and key insights
```

---

## 🛠️ Phase 1: Data Engineering Pipeline (`1_data_cleaning.sql`)
A robust data cleaning workflow was established to ensure absolute data integrity. Because modifying raw production tables directly introduces substantial risk, a multi-tier staging environment was implemented.

### 1. Removing Duplicate Entries
The dataset lacked a primary key, making exact duplicate entries a challenge. 
* **The Strategy:** Rows were partitioned across all descriptive dimensions (Company, Location, Industry, Total Laid Off, Date, Country, etc.) using the `ROW_NUMBER()` window function.
* **The Fix:** Since MySQL does not support direct `DELETE` targets from a standard window partition subquery, a secondary staging table (`layoffs_staging2`) was dynamically engineered with a structural row-index flag column (`row_num`). All records where `row_num >= 2` were systematically dropped.

### 2. Standardizing Data & Fixing Anomalies
* **Industry Alignment:** Resolved multi-naming variances within bleeding-edge tech spaces, programmatically merging inconsistent terminology like `'Crypto Currency'` and `'CryptoCurrency'` into a standardized `'Crypto'` category.
* **Trailing Artifacts:** Cleaned geo-locational strings by removing localized trailing punctuation using `TRIM(TRAILING '.' FROM country)`.
* **Temporal Transformations:** Converted loose textual strings into indexed date objects using `STR_TO_DATE()`, subsequently casting the column via `ALTER TABLE ... MODIFY COLUMN ... DATE` to support time-series analysis.

### 3. Missing Value Imputation
* Empty string blanks (`''`) were converted to database-native `NULL` indicators for uniform handling.
* Performed an internal self-join (`JOIN` on matching `company` fields) to cross-reference historical company profiles and automatically backfill missing operational attributes (e.g., auto-populating missing `'Travel'` fields into blank `Airbnb` records).

### 4. Structural Filtering
* Safely dropped records containing missing values across both primary metrics (`total_laid_off` and `percentage_laid_off`), as these dead records provided no analytic utility.
* Stripped out the internal operational `row_num` tracker column to finish with a clean, high-performance dataset.

---

## 📊 Phase 2: Exploratory Data Analysis (`2_exploratory_data_analysis.sql`)
With a clean analytical layer, EDA was executed to isolate overarching trend dynamics, corporate workforce trajectory vectors, and funding anomalies.

### 🔍 Key Analytical Highlights & SQL Techniques

#### 1. Volatility Metrics & Startup Failures
* Analyzed funding structures relative to complete business closures (`percentage_laid_off = 1`). High-profile, heavily funded ventures were uncovered as structural outliers (e.g., Quibi collapsing despite raising ~$2 Billion).

#### 2. Advanced Enterprise Performance Ranks Per Year
To surface top-tier corporate layoffs chronologically without blending timelines, a multi-layer CTE structure combined with a `DENSE_RANK()` matrix was engineered:
```sql
WITH Company_Year AS (
  SELECT company, YEAR(date) AS years, SUM(total_laid_off) AS total_laid_off
  FROM layoffs_staging2
  GROUP BY company, YEAR(date)
), Company_Year_Rank AS (
  SELECT company, years, total_laid_off, DENSE_RANK() OVER (PARTITION BY years ORDER BY total_laid_off DESC) AS ranking
  FROM Company_Year
)
SELECT company, years, total_laid_off, ranking
FROM Company_Year_Rank
WHERE ranking <= 3 AND years IS NOT NULL
ORDER BY years ASC, total_laid_off DESC;
```
*This query dynamically partitions time horizons to isolate exactly the top 3 companies causing the highest workforce impact for each individual calendar year.*

#### 3. Rolling Month-Over-Month Macro Trends
Calculated a cumulative running total tracking global job contractions across historical timelines to understand market velocity:
```sql
WITH DATE_CTE AS (
  SELECT SUBSTRING(date,1,7) as dates, SUM(total_laid_off) AS total_laid_off
  FROM layoffs_staging2
  GROUP BY dates
  ORDER BY dates ASC
)
SELECT dates, SUM(total_laid_off) OVER (ORDER BY dates ASC) as rolling_total_layoffs
FROM DATE_CTE
ORDER BY dates ASC;
```

---

## 📈 Summary of Data Insights
1. **The Post-COVID Market Correction:** Tech layoffs experienced sharp spikes starting late 2022, signaling an industry-wide stabilization phase following aggressive pandemic-era hyper-growth.
2. **Impacted Sectors:** Consumer-centric tech segments, Retail, and Transport/Travel sectors absorbed the heaviest workforce reductions.
3. **Capital Instability:** High venture capital backing did not guarantee immunity; massive cash reserves frequently preceded total business liquidation in unstable market climates.
4. **Geographic Concentrations:** Global tech clusters within the United States (specifically Silicon Valley/SF Bay Area and New York City) represented the absolute highest volume of global workplace disruptions.

---

## 🚀 How to Run This Project Local
1. Clone this repository to your system.
2. Download the baseline data archive from [Kaggle](https://www.kaggle.com/datasets/swaptr/layoffs-2022).
3. Open your SQL client (e.g., MySQL Workbench) and initialize a local database schema named `world_layoffs`.
4. Import the raw CSV data into a baseline system table named `layoffs`.
5. Run `sql_scripts/1_data_cleaning.sql` to clean the schema data structure.
6. Run `sql_scripts/2_exploratory_data_analysis.sql` to generate insights and view metrics.
