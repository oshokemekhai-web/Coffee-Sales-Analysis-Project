# Coffee Shop Sales Analysis: Data Cleaning & Transformation Log

## Table of Content

- [Project Overview](#project-overview)
- [Cleaning and Preparation](#cleaning-and-preparation)
- [Next Steps](#next-steps)
- [Executive Summary, Key Findings and Recommendations](#executive-summary,-key-findings-and-recommendations)

## Project Overview
This project involves cleaning and transforming a highly inconsistent, multi-regional dataset containing over 10,000 coffee shop transactions, customer details, and product inventory.

The project aims to provide insights into the sales perfomance of a coffee shop over the pass 6 years, we seek to identify trends, make data driven recommendations, and gain a deeper understanding of the company's performance.

## Cleaning and Preparation
**Tool used:** PowerBi [Download here](https://www.microsoft.com/en-us/download/details.aspx?id=58494)

### Phase 1: Dimension Table - Customers
* **Duplicate Resolution:** Identified duplicate `Customer_ID` entries caused by system generation errors. Sorted by data completeness and `Loyalty Card` status to prioritize high-value profiles, then applied **Remove Duplicates** to ensure data model integrity.
* **Text Standardization:** Applied `Trim` and `Capitalize Each Word` to the `Customer Name` column to resolve extreme spacing and inconsistent casing errors.
* **Categorical Cleaning:** Standardized the `Gender` column (mapping 'M' to 'Male', 'F' to 'Female') and `Loyalty Card` column (mapping 'Y' to 'Yes', 'N' to 'No') using the Replace Values function.

### Phase 2: Dimension Table - Products
* **Typographical Correction:** Standardized `Coffee_Type` errors (e.g., changing "Robica" to "Robusta", "Ara" to "Arabica") and `Roast_Level` abbreviations (L, M, D to Light, Medium, Dark).
* **Unit of Measure Conversion:** The `Package_Size` column contained mixed string and numeric formats (1000g, 0.5, 1kg, 250g). Converted all values to a standardized Kilogram (kg) decimal format and changed the data type to Decimal Number.
* **Financial Integrity Check:** Filtered out erroneous `Unit_Price` entries containing negative values or exactly $0.00. 
* **Primary Key Enforcement:** Removed duplicates from `Product_ID` to prepare for a 1-to-Many relationship model.

### Phase 3: Fact Table - Orders
* **Outlier Removal:** Filtered the `Quantity` column to remove impossible transaction values (-5, 0, 999, 5000) and constrained the data to realistic retail bounds (1 to 50 units).
* **Discount Standardization:** Transformed mixed text and percentage values in `Discount_Applied` ('None', '10%', '0.2') into uniform decimal formats (0, 0.1, 0.2) for DAX calculation compatibility.
* **Complex Date Parsing:** The `Order Date` column contained a critical mix of US (MM/DD/YYYY), UK (DD-MM-YYYY), and long-text date formats. Utilized the Power Query **Change Type -> Using Locale (English - United States)** engine to universally translate and standardize all rows into a single Date data type.

## Next Steps
1. Load cleaned tables into Power BI Data Model.
2. Establish 1-to-Many relationships between Dimension and Fact tables.
3. Formulate DAX measures for Total Revenue, Gross Profit, and Customer Acquisition.

### Phase 4: Data Modeling & DAX Calculations

### Core Metrics (Scorecard KPIs)
To provide an accurate financial and overview, several dynamic measures were created using DAX:

**Volume & Foot Traffic Metrics:**
* **Total Orders:** Counted the total number of transactions processed.
  ```dax
  Total Orders = DISTINCTCOUNT('Order'[Transaction ID])
  ```
* **Total Bag Sold:** Aggregated the total physical volume of coffee moved.
  ```dax
  Total Bag Sold = SUM('Order'[Quantity])
  ```
* **Unique Customers:** Calculated the total number of distinct individuals who made a purchase.
  ```dax
  Unique Customers = DISTINCTCOUNT('Order'[Customer_ID])
  ```
**Financial Metrics:**

* **Total Gross Revenue:** 
  Calculated the pre-discount total by multiplying the ordered quantity by the unit price from the dimension table.
  ```dax
  Total Gross Revenue = SUMX('Order', 'Order'[Quantity] * RELATED('products'[Unit_Price]))
  ```
* **Total Net Revenue:**
 Factored in the applied promotional discounts to find the true realized revenue.
  ```dax
  Total Net Revenue = SUMX('Order', ('Order'[Quantity] * RELATED('products'[Unit_Price])) * (1 - 'Order'[Discount_Applied]))
  ```
* **Total Discount Value:**
 Quantified the total financial impact of promotional campaigns.
  ```dax
  Total Discount Value = [Total Gross Revenue] - [Total Net Revenue]
  ```
* **Average Order Value (AOV):**
 Tracked the average customer spend per transaction, utilizing the `DIVIDE` function to elegantly handle potential zero-division errors.
  ```dax
  Average Order Value = DIVIDE([Total Net Revenue], [Total Orders], 0)
  ```
* **Profitability Metrics (With Benchmark Assumption):**
*Note: Because the raw product inventory export did not include wholesale unit costs, a standard specialty-coffee wholesale benchmark of 60% of retail `Unit_Price` was applied to model Cost of Goods Sold (COGS).*

* **Total COGS (Cost of Goods Sold):** Modeled the baseline inventory cost at 60% of the retail unit price, scaled row-by-row by transaction quantity.
  ```dax
  Total COGS = SUMX('Order', 'Order'[Quantity] * (RELATED('products'[Unit_Price]) * 0.60))
* **Total Gross Profit:** Calculated the true operational profitability by subtracting the modeled cost of goods sold from net realized revenue (after promotional discounts).
  ```dax
  Total Gross Profit = [Total Net Revenue] - [Total COGS]
  ```
* **Gross Profit Margin:** Determined the percentage of net revenue retained as gross profit.
  ```dax
  Gross Profit Margin = DIVIDE([Total Gross Profit], [Total Net Revenue], 0)
  ```

#### **Category Metrics**

**Product Dimensions:**

\- Coffee Type (Arabica, Robusta, etc.)  
\- Roast Level (Light, Medium, Dark)  
\- Package Size (0.25kg, 0.5kg, 1kg, etc.)

**Customer Demographics:**

\- Loyalty Card Status (Yes / No)  
\- Gender

**Geographic Dimensions:**

\- City  
\- Country

**Temporal (Time) Dimensions:**

\- Clean Order Date (Used for Year, Quarter, Month, and Day trends)


### Advanced DAX: Dynamic Axis (Country → City and Year → Month)
 To let a slicer control how detailed a chart is, without using drill-down arrows or Field Parameters, I built two disconnected axis tables: `GeoAxis` (Country → City) and `DateAxis` (Year → Month). Each one stacks the labels of every level into a single `Label` column, with a `Level` column (which level the label belongs to) and a `Parent` or `Yr` column (which country or year it belongs to). `DateAxis` also has a `Sort` column so months appear in calendar order.

 The charts use `Label` as their axis. Two measures, `Dynamic Revenue` and `Dynamic Revenue by Date`, use `SWITCH(TRUE())` with `SELECTEDVALUE` and `HASONEVALUE` to read what the slicer currently selects. They return `Total Net Revenue` (through `CALCULATE` and `KEEPFILTERS`) only for labels that belong to the right level, and `BLANK()` for all others, so Power BI hides them.

 With no selection, the charts show countries and years. With one country or year selected, they show that country's cities or that year's months. Clearing the slicer returns the chart to the top level, so it never shows every city or every month at once.


## Executive Summary, Key Findings and Recommendations
<img width="1446" height="811" alt="Coffee Sales Dashboard" src="https://github.com/user-attachments/assets/c860f090-7de6-4ed3-b5da-8bf636583277" />

### 1. Executive Summary
Across the analyzed period (2020–2026), the business processed **6,367 orders** from **2,256 unique customers**, moving **32,081 bags of coffee**. This generated **$681,216.93 in Gross Revenue** and **$578,521.41 in Net Revenue** after promotional discounts (an Average Order Value of **$90.86**). Accounting for an estimated **$408,730.16 in Cost of Goods Sold (COGS)**, the business retained **$169,791.25 in Gross Profit**, achieving a healthy **29.35% Gross Profit Margin**.

---

### 2. Deep-Dive Business Insights

* **Promotional Discount Leakage (Total Gross Revenue - Total Net Revenue = 102.7k):** 
  Comparing Gross Revenue ($681,216.93) against Net Revenue ($578,521.41) reveals that **$102,695.52 (15.08% of gross sales)** was given away in promotional discounts. While discounts drive volume (averaging **5.04 bags per order**), heavy 20% discounting directly compresses the gross margin.
* **Product & Bean Dominance:** 
  **Excelsia** is the flagship revenue generator at **$181,222.50 (31.3% of Net Revenue)**, followed closely by **Arabica ($163,115.89 / 28.2%)** and **Liberica ($148,387.35 / 25.6%)**. **Robusta** trails significantly, generating only **$85,795.67 (14.8%)**.
* **Roast & Packaging Preferences:** 
  Customer palate strongly favors **Light Roast**, which generated **$266,258.22 (46.0% of total revenue)**—nearly as much as **Dark ($160,545.67)** and **Medium ($151,717.52)** roasts combined. For packaging, the **0.50kg bag ($0.21M)** and **2.00kg bulk bag ($0.17M)** are the highest revenue contributors, while the small **0.25kg bag ($0.07M)** underperforms.
* **Geographic Stability:** 
  Revenue is remarkably well-diversified across all eight operating markets. The **United States ($79.29K)**, **France ($76.45K)**, and **Canada ($73.96K)** lead global sales, while **Nigeria ($71.48K)** and **Germany ($72.22K)** anchor strong mid-tier performance.
* **Loyalty Program & Data Capture Gap:** 
  Repeat purchasing is strong (averaging **2.82 orders per unique customer**), yet Non-Loyalty orders (**$239,967.40**) slightly edge out confirmed Loyalty Card members (**$235,934.68**). Crucially, **$102,619.33** in revenue comes from transactions where loyalty status was **"Not Specified"**, indicating a checkout data-capture issue.
* **Revenue Trajectory (2020–2026):** 
  Sales peaked in **2024** (surpassing $90,000) after recovering from a 2022–2023 plateau, before experiencing a downturn through 2025 and into 2026.

---

### 3. Strategic Recommendations

1. **Restructure Discount Tiers to Protect Margin:** 
   With $102.7K lost to discounts, management should restrict 20% discounts exclusively to **2.00kg bulk orders** or **Loyalty Card members**. Shifting standard promotions to a 10% cap will immediately boost the 29.35% Gross Profit Margin.
2. **Optimize Inventory Around Light Roast & Excelsia/Arabica:** 
   Allocate procurement and marketing budgets toward **Light Roast Excelsia and Arabica in 0.50kg and 2.00kg sizes**, which represent the highest velocity products. Reduce **Robusta** and **0.25kg package** inventory to prevent dead stock.
3. **Fix Checkout Data Capture & Convert Non-Members:** 
   Mandate the `Loyalty Card` field at checkout to eliminate the **$102.6K "Not Specified"** blind spot, and launch a targeted checkout prompt to convert the **$240K non-member segment** into recurring loyalty members.
4. **Launch Re-engagement Campaigns for 2025/2026 Attrition:** 
   Target existing customers in top markets (US, France, Canada, Germany, and Nigeria) who ordered during the 2024 peak but have not purchased in 2026 with bundle offers on 0.50kg Light Roast bags.
