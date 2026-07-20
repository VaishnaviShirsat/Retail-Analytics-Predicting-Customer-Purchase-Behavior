# 🛍️ Retail Analytics: Predicting Customer Purchase Behavior

A capstone project for **RetailVC**, exploring customer demographics, purchasing patterns, and marketing campaign responses to guide data-driven promotional strategy. The workflow spans **Excel, SQL, Python (EDA), and Tableau**, using the classic `marketing_campaign` retail dataset (2,240 customers).

---

## 📌 Project Description

RetailVC wanted to understand what drives customers to respond to marketing campaigns and to build a foundation for future response-prediction models. This project analyzes customer demographics (age, education, marital status, income), purchase behavior across product categories (wine, meat, fish, fruit, sweets, gold products), and historical campaign acceptance to surface insights that can improve targeting and ROI on future promotions.

The analysis was carried out in four stages, each using a different tool suited to the task:

| Stage | Tool | Purpose |
|---|---|---|
| 1 | **Excel** | Quick descriptive stats, pivot tables, and first-look charts |
| 2 | **SQL (MySQL Workbench)** | Data loading, schema design, aggregation queries |
| 3 | **Python (pandas, seaborn, matplotlib)** | Full exploratory data analysis (EDA) |
| 4 | **Tableau** | Interactive dashboard for stakeholder consumption |

---

## 🗂️ Dataset

- **Source table:** `marketing_campaign`
- **Rows:** 2,240 customers | **Columns:** ~29 features
- **Key fields:**
  - Demographics: `Year_Birth`, `Education`, `Marital_Status`, `Income`, `Kidhome`, `Teenhome`
  - Engagement: `Dt_Customer` (enrollment date), `Recency`, `NumWebVisitsMonth`
  - Spend by category: `MntWines`, `MntFruits`, `MntMeatProducts`, `MntFishProducts`, `MntSweetProducts`, `MntGoldProds`
  - Purchase channels: `NumDealsPurchases`, `NumWebPurchases`, `NumCatalogPurchases`, `NumStorePurchases`
  - Campaign history: `AcceptedCmp1`–`AcceptedCmp5`, `Response` (target variable), `Complain`

---

## 1️⃣ Excel: Data Exploration

- **Descriptive statistics** generated per numerical feature via `Data → Data Analysis → Descriptive Statistics`.
- **Enrollments by year** (pivot table + line chart): 2012 → 494, **2013 → 1,189 (peak)**, 2014 → 557.
- **Response × Education cross-tab:** Graduation is both the largest segment (1,127 customers) and the largest source of "Yes" responses (152); PhDs show a comparatively higher response rate.
- **Income boxplot:** heavy right-skew — median ≈ **$51,000**, 25th pct ≈ $35,625, 75th pct ≈ $68,592, with high-income outliers pulling the mean up (extreme values near $670K).
- **Age histogram:** most customers fall between **47–56 years old**; very few under 28 or over 82.
- **Response × Marital Status:** Married customers are the largest responder group (98 "Yes"), but **Singles show a higher response *rate*** relative to their smaller segment size (106 "Yes" out of 480).

**Overall response split:** 1,906 "No" (85%) vs. 334 "Yes" (15%) — a fairly imbalanced target variable, relevant for any future classification model.

---

## 2️⃣ SQL: Data Loading & Preprocessing (MySQL)

Schema `retail_data` was created, the CSV loaded via MySQL Workbench's import wizard, and `Dt_Customer` cast to a proper `DATE` type (after reformatting to `yyyy-mm-dd` in Excel first).

**Highlights from the queries** (full script in [`sql/analysis.sql`](sql/analysis.sql)):

- Total customer encounters: **2,216**
- Top purchased category by revenue: **Wines** (highest `SUM(MntWines)`), followed by Meat Products and Gold Products
- Response counts: **0 → 1,883**, **1 → 333**
- Average income of campaign participants: **≈ ₹52,247**
- Average household composition: **0.44 children**, **0.50 teenagers**
- `Age` derived from `YEAR(NOW()) - Year_Birth`, then bucketed into `Age_group` (18-25, 26-35, 36-45, 46-55, 56+)
- Average web visits/month by age group — the **56+ group visits most frequently** (5.09/mo), while 26-35 visits least (4.57/mo)

```sql
-- Example: total promotions accepted per customer
SELECT
    id,
    SUM(AcceptedCmp3 + AcceptedCmp4 + AcceptedCmp5 + AcceptedCmp1 + AcceptedCmp2) AS Total_Promotions_accepted
FROM marketing_campaign
GROUP BY id;

-- Example: age-group segmentation
UPDATE marketing_campaign
SET Age_group_1 = CASE
    WHEN Age BETWEEN 18 AND 25 THEN '18-25'
    WHEN Age BETWEEN 26 AND 35 THEN '26-35'
    WHEN Age BETWEEN 36 AND 45 THEN '36-45'
    WHEN Age BETWEEN 46 AND 55 THEN '46-55'
    ELSE '56+'
END;
```

