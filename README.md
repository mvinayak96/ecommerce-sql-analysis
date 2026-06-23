# 🗃 E-Commerce SQL Analysis

> Advanced SQL project on an online retail database — 10 business queries covering customer segmentation, inventory management, order analysis, shipping logistics, and product performance using MySQL joins, subqueries, CASE statements, and aggregations.

---

## 📌 Project Overview

This project analyses an online e-commerce retail database to answer 10 real business questions across customer behaviour, product inventory, order fulfilment, and logistics. The queries demonstrate advanced SQL techniques used in real-world data analyst roles.

### Business questions answered:
- How can customers be segmented by registration year into categories A, B, and C?
- Which products have never been sold and what discounts should be applied?
- Which product classes have inventory value exceeding ₹1,00,000?
- Which customers have cancelled every single order they placed?
- How many customers and consignments has DHL handled per city?
- What is the total order value for cash-paying customers with last name starting with G?
- What is the biggest product volume that fits inside Carton ID 10?
- What is the inventory status of each product by category-wise threshold rules?
- Which products are frequently bought together with Product ID 201?
- What are the total quantities shipped for even order IDs to non-5xx pincodes?

---

## 🗂 Project Structure

```
ecommerce-sql-analysis/
├── queries/
│   └── ecommerce_sql_analysis.sql    ← All 10 business queries
└── README.md
```

---

## 🗄 Database Schema

The queries run on an online retail database with the following key tables:

| Table | Description |
|---|---|
| `ONLINE_CUSTOMER` | Customer details — name, email, phone, gender, creation date |
| `ADDRESS` | Customer address — city, pincode, country |
| `PRODUCT` | Product details — description, price, quantity available, dimensions |
| `PRODUCT_CLASS` | Product categories — Electronics, Computers, Mobiles, Watches etc. |
| `ORDER_HEADER` | Order-level data — order status, payment mode, shipper, date |
| `ORDER_ITEMS` | Line-item data — product quantity per order |
| `SHIPPER` | Shipping company details — DHL and others |
| `CARTON` | Carton dimensions — length, width, height for packaging |

---

## 🔍 Query Breakdown

### Query 1 — Customer Segmentation by Registration Year
**Concepts:** `CASE`, `CONCAT`, `UPPER`, `YEAR()`

Displays customer full name with title (MR/MS), email, creation date, and category:
- Category A → registered before 2005
- Category B → registered 2005 to 2010
- Category C → registered 2011 onwards

```sql
SELECT 
    CASE 
        WHEN CUSTOMER_GENDER = 'M' THEN CONCAT('MR. ', UPPER(CUSTOMER_FNAME), ' ', UPPER(CUSTOMER_LNAME))
        WHEN CUSTOMER_GENDER = 'F' THEN CONCAT('MS. ', UPPER(CUSTOMER_FNAME), ' ', UPPER(CUSTOMER_LNAME))
    END AS CUSTOMER_FULL_NAME,
    UPPER(CUSTOMER_EMAIL) AS CUSTOMER_EMAIL_ID,
    CUSTOMER_CREATION_DATE,
    CASE 
        WHEN YEAR(CUSTOMER_CREATION_DATE) < 2005 THEN 'CATEGORY A'
        WHEN YEAR(CUSTOMER_CREATION_DATE) >= 2005 AND YEAR(CUSTOMER_CREATION_DATE) < 2011 THEN 'CATEGORY B'
        ELSE 'CATEGORY C'
    END AS CUSTOMER_CATEGORY
FROM ONLINE_CUSTOMER;
```

---

### Query 2 — Unsold Products with Tiered Discount Pricing
**Concepts:** `CASE`, Subquery, `NOT IN`, `ORDER BY DESC`

Finds products with zero sales and applies discount based on price:
- Price > ₹20,000 → 20% discount
- Price > ₹10,000 → 15% discount
- Price ≤ ₹10,000 → 10% discount

```sql
SELECT 
    P.PRODUCT_ID, P.PRODUCT_DESC, P.PRODUCT_QUANTITY_AVAIL, P.PRODUCT_PRICE,
    (P.PRODUCT_QUANTITY_AVAIL * P.PRODUCT_PRICE) AS INVENTORY_VALUE,
    CASE 
        WHEN P.PRODUCT_PRICE > 20000 THEN P.PRODUCT_PRICE * 0.80
        WHEN P.PRODUCT_PRICE > 10000 THEN P.PRODUCT_PRICE * 0.85
        ELSE P.PRODUCT_PRICE * 0.90
    END AS NEW_PRICE
FROM PRODUCT P
WHERE P.PRODUCT_ID NOT IN (SELECT DISTINCT OI.PRODUCT_ID FROM ORDER_ITEMS OI)
ORDER BY INVENTORY_VALUE DESC;
```

---

### Query 3 — Product Class Inventory Value (> ₹1,00,000)
**Concepts:** `INNER JOIN`, `GROUP BY`, `HAVING`, `SUM`, `ORDER BY DESC`

Summarises inventory value by product class, filtered to classes exceeding ₹1,00,000 total value.

```sql
SELECT 
    PC.PRODUCT_CLASS_CODE, PC.PRODUCT_CLASS_DESC,
    COUNT(PC.PRODUCT_CLASS_CODE) AS COUNT_OF_PRODUCT_TYPE,
    SUM((P.PRODUCT_QUANTITY_AVAIL * P.PRODUCT_PRICE)) AS INVENTORY_VALUE
FROM PRODUCT P
INNER JOIN PRODUCT_CLASS PC ON P.PRODUCT_CLASS_CODE = PC.PRODUCT_CLASS_CODE
GROUP BY PC.PRODUCT_CLASS_CODE, PC.PRODUCT_CLASS_DESC
HAVING INVENTORY_VALUE > 100000
ORDER BY INVENTORY_VALUE DESC;
```

