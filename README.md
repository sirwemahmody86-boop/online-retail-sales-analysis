# 📊 Online Retail Sales & Customer Analytics

A complete end-to-end data analysis project — from raw, messy transactional data to a fully interactive Power BI dashboard and a business-ready analytical report.

**[View Dashboard Screenshots](#dashboard-preview)** · **[Read the Full Business Report](./Online_Retail_Business_Report.md)** · **[Read the Technical Cleaning/Modeling Log](./Online_Retail_Cleaning_Roadmap.md)**

---

## 🎯 Project Overview

This project analyzes one year (Dec 2010 – Dec 2011) of transactional data from a UK-based online retailer, containing **541,909 raw records**. The goal was to answer a core business question:

> **Where does revenue come from, who drives it, and where is the business exposed to risk?**

The project covers the full analytics lifecycle: data cleaning, dimensional modeling, DAX measure development, interactive dashboard design, and business reporting.

## 🧰 Tools & Skills Used

| Category | Tools |
|---|---|
| Data Cleaning | Power Query (M language) |
| Data Modeling | Power BI (star-adjacent model, Calendar Table, DAX Time Intelligence) |
| Analysis | DAX Measures, Python (pandas) for RFM segmentation |
| Visualization | Power BI (custom theme, custom navigation, KPI cards) |
| Validation | Cross-checked every key metric against independent Python calculations |

## 🔍 Key Findings

- **Geographic concentration risk:** 85% of total revenue (£10.2M) comes from a single country (UK) — a significant dependency risk.
- **Customer concentration (Pareto pattern):** Just **18% of customers ("Champions")** generate **60% of total revenue**.
- **Return rate:** 7.76% of sales value (~£894K) is reversed via cancellations, concentrated in a small number of specific products rather than spread evenly.
- **Top product:** *Regency Cakestand 3 Tier* alone generated £174K in revenue.

Full findings, methodology, and business recommendations are in the [Business Report](./Online_Retail_Business_Report.md).

## 🧹 Data Cleaning Highlights

Rather than blindly dropping incomplete rows, every data quality issue was investigated, quantified, and handled with a documented, defensible decision:

- 5,268 exact-duplicate rows removed
- 25% of rows had missing Customer IDs → flagged (not deleted) to preserve accuracy of revenue-level metrics
- Discovered and corrected inconsistent product codes caused by case-sensitivity (e.g. `15056BL` vs `15056bl`) affecting 22,833 rows
- Separated genuine sales from cancellations, accounting adjustments, and non-product line items (postage, fees, manual entries)

Full step-by-step log with validation numbers: [`Online_Retail_Cleaning_Roadmap.md`](./Online_Retail_Cleaning_Roadmap.md)

## 📐 Data Model

A dedicated `CalendarTable` was built with `CALENDAR()` (not `CALENDARAUTO()`, which produced an incorrect date range — see the roadmap for why) and marked as the official Date Table, related 1:* to the transactions table to enable proper Time Intelligence (month-over-month comparisons, trend analysis).

## 📊 Dashboard Preview

The dashboard consists of a cover page and 4 interactive pages, connected via custom navigation buttons (not default page tabs):

| Page | Purpose |
|---|---|
| **Overview** | Executive KPIs, monthly revenue trend, revenue by country |
| **Products** | Top-selling products, revenue breakdown |
| **Customers** | Revenue per customer, purchase frequency segmentation, top customers |
| **Returns** | Cancellation trend over time, most-returned products |

*(Add your exported screenshots or a GIF walkthrough here)*

## 🧪 How the Numbers Were Validated

Every DAX measure in this project was cross-checked against an independent Python/pandas calculation on the same cleaned dataset before being trusted — a practice used throughout to catch subtle bugs (e.g., a mismatched granularity issue in a KPI's month-over-month comparison that silently produced an incorrect target value).

## 📁 Repository Structure

```
├── Online_Retail_Business_Report.md      # Executive summary, findings, recommendations
├── Online_Retail_Cleaning_Roadmap.md     # Full technical log: cleaning, modeling, DAX, design
├── p1.ipynb                              # Python/pandas exploratory analysis + RFM segmentation
├── OnlineRetail_NavyTheme.json           # Custom Power BI report theme
└── [YourDashboard].pbix                  # Power BI dashboard file
```

## 🚀 Next Steps (Roadmap)

- [ ] Migrate to a full star schema (separate Customer/Product/Date dimension tables)
- [ ] Bring RFM segmentation into the Power BI model itself (currently in Python)
- [ ] Add drill-through pages for deeper product/customer investigation

---

**Data source:** [UCI Machine Learning Repository — Online Retail Dataset](https://archive.ics.uci.edu/dataset/352/online+retail) (CC BY 4.0)
