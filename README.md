# MIST 4610 – Group Project 2: Northline Outfitters

## Group Information

**Group Name:** [Your Group Name]

- Haylesh Fernandez
- Zain 
- Italia Roman
- Alden Majors

---

## Case Summary

Northline Outfitters is a small online retail company that sells student-friendly lifestyle and tech accessories — including hoodies, water bottles, desk lamps, phone cases, keyboards, mouse pads, and backpacks — to customers in the United States and Canada. The company purchases merchandise from external vendors and sells directly to consumers.

Because the business is still growing, its records are maintained in Excel spreadsheets rather than a proper relational database. We were provided two exports: a `Sales_Dump` containing order line-level transaction data, and a `Product_Supplier_Master` containing product catalog and vendor information.

Both files were intentionally dirty, partially duplicated, and unnormalized. Issues ranged from mixed date formats and inconsistent country codes to blended customer name fields and messy discount and tax representations. Our task was to assess those issues, clean the data, design and implement a normalized relational database, import the cleaned data, and write meaningful SQL queries against it.

---

## Conceptual Model

![Data Model](MIST4610-Data-Model.png)

### Entity & Relationship Explanation

The data model contains 8 entities that reflect the core business objects of Northline Outfitters:

**BUYER** — Stores customer identity information including first name, last name, email, and customer type (Student, Loyalty, Guest, Standard). Each buyer can be associated with many orders, but each order belongs to one buyer.

**EMPLOYEE** — Tracks employees who process orders. The table includes a recursive self-referencing relationship through `Employee_Manager_Id`, meaning each employee can report to a manager who is also stored as an employee in the same table. This captures Northline's internal hierarchy without needing a separate manager table.

**PAYMENT_METHOD** — A lookup entity that stores payment types such as Visa, Mastercard, Debit, and Apple Pay. One payment method can be associated with many orders.

**ORDERS** — The central transaction entity linking together a buyer, an employee, and a payment method. Each order has a sale date, a ship-to destination, and a ship country, and can contain multiple order lines.

**ORDERLINE** — Represents individual line items on an order. Each line is tied to one order and one product, and stores quantity, unit price, discount, tax, return flag, size/weight, notes, and the calculated line total.

**PRODUCT** — Contains the product catalog including SKU, alternate SKU, description, cost, list price, reorder level, pack size, dimensions, discontinued status, and notes. Each product belongs to one category and one vendor. Products also have a self-referencing `Parent_Product_Id` to represent variants such as Mini or Student Edition versions of a base product.

**CATEGORY** — A lookup entity for the seven product categories used by Northline: Tech, Apparel, Audio, School, Accessories, Lifestyle, and Desk Setup. One category can apply to many products.

**VENDOR** — Stores supplier information including vendor name, phone number, and the rep's first and last name. One vendor can supply many products.

---

## Data Quality Assessment

Both source files contained significant quality issues across formatting, structure, and completeness. The main problems are identified below.

### Sales_Dump Issues

| Issue | Description |
|-------|-------------|
| Mixed date formats | `sale_date` appeared in 6+ formats including `Oct 17 25`, `10-11-2025`, and `31/10/2025`. Canadian orders (CORD- prefix) used DD-MM-YYYY while US orders used MM-DD-YYYY, causing ambiguous parses. |
| Inconsistent `ship_country` | Values included `US`, `USA`, `Canada`, `CA`, and 3 null rows. |
| Mixed payment method casing | Values like `visa`, `VISA`, and `MC` referred to the same payment type. |
| Mixed discount formats | Discounts appeared as `10%`, `5`, `promo5`, and `student 10%` across different rows. |
| Mixed tax formats | Tax appeared as `13%`, `0.13`, `HST 13%`, and `8.25%`. 33 rows were legitimately null. |
| Currency prefixes in price fields | `unit_price` and `line_total` had `USD`/`CAD` prefixes and `$` signs mixed with bare numbers. |
| Text embedded in quantity | 32 rows had values like `"2 units"` instead of a plain integer. |
| Blended `customer_info` field | A single field combined first name, last name, and customer type flags (Student, Loyalty, Guest) with inconsistent delimiters (`;`, `|`, `/`). |
| Category variants | Values like `"Tech / Student"` mixed product category with customer type in the same field. |
| Inconsistent `size_or_weight` units | Weight values in oz, g, kg, and lb were mixed with length values in inches and cm, with no unit indicator column. |
| ALL CAPS product descriptions | 11 SKUs had descriptions entered in all caps (e.g., `CAMPUS HOODIE`, `BREEZE RING LIGHT`). |
| Malformed customer emails | 3 emails had typos: `@school.ed`, `@school`, and `@example.caa`. |
| Null customer emails | 50 rows were missing an email address before cleaning. |
| Null return flags | 40 rows had no value in the `return_flag` field. |