---

### Query 4 — Customers Who Cancelled ALL Their Orders
**Concepts:** Correlated Subquery, `JOIN`, `GROUP BY`, `HAVING COUNT`

Identifies customers where every single order placed is in Cancelled status.

```sql
SELECT 
    OC.CUSTOMER_ID,
    CASE 
        WHEN OC.CUSTOMER_GENDER = 'M' THEN CONCAT('MR. ', UPPER(OC.CUSTOMER_FNAME), ' ', UPPER(OC.CUSTOMER_LNAME))
        WHEN OC.CUSTOMER_GENDER = 'F' THEN CONCAT('MS. ', UPPER(OC.CUSTOMER_FNAME), ' ', UPPER(OC.CUSTOMER_LNAME))
    END AS CUSTOMER_FULL_NAME,
    OC.CUSTOMER_EMAIL, OC.CUSTOMER_PHONE, A.COUNTRY
FROM ONLINE_CUSTOMER OC
JOIN ADDRESS A ON OC.ADDRESS_ID = A.ADDRESS_ID
WHERE OC.CUSTOMER_ID IN (
    SELECT OH.CUSTOMER_ID FROM ORDER_HEADER OH
    WHERE OH.ORDER_STATUS = 'Cancelled'
    GROUP BY OH.CUSTOMER_ID
    HAVING COUNT(*) = (SELECT COUNT(*) FROM ORDER_HEADER WHERE CUSTOMER_ID = OH.CUSTOMER_ID)
);
```

---

### Query 5 — DHL Shipper City-wise Customer and Consignment Count
**Concepts:** Multi-table `JOIN`, `GROUP BY`, `COUNT(DISTINCT)`, `WHERE` filter

Shows each city DHL serves with count of unique customers and total consignments delivered.

---

### Query 6 — Cash Orders for Customers with Last Name Starting 'G'
**Concepts:** `JOIN`, `WHERE LIKE`, `SUM`, `GROUP BY`

Calculates total quantity and value for cash-paying customers whose last name starts with G.

---

### Query 7 — Biggest Order Volume Fitting Carton ID 10
**Concepts:** `JOIN`, dimension comparison (`<=`), `MAX`, `LIMIT`

Finds the order with the single largest product volume that physically fits within Carton ID 10's dimensions.

---

### Query 8 — Category-wise Inventory Status with Nested CASE
**Concepts:** Nested `CASE`, `LEFT JOIN`, `GROUP BY`, category-specific thresholds

Applies different inventory threshold rules per product category:

| Category | Low | Medium | Sufficient |
|---|---|---|---|
| Electronics / Computer | < 10% of sold | < 50% of sold | ≥ 50% of sold |
| Mobiles / Watches | < 20% of sold | < 60% of sold | ≥ 60% of sold |
| All others | < 30% of sold | < 70% of sold | ≥ 70% of sold |

---

### Query 9 — Products Frequently Bought with Product ID 201
**Concepts:** Subquery, `INNER JOIN`, city exclusion (`NOT IN`), `ORDER BY DESC`

Finds products ordered in the same transaction as Product ID 201, excluding shipments to Bangalore and New Delhi.

---

### Query 10 — Even Order IDs Shipped to Non-5xx Pincodes
**Concepts:** Modulo operator (`%`), `NOT LIKE`, multi-table `JOIN`, `GROUP BY`

Displays even-numbered order IDs that are shipped and delivered to pincodes not starting with 5.

---

## 🛠 Tools & Technologies

| Tool | Purpose |
|---|---|
| MySQL 8.0 | Query execution and database management |
| MySQL Workbench | SQL editor and schema visualisation |

---

## 💡 Key SQL Concepts Demonstrated

- `CASE` statements — single and nested, for conditional logic
- Subqueries — correlated and non-correlated
- Multi-table `JOIN` — INNER JOIN, LEFT JOIN across 4+ tables
- Aggregate functions — `SUM`, `COUNT`, `MAX` with `GROUP BY`
- `HAVING` clause — filtering aggregated results
- String functions — `CONCAT`, `UPPER`, `LIKE`, `NOT LIKE`
- Date functions — `YEAR()` for date-based segmentation
- Modulo operator — even/odd filtering with `%`
- `NOT IN` with subquery — exclusion logic

---

## 🚀 How to Run

1. Install MySQL 8.0+ and MySQL Workbench
2. Load the online retail database schema and data
3. Open `queries/ecommerce_sql_analysis.sql` in MySQL Workbench
4. Select any query → press **Ctrl + Shift + Enter** to run selected query
5. Or press **Ctrl + Enter** to run the query at cursor position

---

## 🎓 Course & Certification

This project was developed as part of the:
- **PG Program in Data Science & Analytics** — Great Lakes / Great Learning *(Sep 2024)*

---

## 👤 Author

**Vinayak M**
- 📧 [mvinayak96@gmail.com](mailto:mvinayak96@gmail.com)
- 💼 [LinkedIn](https://www.linkedin.com/in/vinayak-m-b78196ab)
- 🌐 [Portfolio](https://vinayakm.vercel.app)
