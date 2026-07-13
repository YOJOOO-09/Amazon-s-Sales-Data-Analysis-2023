# Amazon-s-Sales-Data-Analysis-2023

# Amazon Marketplace Performance Analysis

End-to-end analysis of a large Amazon product catalog to identify which product categories drive revenue, where product quality is actually broken versus just under-marketed, and where marketing spend should be scaled up or pulled back.

**Tools:** Python (Pandas, NumPy) · PostgreSQL · Power BI

---

## Overview

This project takes a raw, messy Amazon product export and turns it into three strategic decision frameworks that a business/marketing team could act on directly:

1. **Revenue vs. Volume Segmentation** — which categories are premium/low-volume vs. scale/high-volume.
2. **Rating Gap / Quality Diagnostic Analysis** — which categories have a real product-quality problem vs. a listing/marketing problem.
3. **Trust & Engagement Segmentation** — a weighted trust score combining rating and review volume, used to decide where to scale, fix, promote, or deprioritize spend.

Each phase produces an interactive Power BI dashboard (`.pbix`) and a written report (`.pdf`) summarizing findings and recommended actions.

---

## Dataset

**Amazon Sales Data 2023** — 551,585 products across 19 main categories and 112 sub-categories.

| Column | Description |
|---|---|
| `name` | Product name |
| `main_category` | Top-level category |
| `sub_category` | Finer-grained category |
| `discount_price` | Current/selling price |
| `actual_price` | List/original price |
| `ratings` | Average customer star rating |
| `no_of_ratings` | Number of ratings (proxy for engagement/demand) |
| `image` | Product image URL |
| `link` | Product page URL |

Note: this is catalog/listing data, not transactional sales data — there is no units-sold or order-count field. "Volume" is measured as product count per category, and "engagement" is approximated using `no_of_ratings`.

---

## Pipeline

### 1. Data cleaning — Python (Pandas)

- Loaded the raw CSV with pandas and previewed records to understand structure.
- Standardized numeric fields — stripped currency symbols, commas, and stray text from price columns using regex.
- Converted cleaned price/rating fields to numeric dtypes.
- Removed duplicate products using a defined business key (`name` + `main_category` + `sub_category`) instead of exact-row matching, since duplicate listings weren't always identical across every column (e.g. differing image URLs).
- Renamed columns for consistency before export.

```python
import pandas as pd
import re

df = pd.read_csv("amazon_sales_2023.csv")
df.columns = [c.strip().lower().replace(" ", "_") for c in df.columns]

def clean_price(x):
    if pd.isna(x):
        return None
    x = re.sub(r'[^\d.]', '', str(x))
    return float(x) if x else None

for col in ['discount_price', 'actual_price']:
    df[col] = df[col].apply(clean_price)

df['ratings'] = pd.to_numeric(df['ratings'], errors='coerce')
df['no_of_ratings'] = pd.to_numeric(df['no_of_ratings'], errors='coerce')

df = df.drop_duplicates(subset=['name', 'main_category', 'sub_category'])
df.to_csv("amazon_sales_2023_cleaned.csv", index=False)
```

### 2. Structured analysis — PostgreSQL

The cleaned CSV was loaded into a PostgreSQL table (`amazon_database`), then queried to compute category-level revenue, product counts, median ratings, median engagement, and a custom trust score.

**Trust Score** — combines rating quality with review volume so a product needs both to rank highly:

```sql
select main_category,
       sub_category,
       ROUND(AVG(Ratings * LOG(no_of_ratings)), 2) AS Trust_Score_subcat
from amazon_database
GROUP BY main_category, sub_category
order by trust_score_subcat DESC
limit 10;
```

**Low Engagement + Low Rating segment** (Deprioritize) — two CTEs, each computing a median via `PERCENTILE_CONT`, joined on `sub_category`:

```sql
WITH low_engagement AS (
    SELECT main_category, sub_category,
        ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY no_of_ratings)::numeric, 2) AS no_of_ratings_median
    FROM amazon_database
    GROUP BY main_category, sub_category
    LIMIT 50
),
low_rating AS (
    SELECT main_category, sub_category,
        ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY RATINGS)::numeric, 2) AS median_rating_subcat
    FROM amazon_database
    GROUP BY main_category, sub_category
    LIMIT 50
)
SELECT low_engagement.sub_category, low_engagement.main_category,
       low_engagement.no_of_ratings_median, low_rating.median_rating_subcat
FROM low_engagement
INNER JOIN low_rating ON low_engagement.sub_category = low_rating.sub_category
ORDER BY median_rating_subcat ASC, no_of_ratings_median ASC
LIMIT 10;
```