### Product_Supplier_Master Issues

| Issue | Description |
|-------|-------------|
| Category inconsistency | 20+ variant strings like `"Tech & Student"` and `"Apparel / Student"` required normalization to the 7 canonical categories. |
| Currency prefixes in cost/price | Values like `USD 6.25` were mixed with bare numerics like `31.4`. |
| Mixed weight units | 5+ unit formats: ounces, grams, kg, lbs, and "pound". |
| Mixed length units | 4+ formats: in, cm, `"` notation, and "centimetres". |
| Exact duplicate rows | Some SKUs appeared up to 3 times with fully identical data. |
| `pack_size` inconsistency | One row had an integer `1` instead of the `"1 each"` text used everywhere else. |
| Null `alt_sku`, `reorder_level`, `pack_size`, `weight_g`, `length_cm` | Multiple fields were null where a consistent value could be inferred from sibling rows sharing the same SKU. |
| `reorder_level` stored as text | 4 rows had the value `"ten"` instead of the integer `10`. |
| Null `parent_sku` on variant rows | 41 variant rows (Mini, Student Edition, color/size variants) were missing a `parent_sku` value. |
| Vendor name typo | `"Urban Sources"` vs. `"Urban Source"` — confirmed to be the same vendor by matching phone number. |
| Vendor phone format inconsistency | Formats mixed `XXX-XXX-XXXX`, `XXX.XXX.XXXX`, and `(XXX)XXX-XXXX`. |
| Vendor rep contact noise | Some rep entries included appended `"/ email missing"` notes or had typos (e.g., `"Mia Dia zFernandez"`). |

---

## Data Cleaning Process

All cleaning was performed using SQL in MySQL Workbench after importing the raw data. The cleaned outputs are saved in `MIST4610-Cleaned-Data.xlsx` as `Sales_Dump_Cleaned` and `Product_Supplier_Cleaned`. A full audit trail of every change made is documented in the `Cleaning_Log` and `Deterministic_Fill_Log` sheets within the same file.

### Sales_Dump Cleaning

**Dates** — All date values were standardized using SQL after import. Canadian orders (identified by the `CORD-` prefix in `order_id`) were treated with DD-MM-YYYY convention; US orders used MM-DD-YYYY. Four specific rows required manual correction after cross-referencing other lines within the same `order_id`:
- LN-019: `09-10-2025` → `2025-10-09`
- LN-042: `09-11-2025` → `2025-11-09`
- LN-105: `08/10/2025` → `2025-10-08`
- LN-118: `04-11-2025` → `2025-11-04`

All dates standardized to ISO `YYYY-MM-DD`.

**Country** — Normalized all `ship_country` values to `US` or `CA`. The 3 null rows were filled using the `order_id` prefix (`CORD` = Canada, `UORD` = US), a pattern that held consistently across all 200 rows.

**Payment Method** — Title-cased all values. Mapped `MC` → `Mastercard`.

**Discounts** — Converted all formats to decimal rates: `10%` → `0.10`, `5` → `0.05`, `promo5` → `0.05`, `student 10%` → `0.10`. Text-labeled values were cleaned using SQL string functions to extract the numeric portion.

**Tax** — Stripped `%` signs and text labels; converted all to decimal rates (`13%` → `0.13`, `8.25%` → `0.0825`). 12 null rows were filled deterministically using the unique non-null tax rate present on another line within the same `order_id`. 33 remaining nulls were left as-is.

**Prices / Line Totals** — Stripped all `$`, `USD`, and `CAD` prefixes; stored as numeric. `line_total` was left null where missing — it does not consistently reconcile to a single formula and was preserved as-is.

