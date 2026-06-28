
1. You cleaned raw Marc Jacobs Beauty data into structured tables.
2. You loaded or created those tables in Snowflake.
3. You created analytics views/tables in Snowflake.
4. You connected Row Zero to Snowflake.
5. You pulled only the clean analytics tables into Row Zero.
6. You used Row Zero to inspect, pivot, and make quick tables.
7. You realized final storytelling charts may need Python because Row Zero/Snowflake are not enough for custom design.


-----
1. collect raw data from the internet
2026 - browsed marc jacobs website and sephora to get all the information i wanted + color picker for color information
2013 - browsed across temptalia, beautylish, ebay and poshmark listings, the beautylookbook to find images and data about original launch products + color picker

2. store all that messy data in Row Zero
   https://rowzero.com/workbook/6B4E2FC9167D2462CAACC1AA
3. use the ai agent in row zero to clean up any immediate stragglers i see like 
light normalization** after loading the raw notes into Row Zero

- **Split product-level data from shade-level data**  
    Your notes mix one product record with many shade records. For example, Drawn This Way has one product entry but 21 shade rows, while Heart On has one product entry but 20 shades. Those need to become separate tables:  
    `products_master` = one row per product  
    `shades_master` = one row per shade  
    
- **Add an `era` column**  
    Every row needed either:
    
    - `original_launch`
    - `revival`
    
    The 2013 notes say the original brand debuted with 120 products and include the original product set, while the 2026 file has 7 revival products.
    
- **Standardize column names**  
    Your notes had variations like:
    
    - `Finish` vs `Finishes`
    - `Size g` vs `Size grams`
    - `$ 29`, `$29`, `29`
    - `0.04oz` vs `0.04 oz`
    
    Clean them into consistent columns like:
era
product_name
sku
category
product_type
formula
price_usd
size_oz
size_g
size_ml
finish
shade_count
packaging_description
description

- **Parse each shade into its own row**  
    Each bullet under “Shades” should become a row with fields like:
    
    ```
    eraproduct_nameshade_numbershade_nameshade_descriptionfinishrgbhslhexis_oos
    ```
    
    Example: `Hot Take` should have `is_oos = TRUE` because your note marks it `OOS`; most others would be `FALSE`.
    
- **Normalize shade colors**  
    The revival data already has RGB, HSL, and hex for many shades, while the 2013 data often uses CMYK or only shade descriptions. So you needed to:
    - keep available RGB/HSL/hex as-is
    - convert CMYK to RGB/hex where possible
    - leave missing colors as null instead of guessing
    - create a derived `color_family` column manually or with rules
- **Create a `color_family` field**  
    This was important for the charts you wanted: pink, red, berry/plum, brown, nude, coral/orange, blue, green, white/silver, gold/bronze, purple, black/gray, etc.
    
    This is the field that powers tables like:
    
    ```
    era | category | color_family | shade_rows
    ```
    
- **Create packaging feature flags**  
    Because your thesis depends on original uniformity vs revival playfulness, you needed to turn packaging descriptions into structured columns:
    
    ```
    packaging_colorpackaging_shapehas_chromehas_charmhas_mirroris_colorfulis_novelty_shapehas_logo_embossingpackaging_notes
    ```
    
    For example, revival products include star, heart, daisy, lilac, yellow, chrome charm language, while original products are more often black/glossy/pill-shaped in your analysis.
    
- **Fix obvious data-entry inconsistencies**  
    The big one: **Flashes Mascara** has `Formula: Single Pressed Eyeshadow`, which is almost certainly wrong and should be `Mascara`.
- **Decide what to exclude from certain analyses**  
    For your core beauty comparison, you probably should separate or flag:
    
    - nail polish
    - brow gel
    - lip balm
    - tools/brush references
    - products with incomplete shade/color data

4. download the csv and upload into snowflake as raw data
5. created databse schema in snowflake
   ```
CREATE DATABASE MARC_JACOBS_BEAUTY;
CREATE SCHEMA RAW;
CREATE SCHEMA CLEAN;
CREATE SCHEMA ANALYTICS;
   ```
loaded my csvs from row zero into raw

Once you were in **Snowflake**, the workflow was:

### 1. Create a database/schema for the project

Something like:

```sql
CREATE DATABASE MARC_JACOBS_BEAUTY;
CREATE SCHEMA RAW;
CREATE SCHEMA CLEAN;
CREATE SCHEMA ANALYTICS;
```