**High Engagement + Low Rating segment** (Fix First) — same CTE pattern, but each side is pre-filtered to its extreme (top 50 by engagement, bottom 50 by rating) before joining:

```sql
WITH high_engagement AS (
    SELECT main_category, sub_category,
        ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY no_of_ratings)::numeric, 2) AS no_of_ratings_median
    FROM amazon_database
    GROUP BY main_category, sub_category
    ORDER BY no_of_ratings_median DESC
    LIMIT 50
),
low_rating AS (
    SELECT main_category, sub_category,
        ROUND(PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY ratings)::numeric, 2) AS median_rating_subcat
    FROM amazon_database
    GROUP BY main_category, sub_category
    ORDER BY median_rating_subcat ASC
    LIMIT 50
)
SELECT high_engagement.sub_category, high_engagement.main_category,
       high_engagement.no_of_ratings_median, low_rating.median_rating_subcat
FROM high_engagement
INNER JOIN low_rating ON high_engagement.sub_category = low_rating.sub_category
ORDER BY high_engagement.no_of_ratings_median DESC, low_rating.median_rating_subcat ASC
LIMIT 10;
```

### 3. Dashboards — Power BI

Each phase's findings are published as an interactive `.pbix` file with slicers and cross-filtering by category, so any category can be drilled into live rather than read as a static chart.

---

## Findings

### Phase 1 — Revenue vs. Volume Segmentation

Categories segmented into four quadrants: High Revenue/Low Volume, Low Revenue/High Volume, High Revenue/High Volume, Low Revenue/Low Volume.

- **Television** generates ~₹1.7 lakh crore in revenue from only ~3,000 products — a premium, brand-driven, High Revenue/Low Volume market.
- Categories like men's/women's clothing and accessories move 200K+ products at much lower revenue per unit — High Volume, scale-driven markets.
- Strategic guidance per quadrant: High Rev/Low Vol → focus on niche expertise and retention; High Rev/High Vol → build competitive moats (most contested segment); Low Rev/Low Vol → pivot/reposition, may be an awareness gap rather than a demand gap.

### Phase 2 — Rating Gap / Quality Diagnostic Analysis

- Compared average rating at the sub-category level to separate stable, high-trust categories from consistently dissatisfied ones.
- Measured the rating **gap** within a main category across its sub-categories: large gaps (>0.05 variance) indicate structural issues (durability, supplier reliability) needing operational fixes; small gaps (<0.03 variance) indicate "last mile" issues (listing images, descriptions) that are cheap to fix.

### Phase 3 — Trust & Engagement Segmentation

Categories segmented by rating (satisfaction) x number of ratings (engagement/demand proxy) into four groups:

- **High Engagement / High Ratings — Scale:** increase inventory and marketing spend; predictable revenue.
- **High Engagement / Low Ratings — Fix First:** demand exists but experience is broken; diagnose negative reviews and fix core issues before buying more traffic.
- **Low Engagement / High Ratings — Unlock Demand:** good product, low visibility; invest in search placement and content rather than discounting.
- **Low Engagement / Low Ratings — Deprioritize:** stop paid promotion; investigate root cause before scaling.

---

## Repository Structure

```
├── Phase 1 Revenue & Volume Analysis.pbix
├── Phase 1  Marketplace category segmentation using revenue and sales volume.pdf
├── Phase 3 Engagement Analysis.pbix
├── Phase 3 Report Customer Rating Analysis of Amazon Product Categories.pdf
├── Phase 4 Engagement Analysis.pbix
├── Phase 4 Customer Trust & Engagement Segmentation using Ratings Data.pdf
└── README.md
```

---

## Limitations

- `no_of_ratings` is a proxy for demand, not actual sales volume — a product can accumulate reviews over years without being a current bestseller, and there's no timestamp data to normalize for that.
- The dataset is a single snapshot, not a time series, so trend direction (improving vs. declining) can't be measured directly.
- SQL logic in Phases 1–2 is summarized through dashboards/PDF rather than checked into this repo; only Phase 3's queries are included above.

---

## How to Reproduce

1. Clean the raw CSV with the Python script in `pipeline/clean.py` (see snippet above).
2. Load the cleaned CSV into a PostgreSQL database (`amazon_database` table).
3. Run the SQL queries above (or your own) to generate category-level aggregates.
4. Open the relevant `.pbix` file in Power BI Desktop, or upload it to [app.powerbi.com](https://app.powerbi.com) to view/interact with it in the browser.