**Quantity** — Extracted the numeric portion from values like `"2 units"` using SQL string functions. Stored as integer.

**Customer Info** — Split the blended `customer_info` field into `customer_f_nm`, `customer_l_nm`, and `customer_type` using SQL string functions. The type was identified from embedded keywords (Student, Loyalty, Guest); rows without a keyword were defaulted to `Standard`. Multiple delimiters (`;`, `|`, `/`) were all handled.

**Category** — Extracted the primary category before any `/` or `&` delimiter. Normalized to 7 canonical categories: Tech, Apparel, Audio, School, Accessories, Lifestyle, Desk Setup.

**Size/Weight** — Standardized using SQL by detecting the unit type in each value: weight values (oz, g, kg, lb) were converted to grams; length values (in, `"`, cm) were converted to cm. `"one size"` was preserved as a text value.

**Product Descriptions** — 11 ALL CAPS descriptions were normalized to title case using the `Product_Supplier_Master` as the canonical reference per SKU. An additional 14 rows that had been mapped to wrong variants were corrected by matching the original ALL CAPS value against the product table.

**Emails** — Fixed 3 malformed emails using SQL UPDATE statements. Filled 24 of the 50 null emails by matching on the `(customer_f_nm, customer_l_nm, customer_type, ship_country)` composite key — applied only where that key mapped to exactly one unique email across the dataset. 26 nulls remained unfillable and were left as-is.

**Return Flag** — 40 null rows filled with `"N"`. All 40 had no return-related notes, supporting the assumption that the absence of a flag means no return was recorded.

### Product_Supplier_Master Cleaning

**Category** — Mapped 20+ variant strings to the same 7 canonical categories used in the sales data.

**Cost / List Price** — Stripped `USD`/`CAD` prefixes; stored as numeric.

**Weight** — Converted all weight values to grams; column renamed `weight_g`.

**Length** — Converted all length values to cm; column renamed `length_cm`.

**Exact Duplicates** — Removed fully identical duplicate rows. Variant rows sharing a SKU but differing in description were retained; the `parent_sku` field tracks their relationship to the base product.

**Null Fields** — Filled `alt_sku`, `reorder_level`, `pack_size`, `weight_g`, and `length_cm` deterministically where a consistent non-null value existed across all other rows with the same SKU. Full cell-level detail is logged in the `Deterministic_Fill_Log` sheet.

**Reorder Level** — Converted 4 text values of `"ten"` to the integer `10`. Two rows (ClickStorm Gaming Mouse Mini and TrailSip Bottle) had been incorrectly filled with a sibling-row value and were corrected to `10`.

**Parent SKU** — Filled `parent_sku` for 41 variant rows (Mini, Student Edition, color/size variants) that were missing this value. Set to the base product's SKU consistent with the pattern used throughout the source file.

**Vendor Name** — Corrected `"Urban Sources"` → `"Urban Source"` in 2 rows. Confirmed by matching phone number and majority form (14 rows vs. 2).

**Vendor Phone** — Normalized all formats to `XXX-XXX-XXXX`.

**Vendor Rep** — Stripped `"/ email missing"` noise from rep name fields. Corrected typo `"Mia Dia zFernandez"` → `"Mia Diaz"`. Split into `Vendor_Rep_F_Nm` and `Vendor_Rep_L_Nm` to match the updated data model.

**Pack Size** — Replaced the integer `1` with `"1 each"` for consistency with all other rows.

### SQL Used for Post-Import Standardization

```sql
-- Normalize ship_country after import
UPDATE ORDERS
SET Orders_Ship_Country = 'US'
WHERE Orders_Ship_Country IN ('USA', 'United States');

UPDATE ORDERS
SET Orders_Ship_Country = 'CA'
WHERE Orders_Ship_Country IN ('Canada', 'CAN');

-- Normalize payment method names
UPDATE PAYMENT_METHOD
SET Payment_Nm = 'Mastercard'
WHERE Payment_Nm IN ('MC', 'mastercard', 'MASTERCARD');

UPDATE PAYMENT_METHOD
SET Payment_Nm = CONCAT(UPPER(LEFT(Payment_Nm, 1)), LOWER(SUBSTRING(Payment_Nm, 2)));

-- Set null return flags to 'N'
UPDATE ORDERLINE
SET OrderLine_Return_Flag = 'N'
WHERE OrderLine_Return_Flag IS NULL;

-- Normalize discontinued flag nulls
UPDATE PRODUCT
SET Product_Discontinued = 'N'
WHERE Product_Discontinued IS NULL OR Product_Discontinued = '';

-- Strip currency prefix from cost if imported as string
UPDATE PRODUCT
SET Product_Cost = CAST(REPLACE(REPLACE(Product_Cost, 'USD ', ''), 'CAD ', '') AS DECIMAL(10,2))
WHERE Product_Cost REGEXP '^[A-Z]';
```