The idea was:

```text
RAW = messy imported notes / CSVs
CLEAN = cleaned master tables
ANALYTICS = summary tables for charts
```

### 2. Load your cleaned CSVs into `RAW`

You pulled in the Row Zero-exported or manually cleaned tables, probably something like:

```text
RAW.PRODUCTS_RAW
RAW.SHADES_RAW
RAW.PACKAGING_RAW
```

At minimum, the two most important ones were:

```text
PRODUCTS_RAW
SHADES_RAW
```

### 3. Create a cleaned master product table

You then wrote SQL to standardize columns and fix messy values:

```sql
CREATE OR REPLACE TABLE CLEAN.PRODUCTS_MASTER AS
SELECT
    LOWER(TRIM(era)) AS era,
    TRIM(product_name) AS product_name,
    TRIM(category) AS category,
    TRIM(product_type) AS product_type,
    TRIM(formula) AS formula,
    TRY_TO_NUMBER(REPLACE(price_usd, '$', '')) AS price_usd,
    TRY_TO_NUMBER(size_oz) AS size_oz,
    TRY_TO_NUMBER(size_g) AS size_g,
    TRY_TO_NUMBER(size_ml) AS size_ml,
    TRIM(finish) AS finish,
    TRIM(packaging_description) AS packaging_description
FROM RAW.PRODUCTS_RAW;
```

### 4. Create a cleaned master shade table

This was the big one, because the shade data powers most of your story.

```sql
CREATE OR REPLACE TABLE CLEAN.SHADES_MASTER AS
SELECT
    LOWER(TRIM(era)) AS era,
    TRIM(product_name) AS product_name,
    TRIM(category) AS category,
    TRIM(product_type) AS product_type,
    TRIM(shade_number) AS shade_number,
    TRIM(shade_name) AS shade_name,
    TRIM(shade_description) AS shade_description,
    LOWER(TRIM(finish)) AS finish,
    LOWER(TRIM(color_family)) AS color_family,
    TRIM(hex) AS hex,
    TRY_TO_NUMBER(rgb_r) AS rgb_r,
    TRY_TO_NUMBER(rgb_g) AS rgb_g,
    TRY_TO_NUMBER(rgb_b) AS rgb_b,
    IFF(LOWER(oos) IN ('true', 'yes', 'oos'), TRUE, FALSE) AS is_oos
FROM RAW.SHADES_RAW;
```

### 5. Add derived fields for your thesis

This is where you made the data useful, not just clean.

You created fields like:

```text
is_complexion
is_color_cosmetic
is_high_chroma
is_playful_packaging
is_novelty_shape
has_chrome
has_charm
is_colorful_packaging
```

For example:

```sql
CASE 
  WHEN product_type ILIKE '%foundation%'
    OR product_type ILIKE '%concealer%'
    OR product_type ILIKE '%bronzer%'
  THEN TRUE
  ELSE FALSE
END AS is_complexion
```

This helped you show:

```text
Original launch = more complexion + broader classic beauty assortment
Revival = fewer products, more color cosmetics, more playful packaging
```

### 6. Create analytics summary tables

Then you made small tables/views for charts.

For color family by era/category:

```sql
CREATE OR REPLACE TABLE ANALYTICS.SHADE_COLOR_DISTRIBUTION AS
SELECT
    era,
    category,
    color_family,
    COUNT(*) AS shade_rows
FROM CLEAN.SHADES_MASTER
GROUP BY era, category, color_family
ORDER BY era, category, shade_rows DESC;
```

For product counts:

```sql
CREATE OR REPLACE TABLE ANALYTICS.PRODUCT_COUNT_BY_ERA_CATEGORY AS
SELECT
    era,
    category,
    COUNT(DISTINCT product_name) AS product_count
FROM CLEAN.PRODUCTS_MASTER
GROUP BY era, category
ORDER BY era, category;
```

For finish distribution:

```sql
CREATE OR REPLACE TABLE ANALYTICS.FINISH_DISTRIBUTION AS
SELECT
    era,
    category,
    finish,
    COUNT(*) AS shade_count
FROM CLEAN.SHADES_MASTER
GROUP BY era, category, finish
ORDER BY era, category, shade_count DESC;
```

For packaging comparison:

