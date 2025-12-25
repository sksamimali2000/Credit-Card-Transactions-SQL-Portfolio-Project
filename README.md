# 💳 Credit Card Transactions SQL Portfolio Project

This project demonstrates SQL data exploration and analytics on a real-world credit card transactions dataset from Kaggle. The goal is to import the dataset into SQL Server, clean and optimize it, then solve analytical business problems using SQL queries.

---

## 📂 Dataset

**Source:** [Kaggle – Analyzing Credit Card Spending Habits in India](https://www.kaggle.com/datasets/thedevastator/analyzing-credit-card-spending-habits-in-india)

**Table Name in SQL Server:** `credit_card_transcations`

### Data Preparation Steps

1. Download dataset and unzip if required.
2. Rename all column names:

   * Convert to lowercase
   * Replace spaces with underscores
3. Import into SQL Server with correct data types:

   * `transaction_id` → INT
   * `transaction_date` → DATE/DATETIME
   * `amount` → DECIMAL(18,2)
   * `city`, `exp_type`, `gender`, `card_type` → NVARCHAR

---

## 🔍 SQL Exploration Queries

A few exploratory queries to understand the dataset:

```sql
-- Total records
SELECT COUNT(*) FROM credit_card_transcations;

-- Distinct cities, expense types, card types, genders
SELECT COUNT(DISTINCT city) AS no_of_cities,
       COUNT(DISTINCT exp_type) AS no_of_expense_types,
       COUNT(DISTINCT card_type) AS no_of_card_types,
       COUNT(DISTINCT gender) AS no_of_genders
FROM credit_card_transcations;

-- Top 10 transactions by amount
SELECT TOP 10 * FROM credit_card_transcations ORDER BY amount DESC;

-- Total spend by gender
SELECT gender, SUM(amount) AS total_spend
FROM credit_card_transcations
GROUP BY gender;
```

---

## 📝 Problem Statements & Solutions

### **1. Top 5 cities with highest spends and % contribution**

```sql
WITH cte1 AS (
    SELECT city, SUM(amount) AS total_spend
    FROM credit_card_transcations
    GROUP BY city
),
 total_spent AS (
    SELECT SUM(CAST(amount AS BIGINT)) AS total_amount
    FROM credit_card_transcations
)
SELECT TOP 5 cte1.*, ROUND(total_spend*1.0/total_amount*100,2) AS percentage_contribution
FROM cte1 INNER JOIN total_spent ON 1=1
ORDER BY total_spend DESC;
```

---

### **2. Highest spend month per card type**

```sql
WITH cte AS (
  SELECT card_type,
         DATEPART(YEAR, transaction_date) yt,
         DATEPART(MONTH, transaction_date) mt,
         SUM(amount) AS total_spend
  FROM credit_card_transcations
  GROUP BY card_type, DATEPART(YEAR, transaction_date), DATEPART(MONTH, transaction_date)
)
SELECT *
FROM (
  SELECT *, RANK() OVER(PARTITION BY card_type ORDER BY total_spend DESC) AS rn
  FROM cte
) a
WHERE rn = 1;
```

---

### **3. Transaction details when each card type reaches ₹1,000,000 cumulative spend**

```sql
WITH cte AS (
  SELECT *,
         SUM(amount) OVER(PARTITION BY card_type ORDER BY transaction_date, transaction_id) AS total_spend
  FROM credit_card_transcations
)
SELECT *
FROM (
  SELECT *, RANK() OVER(PARTITION BY card_type ORDER BY total_spend) AS rn
  FROM cte WHERE total_spend >= 1000000
) a
WHERE rn = 1;
```

---

### **4. City with lowest percentage spend for Gold cards**

```sql
WITH cte AS (
  SELECT city,
         SUM(CASE WHEN card_type='Gold' THEN amount END) AS gold_amount,
         SUM(amount) AS total_amount
  FROM credit_card_transcations
  GROUP BY city
)
SELECT TOP 1 city, (gold_amount*1.0/total_amount) AS gold_ratio
FROM cte
WHERE gold_amount IS NOT NULL AND gold_amount > 0
ORDER BY gold_ratio ASC;
```

---

### **5. City-wise highest & lowest expense type**

```sql
WITH cte AS (
  SELECT city, exp_type, SUM(amount) AS total_amount
  FROM credit_card_transcations
  GROUP BY city, exp_type
)
SELECT city,
       MAX(CASE WHEN rn_desc=1 THEN exp_type END) AS highest_expense_type,
       MAX(CASE WHEN rn_asc=1 THEN exp_type END) AS lowest_expense_type
FROM (
  SELECT *,
         RANK() OVER(PARTITION BY city ORDER BY total_amount DESC) rn_desc,
         RANK() OVER(PARTITION BY city ORDER BY total_amount ASC) rn_asc
  FROM cte
) A
GROUP BY city;
```

---

### **6. Female spends % contribution per expense type**

```sql
SELECT exp_type,
       SUM(CASE WHEN gender='F' THEN amount ELSE 0 END)*1.0/SUM(amount) AS percentage_female_contribution
FROM credit_card_transcations
GROUP BY exp_type
ORDER BY percentage_female_contribution DESC;
```

---

### **7. Highest MoM growth (Jan 2014)**

```sql
WITH cte AS (
  SELECT card_type, exp_type,
         DATEPART(YEAR, transaction_date) yt,
         DATEPART(MONTH, transaction_date) mt,
         SUM(amount) AS total_spend
  FROM credit_card_transcations
  GROUP BY card_type, exp_type, DATEPART(YEAR, transaction_date), DATEPART(MONTH, transaction_date)
)
SELECT TOP 1 *, (total_spend - prev_mont_spend) AS mom_growth
FROM (
  SELECT *, LAG(total_spend,1) OVER(PARTITION BY card_type, exp_type ORDER BY yt,mt) AS prev_mont_spend
  FROM cte
) A
WHERE prev_mont_spend IS NOT NULL AND yt=2014 AND mt=1
ORDER BY mom_growth DESC;
```

---

### **8. Weekend city spend efficiency (Spend / Transaction ratio)**

```sql
SELECT TOP 1 city, SUM(amount)*1.0/COUNT(1) AS ratio
FROM credit_card_transcations
WHERE DATEPART(WEEKDAY, transaction_date) IN (1,7)
GROUP BY city
ORDER BY ratio DESC;
```

---

### **9. City reaching 500th transaction fastest**

```sql
WITH cte AS (
  SELECT *, ROW_NUMBER() OVER(PARTITION BY city ORDER BY transaction_date,transaction_id) AS rn
  FROM credit_card_transcations
)
SELECT TOP 1 city, DATEDIFF(DAY, MIN(transaction_date), MAX(transaction_date)) AS days_taken
FROM cte
WHERE rn=1 OR rn=500
GROUP BY city
HAVING COUNT(1)=2
ORDER BY days_taken;
```

---

## 📊 Key Insights

* Spending is highly concentrated in a few major cities, contributing disproportionately to total spends.
* Certain card types dominate in specific months due to festive or seasonal shopping patterns.
* Gold cards show significant variation across cities, with some cities preferring them much less.
* Female contribution to spends is higher in lifestyle categories compared to fuel/utility.
* City spend behavior during weekends reveals insights about urban leisure vs work-related spends.

---

## 🛠️ Tech Stack

* SQL Server 2019+
* T-SQL (Window Functions, CTEs, Aggregations)
* Dataset from Kaggle

---

## 📌 How to Run

1. Import dataset into SQL Server as `credit_card_transcations`.
2. Apply transformations (lowercase column names, underscores).
3. Copy queries from this README into SSMS or Azure Data Studio.
4. Validate outputs and derive insights.

---

## 🤝 Contributing

Feel free to open issues or PRs to add more queries, optimizations, or insights.

---

## 📅 Last Updated

**2025-09-19**