---

## Queries

### Required Query 1 — Highest Total Sales Revenue by Country

**Question:** Which products generated the highest total sales revenue, by country?

```sql
SELECT
    ORDERS.Orders_Ship_Country,
    PRODUCT.Product_Description,
    SUM(ORDERLINE.OrderLine_Quantity * ORDERLINE.OrderLine_Unit_Price * (1 - ORDERLINE.OrderLine_Discount)) AS Total_Revenue
FROM ORDERLINE, ORDERS, PRODUCT
WHERE ORDERLINE.ORDERS_Orders_Id = ORDERS.Orders_Id
  AND ORDERLINE.PRODUCT_Product_Id = PRODUCT.Product_Id
GROUP BY ORDERS.Orders_Ship_Country, PRODUCT.Product_Description
ORDER BY ORDERS.Orders_Ship_Country, Total_Revenue DESC;
```

**Results:**

| Ship Country | Product | Total Revenue |
|---|---|---|
| CA | Aurora Mechanical Keyboard | [value] |
| CA | EchoWave Headphones | [value] |
| US | Aurora Mechanical Keyboard | [value] |
| US | Halo Desk Lamp | [value] |
| ... | ... | ... |

---

### Required Query 2 — Employees with Largest Order Counts vs. Manager Peers

**Question:** Which employees handled the largest number of orders, and how do their results compare with other employees under the same manager?

```sql
SELECT
    mgr.Employee_ref AS Manager_Ref,
    emp.Employee_ref AS Employee_Ref,
    COUNT(ORDERS.Orders_Id) AS Orders_Handled
FROM EMPLOYEE emp, EMPLOYEE mgr, ORDERS
WHERE emp.Employee_Manager_Id = mgr.Employee_Id
  AND ORDERS.EMPLOYEE_Employee_Id = emp.Employee_Id
GROUP BY mgr.Employee_ref, emp.Employee_ref
ORDER BY mgr.Employee_ref, Orders_Handled DESC;
```

**Results:**

| Manager | Employee | Orders Handled |
|---|---|---|
| EMU-M01 | EMU-102 | [value] |
| EMU-M01 | EMU-101 | [value] |
| EMU-M03 | EMU-201 | [value] |
| EMU-M03 | EMU-202 | [value] |
| ... | ... | ... |

---

### Required Query 3 — Vendors Supplying Products in More Than One Category

**Question:** Which vendors supply products that appear in more than one category?

```sql
SELECT
    VENDOR.Vendor_Nm,
    COUNT(DISTINCT CATEGORY.Category_Id) AS Num_Categories,
    GROUP_CONCAT(DISTINCT CATEGORY.Category_Nm ORDER BY CATEGORY.Category_Nm SEPARATOR ', ') AS Categories
FROM VENDOR, PRODUCT, CATEGORY
WHERE PRODUCT.VENDOR_Vendor_Id = VENDOR.Vendor_Id
  AND PRODUCT.CATEGORY_Category_Id = CATEGORY.Category_Id
GROUP BY VENDOR.Vendor_Nm
HAVING COUNT(DISTINCT CATEGORY.Category_Id) > 1
ORDER BY Num_Categories DESC;
```

**Results:**

| Vendor | # Categories | Categories |
|---|---|---|
| Urban Source | 4 | Accessories, Desk Setup, Lifestyle, School |
| Vendor North | 3 | Accessories, Desk Setup, Tech |
| Maple Supply | 3 | Accessories, School, Tech |
| ... | ... | ... |

---

### Additional Query 1 — Return Rate by Product Category

**Question:** What is the return rate for each product category?