```sql
CREATE OR REPLACE TABLE ANALYTICS.PACKAGING_FEATURES_BY_ERA AS
SELECT
    era,
    COUNT(*) AS product_count,
    SUM(IFF(is_colorful_packaging, 1, 0)) AS colorful_packaging_count,
    SUM(IFF(is_novelty_shape, 1, 0)) AS novelty_shape_count,
    SUM(IFF(has_chrome, 1, 0)) AS chrome_count,
    SUM(IFF(has_charm, 1, 0)) AS charm_count
FROM CLEAN.PRODUCTS_MASTER
GROUP BY era;
```

### 7. Query the analytics tables for Row Zero / charts

This is the table you were asking about earlier:

```text
ANALYTICS.SHADE_COLOR_DISTRIBUTION
```

And then you wanted to split it into readable tables, like:

```sql
SELECT
    category,
    color_family,
    shade_rows
FROM ANALYTICS.SHADE_COLOR_DISTRIBUTION
WHERE era = 'original_launch'
ORDER BY category, shade_rows DESC;
```

Or for revival:

```sql
SELECT
    category,
    color_family,
    shade_rows
FROM ANALYTICS.SHADE_COLOR_DISTRIBUTION
WHERE era = 'revival'
ORDER BY category, shade_rows DESC;
```

### The mental model

Once you were in Snowflake, you were doing this:

```text
Raw imported tables
↓
Clean master tables
↓
Derived thesis fields
↓
Analytics summary tables
↓
Charts in Row Zero / Python / Tableau
```

Your goal was not to make Snowflake the final visual layer. Snowflake was the **data-cleaning and summary engine**. Row Zero, Python, or Tableau could then become the storytelling layer.



---

plugging it back into row zero

You connected **Row Zero to Snowflake** by treating Snowflake as the warehouse and Row Zero as the spreadsheet/BI front end.

The flow was basically:

```text
Snowflake account
↓
Database / schema / analytics tables
↓
Row Zero connection
↓
Query Snowflake tables directly in Row Zero
↓
Build pivot tables / charts / exports
```

## What you did

You created the cleaned and analytics tables in Snowflake first, then connected Row Zero to the Snowflake database instead of manually pasting everything into Row Zero.

You would have needed Snowflake connection details like:

```text
Account identifier
Username
Password / authentication method
Warehouse
Database
Schema
Role
```

Then in Row Zero, you went to the database connection/import flow and selected **Snowflake** as the source. After connecting, you could pull in tables like:

```text
ANALYTICS.SHADE_COLOR_DISTRIBUTION
ANALYTICS.PRODUCT_COUNT_BY_ERA_CATEGORY
ANALYTICS.FINISH_DISTRIBUTION
ANALYTICS.PACKAGING_FEATURES_BY_ERA
```

The key idea was: **don’t make Row Zero hold the messy raw data. Let Snowflake clean and summarize it, then let Row Zero consume the clean summary tables.**

## What you struggled with

The main struggle was probably **figuring out where each tool starts and stops**.

At first, it felt like:

```text
Do I clean in Row Zero?
Do I clean in Snowflake?
Do I visualize in Row Zero?
Do I use Tableau?
Do I use Python?
```

The better division became:

```text
Snowflake = cleaning, joins, calculated fields, summary tables
Row Zero = spreadsheet-style inspection, pivoting, lightweight charting, sharing
Python = polished custom charts
Tableau = optional BI storytelling, but not necessary
```

## The specific Row Zero/Snowflake friction

You also seemed to struggle with **what exactly to pull into Row Zero**.

The answer was: not everything.

You did **not** need to pull in giant raw notes or every messy intermediate table. You needed to pull in the clean analytics-ready outputs:

```text
One table for product counts
One table for shade color distribution
One table for finish distribution
One table for packaging features
One table for price comparison
```

## The other thing you struggled with: visual polish

You also ran into the limitation that Row Zero/Snowflake are good for data work, but not ideal for highly custom visual design.

I really wanted pie charts and more features for plot design and format

That’s why we talked about using:

```text
Snowflake → data prep
Row Zero → exploration / simple tables / pivots
Python locally → final designed plots
```

You were especially running into issues around **fonts and stylistic control**. Snowflake is not where you want to fight with custom fonts, chart aesthetics, or publication-grade visual styling. For that, your own Python environment on your Mac is better.

here's what the connected tables look like in row zero: https://rowzero.com/workbook/51FC4EA39934601C329577DA

made some pivot tables from the data i pulled in from snowflake

----

made the plots in snowflake ipynb and imported fonts and made horizontal and vertical versions for youtube and write ups and mobiel content