---

## 3️⃣ Python: Exploratory Data Analysis

Performed in a `pandas` / `seaborn` / `matplotlib` pipeline (full script in [`python/eda.py`](python/eda.py)):

**Data cleaning**
- 24 missing values found in `Income`, imputed with the **median (≈ 51,381.5)**
- No other missing values after imputation

**Univariate analysis**
- Spend variables (`MntWines`, `MntMeatProducts`, etc.) are strongly **right-skewed** — most customers spend little, a small segment spends heavily
- `Income` and `Age` are roughly bell-shaped once outliers are handled
- Categorical distributions: **Graduation** dominates `Education`; **Married** dominates `Marital_Status`

**Bivariate analysis**
- **Correlation matrix**: `MntWines`, `MntMeatProducts`, and `NumCatalogPurchases` correlate positively with `Response`; `NumWebVisitsMonth` correlates negatively (customers who visit the site often without buying are less likely to respond)
- **MntWines vs. Response boxplot**: responders spend ~3x more on wine (median ≈ $450 vs ≈ $150) — the single strongest behavioral signal in the dataset
- **Response by Education / Marital Status bar charts**: response propensity is fairly proportional across groups, meaning demographic-only targeting is weak — behavioral signals (spend, recency, channel usage) matter more

**Fraud-detection framing (deliverable requirement)**
While this dataset is a marketing-response dataset rather than a labeled fraud dataset, the EDA workflow doubles as a template for a retail fraud-detection pipeline: outlier detection on `Income` and category spend (box plots), correlation analysis to catch anomalous co-purchase patterns, and channel-mix analysis (web/catalog/store) to flag accounts with abnormal purchase-channel behavior. The same missing-value handling, distribution checks, and bivariate analysis against a binary target (`Response` → analogous to `is_fraud`) generalize directly to a fraud use case.

```python
# Core EDA snippet
df['Income'] = df['Income'].fillna(df['Income'].median())

num_cols = df.select_dtypes(include=np.number).columns
for col in num_cols:
    sns.histplot(df[col], kde=True)
    plt.title(col)
    plt.show()

plt.figure(figsize=(16, 16))
sns.heatmap(df.select_dtypes([np.number]).corr(), annot=True, cmap='coolwarm')
plt.title('Correlation Matrix')
plt.show()
```

---

## 4️⃣ Tableau: Dashboard

An interactive **4-view dashboard** combining all visual analysis (see [`tableau/`](tableau/) for the workbook):

1. **Yearly Income Distribution** — income binned in $5,000 buckets by year of registration; most customers cluster around $35K, very few earn six figures.
2. **Education & Marital Status breakdown** — side-by-side circle plot; **Married + Graduation** is the largest combined segment (~429 customers).
3. **Income vs. Wine Spending** — scatter plot showing wine spend is **not strictly income-driven**; the heaviest normalized spenders sit in the middle-income band, suggesting lifestyle/preference effects dominate over pure purchasing power.
4. **Multiple Transactions** — spend-by-category bar strip per customer ID; **Wine > Gold Products > Meat/Fish/Sweets > Fruits**, with most customers spending near-zero and a long tail of high spenders.

All four views are combined into a single **Tableau Dashboard** using horizontal/vertical containers for stakeholder review.

---

## 🔑 Key Business Insights

1. **Wine spend is the strongest behavioral predictor of campaign response** — far more informative than any single demographic field.
2. **The response base is heavily imbalanced (85/15)** — any predictive model should account for this (e.g., class weighting, resampling) rather than optimizing for raw accuracy.
3. **Demographics alone are weak targeting criteria** — education and marital status shift response rates only modestly; combining them with spend/recency/channel behavior is far more actionable.
4. **Middle-income, mid-40s–mid-50s customers are the core segment** — both in volume and in purchasing activity — and should anchor future campaign targeting.
5. **Web visit frequency without purchase is a negative signal** — high `NumWebVisitsMonth` correlates with lower response, useful for filtering low-intent traffic out of campaign lists.

---

## 🧰 Tech Stack

`Excel` · `MySQL / MySQL Workbench` · `Python 3.13` (`pandas`, `numpy`, `seaborn`, `matplotlib`, `scipy`) · `Tableau Public/Desktop`





## 📄 License

This project is for educational/capstone purposes. Dataset is the publicly available "Marketing Campaign" retail dataset.