**Business Justification:** A high return rate in a specific category may point to quality issues, misleading product descriptions, or fulfillment problems. This helps Northline prioritize vendor conversations and review category-level merchandising decisions.

```sql
SELECT
    CATEGORY.Category_Nm,
    COUNT(ORDERLINE.OrderLine_Id) AS Total_Lines,
    SUM(CASE WHEN ORDERLINE.OrderLine_Return_Flag = 'Y' THEN 1 ELSE 0 END) AS Returned_Lines,
    ROUND(
        SUM(CASE WHEN ORDERLINE.OrderLine_Return_Flag = 'Y' THEN 1 ELSE 0 END) * 100.0
        / COUNT(ORDERLINE.OrderLine_Id), 2
    ) AS Return_Rate_Pct
FROM ORDERLINE, PRODUCT, CATEGORY
WHERE ORDERLINE.PRODUCT_Product_Id = PRODUCT.Product_Id
  AND PRODUCT.CATEGORY_Category_Id = CATEGORY.Category_Id
GROUP BY CATEGORY.Category_Nm
ORDER BY Return_Rate_Pct DESC;
```

**Results:**

| Category | Total Lines | Returned Lines | Return Rate % |
|---|---|---|---|
| [category] | [value] | [value] | [value] |
| ... | ... | ... | ... |

---

### Additional Query 2 — Average Discount Rate by Customer Type and Country

**Question:** What is the average discount rate applied to orders broken down by customer type and shipping country?

**Business Justification:** This reveals whether discount policies are being applied consistently across customer segments. If Student customers in Canada are receiving significantly lower discounts than their US counterparts, it may indicate an inconsistency in how promotions were entered at the time of sale.

```sql
SELECT
    BUYER.Customer_Type,
    ORDERS.Orders_Ship_Country,
    ROUND(AVG(ORDERLINE.OrderLine_Discount), 4) AS Avg_Discount_Rate,
    COUNT(ORDERLINE.OrderLine_Id) AS Line_Count
FROM ORDERLINE, ORDERS, BUYER
WHERE ORDERLINE.ORDERS_Orders_Id = ORDERS.Orders_Id
  AND ORDERS.CUSTOMER_Customer_Id = BUYER.Customer_Id
GROUP BY BUYER.Customer_Type, ORDERS.Orders_Ship_Country
ORDER BY BUYER.Customer_Type, ORDERS.Orders_Ship_Country;
```

**Results:**

| Customer Type | Country | Avg Discount | Lines |
|---|---|---|---|
| Guest | CA | [value] | [value] |
| Guest | US | [value] | [value] |
| Loyalty | CA | [value] | [value] |
| ... | ... | ... | ... |

---

### Additional Query 3 — Top Products by Gross Margin

**Question:** Which products generate the highest gross margin per unit sold?

**Business Justification:** Northline buys from external vendors at cost and sells at list price. Knowing which products carry the best margins per unit helps the business decide what to promote, reorder, and prioritize when negotiating with vendors.

```sql
SELECT
    PRODUCT.Product_Sku,
    PRODUCT.Product_Description,
    CATEGORY.Category_Nm,
    VENDOR.Vendor_Nm,
    PRODUCT.Product_List_Price,
    PRODUCT.Product_Cost,
    ROUND(PRODUCT.Product_List_Price - PRODUCT.Product_Cost, 2) AS Gross_Margin_Per_Unit,
    ROUND(
        (PRODUCT.Product_List_Price - PRODUCT.Product_Cost) / PRODUCT.Product_List_Price * 100, 2
    ) AS Margin_Pct
FROM PRODUCT, CATEGORY, VENDOR
WHERE PRODUCT.CATEGORY_Category_Id = CATEGORY.Category_Id
  AND PRODUCT.VENDOR_Vendor_Id = VENDOR.Vendor_Id
  AND PRODUCT.Parent_Product_Id IS NULL
ORDER BY Gross_Margin_Per_Unit DESC;
```

**Results:**

| SKU | Product | Category | Vendor | List Price | Cost | Margin/Unit | Margin % |
|---|---|---|---|---|---|---|---|
| [sku] | [product] | [category] | [vendor] | [value] | [value] | [value] | [value] |
| ... | ... | ... | ... | ... | ... | ... | ... |
