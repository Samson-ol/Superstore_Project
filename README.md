# Superstore Sales & Profitability Dashboard (Power BI)

An end-to-end data analytics project using the Sample Superstore dataset, covering data cleaning, data modeling, DAX measures, and an interactive 4-page Power BI dashboard that surfaces where the business makes money, where it loses money, and why.

---

## 1. Project Overview

**Objective:** Analyze four years (2014–2017) of US retail order data to identify sales and profit trends, uncover the drivers of unprofitable orders, and present the findings in an interactive, navigable Power BI dashboard.

**Tools used:** Power BI Desktop (Power Query, Data Modeling, DAX), Figma (dashboard background/visual design), Python/Pandas (initial exploratory analysis).

**Dataset:** [Sample – Superstore](https://community.tableau.com/s/question/0D54T00000CWeX8SAL/sample-superstore-sales-excelxls) — a widely used public retail dataset (9,994 rows, 21 columns) covering orders, customers, products, and shipping across the United States.

---

## 2. Dataset Summary

| Field | Detail |
|---|---|
| Rows | 9,994 order lines |
| Columns | 21 |
| Date range | January 2014 – December 2017 |
| Orders | 5,009 unique orders |
| Customers | 793 unique customers |
| Products | 1,862 unique products |
| Geography | United States only |
| File encoding | Latin-1 (not UTF-8) |

**Columns:** Row ID, Order ID, Order Date, Ship Date, Ship Mode, Customer ID, Customer Name, Segment, Country, City, State, Postal Code, Region, Product ID, Category, Sub-Category, Product Name, Sales, Quantity, Discount, Profit.

**Key definitions established during analysis:**
- `Sales` = revenue actually collected on the order line, calculated as `Quantity × List Price × (1 − Discount)`. It is **not** a unit price and **not** pre-discount.
- `Profit` is provided directly in the dataset. Implied cost can be derived as `Sales − Profit`, and this implied unit cost is consistent per product regardless of quantity or discount, confirming each product has a fixed underlying cost.

---

## 3. Data Cleaning Process

The dataset was structurally clean (no missing values, no duplicate rows), but required the following fixes before modeling:

### 3.1 File encoding
- The CSV is Latin-1 encoded. Set **File Origin → 1252: Western European (Windows)** on import to prevent garbled characters in names.

### 3.2 Data types
- `Order Date` and `Ship Date` were stored as text. Converted both to **Date** type using locale **English (United States)** to avoid day/month misreads.

### 3.3 Postal Code
- 438 rows (in CT, ME, MA, NH, NJ, RI) had lost their leading zero (e.g., `2116` instead of `02116`).
- Fixed by setting the column type to **Text**, then applying a custom column:
  ```
  Text.PadStart(Text.From([Postal Code]), 5, "0")
  ```

### 3.4 Text formatting
- 16 product names had leading/trailing whitespace.
- ~90 customer names and ~280 product names contained non-ASCII artifacts from the encoding.
- Fixed using Power Query's **Trim** and **Clean** transforms across all text columns.

### 3.5 Referential consistency checks
- 32 Product IDs mapped to more than one Product Name, and 16 Product Names mapped to more than one Product ID.
- Resolved by creating a separate **Products** dimension table (duplicate query → kept Product ID, Product Name, Category, Sub-Category → removed duplicates on Product ID), joined back to the fact table via relationship. This ensures accurate distinct product counts.

### 3.6 Redundant columns
- `Country` contained a single value ("United States") across all rows — removed in Power Query.
- `Row ID` was a simple row index with no analytical value — retained in the model but hidden from the report view (Model view → right-click column → *Hide in Report View*).

### 3.7 Derived columns added
| Column | Logic | Purpose |
|---|---|---|
| `Ship Days` | `Duration.Days([Ship Date] - [Order Date])` | Measures fulfillment speed |
| `Cost` | `[Sales] - [Profit]` | Implied product cost |
| `Location` | `[City] & ", " & [State]` | Disambiguates repeated city names across states |
| `Discount Band` | Conditional column: 0 → "No discount"; ≤0.2 → "Up to 20%"; ≤0.4 → "21–40%"; else → "Over 40%" | Groups discount levels for profitability analysis |

### 3.8 Data validation notes (documented, not altered)
- 57 city names appear in more than one state — resolved at the reporting layer by using City + State or State/Postal Code for geographic visuals, not City alone.
- Sales values with more than 2 decimal places (~4,000 rows) were left as-is and simply formatted for display.
- Large outlier sales/loss values were confirmed valid (not data errors) and retained.

---

## 4. Data Modeling

- Built a dedicated **Date table** using DAX (`CALENDAR` + `ADDCOLUMNS`) spanning 2014–01–01 to 2017–12–31, with Year, Month Number, Month (text), and Quarter columns.
- Marked the Date table as the official **date table** in the model.
- Applied **Sort by Column** so the text `Month` field (Jan, Feb, Mar…) displays in chronological rather than alphabetical order, using `Month Number` as the sorting column.
- Related `Date[Date]` to the fact table's `Order Date` (one-to-many).
- Set **Data Category** on `State` (State or Province), `City` (City), and `Postal Code` (Postal Code) to enable accurate geographic mapping.
- Established a one-to-many relationship between the `Products` dimension table and the fact table via `Product ID`.

---

## 5. Key Measures (DAX)

```DAX
Total Sales        = SUM(Orders[Sales])
Total Profit        = SUM(Orders[Profit])
Profit Margin %      = DIVIDE([Total Profit], [Total Sales])
Total Orders        = DISTINCTCOUNT(Orders[Order ID])
Customers          = DISTINCTCOUNT(Orders[Customer ID])
Avg Order Value      = DIVIDE([Total Sales], [Total Orders])
Avg Discount        = AVERAGE(Orders[Discount])
Avg Ship Days        = AVERAGE(Orders[Ship Days])
Loss Line %         = DIVIDE(COUNTROWS(FILTER(Orders, Orders[Profit] < 0)), COUNTROWS(Orders))
Sales LY           = CALCULATE([Total Sales], SAMEPERIODLASTYEAR('Date'[Date]))
Sales YoY %         = DIVIDE([Total Sales] - [Sales LY], [Sales LY])
```

**Validation benchmarks** (checked after modeling to confirm data integrity):
- Total Sales ≈ **$2,297,201**
- Total Profit ≈ **$286,397**
- Total Orders = **5,009**
- Customers = **793**

---

## 6. Dashboard Structure

The report is organized into 4 pages, connected with a built-in **Page Navigator** for seamless browsing:

### Page 0 — Home / Landing Page
- Report title, short project description, and navigation entry points (buttons or image thumbnails) into each analysis page.
- Set as the default page shown on open.

### Page 1 — Overview
- **KPI cards:** Total Sales, Total Profit, Profit Margin %, Total Orders, Customers, Avg Order Value, Sales YoY %
- **Line chart:** Monthly Sales & Profit trend (reveals Q4 seasonality)
- **Clustered column chart:** Sales & Profit by Year
- **Bar chart:** Profit by Category
- **Slicers:** Year, Region, Segment

### Page 2 — Products & Discounts
- **"Profit by Sub-Category"** — sorted bar chart, conditional red formatting for negative values
- **"Profit by Discount Band"** — column chart showing the drop-off in profitability past 20% discount
- **Sales vs Margin scatter plot** — Sub-Category level, Sales on X, Profit Margin % on Y, bubble size by Total Profit
- **Category × Region matrix** — Profit Margin % with conditional background shading

### Page 3 — Geography & Customers
- **"Profit by State"** — Shape Map with color saturation on Total Profit, red–white–green diverging scale (red = loss, green = profit)
- **"Top 10 States by Profit"** — bar chart filtered via Top N
- **"Bottom 10 States by Profit"** — bar chart filtered via Top N on a negated profit measure
- **"Top Customers by Sales"** — table of Customer Name, Total Sales, Total Profit, sorted descending by Sales

---

## 7. Key Insights

### Overall performance
- Total sales: **~$2.30M** | Total profit: **~$286K** | Overall margin: **12.5%**
- Sales grew every year, from $484K (2014) to $733K (2017); margin stayed flat in the 10–13% range despite revenue growth.
- Strong seasonality: sales peak September–December (November alone ≈ $352K vs. February ≈ $60K).
- Average fulfillment time: ~4 days, with no meaningful link between shipping mode and profitability.

### Category & sub-category performance
- **Technology** is the top profit driver ($145K, 17.4% margin); **Office Supplies** follows closely ($122K, 17.0%).
- **Furniture** is the weak link: $742K in sales but only $18K profit (2.5% margin).
- Copiers (37% margin), Phones, and Accessories are the strongest sub-categories; Tables, Bookcases, and Supplies are consistently unprofitable.

### The discount problem (core finding)
- Discounts **above 20%** are the single biggest driver of lost profit.
  - 0% discount → 29.5% margin
  - 10–20% discount → 11.6% margin
  - 20%+ discount → **negative margin**, worsening to roughly **–119%** at 50%+ discount
- ~19% of all order lines (1,871 rows) are loss-making, totaling approximately **–$156K** — more than half of total net profit.
- The single worst example: a 3D printer sold at 70% off, losing $6,600 on one order line.

### Geography
- West and East regions lead in profit; California and New York alone contribute ~$150K.
- **Texas, Ohio, Pennsylvania, Illinois, North Carolina, Colorado, Tennessee, and Arizona are all net unprofitable**, and these states also carry the highest average discount rates (30–40%), directly linking discounting behavior to regional losses.
- Central region has the lowest overall margin (7.9%).

### Customers
- Revenue is not overly concentrated — the top 10 products account for only ~10.6% of total sales.
- The largest customer by revenue (Sean Miller, ~$25K in sales) is actually unprofitable overall (–$2K), illustrating that high sales volume does not guarantee profitability.

---

## 8. Recommendations

1. Cap discounts at or below 20% across the board, with specific review of Tables, Bookcases, and Machines.
2. Investigate discounting practices in Texas, Illinois, and other Central-region states driving regional losses.
3. Reassess Furniture category pricing/cost structure given its persistently thin margin despite strong sales volume.
4. Flag high-revenue, low/negative-profit customers (like Sean Miller) for account-level profitability review rather than judging customer value by sales alone.

---

## 9. Limitations

- Cost is **implied** (Sales − Profit), not provided directly, and may include factors beyond product cost (e.g., shipping/handling) that cannot be isolated from this dataset.
- This is a well-known public/synthetic dataset (commonly used for BI training), not live business data — findings demonstrate analytical method rather than real operational decisions.
- Product ID/Name mismatches (32 IDs) were resolved by keeping the first encountered name per ID; a full data-governance pass would require manual verification against a source system.

---

## 10. Project Files

```
├── Sample_-_Superstore.csv       # Raw source data
├── superstore_dashboard.pbix     # Power BI report file
├── README.md                     # This documentation
└── screenshots/                  # Dashboard page exports (optional)
```

---

## 11. Tools & Skills Demonstrated

- Data cleaning & transformation (Power Query / M)
- Data modeling & relationships (star schema basics)
- DAX measure development (time intelligence, ratios, filtered aggregations)
- Data visualization design (KPI cards, trend lines, geographic maps, matrices)
- Dashboard UX (page navigation, landing page, cross-page slicer sync)
- Exploratory data analysis in Python/Pandas prior to BI tool build
- UI/visual design direction in Figma

---

*Built as a self-directed portfolio project to demonstrate end-to-end analytics workflow: raw data → cleaning → modeling → insight generation → interactive dashboard.*
