---
title: Excel fundamentals revision pack
tags: []
created: 2026-03-02
updated:
status: draft
---
# 📊 Sample Dataset: Sales Data

|OrderID|Date|Region|Salesperson|Product|Category|Units|Unit Price|Payment Mode|Customer Type|
|---|---|---|---|---|---|---|---|---|---|
|1001|01-01-2026|North|Rahul|Laptop|Electronics|2|55000|Card|Retail|
|1002|02-01-2026|South|Anjali|Printer|Electronics|1|12000|Cash|Corporate|
|1003|03-01-2026|East|Vikram|Desk Chair|Furniture|4|4500|UPI|Retail|
|1004|04-01-2026|West|Sneha|Monitor|Electronics|3|15000|Card|Corporate|
|1005|05-01-2026|North|Rahul|Table|Furniture|2|8000|Cash|Retail|
|1006|06-01-2026|South|Anjali|Laptop|Electronics|1|60000|UPI|Corporate|
|1007|07-01-2026|East|Vikram|Mouse|Accessories|5|800|Card|Retail|
|1008|08-01-2026|West|Sneha|Keyboard|Accessories|6|1200|Cash|Retail|
|1009|09-01-2026|North|Rahul|Printer|Electronics|2|11000|UPI|Corporate|
|1010|10-01-2026|South|Anjali|Desk Chair|Furniture|3|5000|Card|Retail|

---

# 🟢 LEVEL 1: Basic Tasks (Formulas + Formatting)

### 1️⃣ Add a Total Sales Column

Create a new column:

```
Total Sales = Units * Unit Price
```

### 2️⃣ Calculate:

- Total Revenue
- Average Revenue per Order
- Maximum Sale Value 
- Minimum Sale Value

Use:

- `SUM()`    
- `AVERAGE()`
- `MAX()`
- `MIN()`

### 3️⃣ Count:

- Total number of orders → `COUNT()`
- Orders from North region → `COUNTIF()`
- Electronics category orders → `COUNTIF()`

### 4️⃣ Conditional Formatting

Highlight:

- Sales greater than ₹50,000
- Units greater than 4

# 🟡 LEVEL 2: Logical + Text Functions

### 5️⃣ Add Performance Column

If Total Sales > 50,000 → "High"  
Else → "Normal"

Use:

```
IF()
```

### 6️⃣ Extract Month from Date

Create a column:

```
=MONTH(Date)
```

### 7️⃣ Combine Fields

Create a column:

```
Region - Salesperson
```

Use:

```
CONCAT() or &
```

### 8️⃣ Round Off Total Sales

Use:

```
ROUND()
```

# 🔵 LEVEL 3: Sorting, Filtering, and Data Analysis

### 9️⃣ Sort:

- Highest to lowest Total Sales
- Region A → Z

### 🔟 Filter:

- Only Electronics category
- Only Corporate customers
- Sales above ₹30,000

# 🟣 LEVEL 4: Lookup & Reference

Create a new table:

## Commission Table

|Salesperson|Commission %|
|---|---|
|Rahul|5%|
|Anjali|6%|
|Vikram|4%|
|Sneha|5%|
### Use:

```
VLOOKUP() or XLOOKUP()
```

To fetch commission %.

### Calculate Commission Amount

```
Commission = Total Sales * Commission %
```

# 🟠 LEVEL 5: Pivot Tables

Create Pivot Tables to answer:

1. Total Sales by Region
2. Total Sales by Category
3. Total Sales by Salesperson
4. Count of Orders by Payment Mode
5. Region vs Category matrix

# 🔴 LEVEL 6: Charts

Create:

- Column chart → Sales by Region
- Pie chart → Sales by Category
- Line chart → Sales over Date
- Bar chart → Salesperson Performance

# 🧹 LEVEL 7: Data Cleaning Practice

Practice:

- Remove duplicates
- Convert text dates to proper date format
- Check for blanks
- Trim extra spaces (`TRIM()`)
- Convert text to proper case (`PROPER()`)
