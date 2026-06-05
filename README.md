# Inventory Demand and Availability Management (Migration and Environment Transition)

## Executive Summary
Lending and retail operations require tight alignment between inventory availability and customer demand. A breakdown in this alignment leads to supply shortages, which represent lost revenue opportunities and reduced customer satisfaction. 

This project implements an inventory tracking system that monitors product availability, customer demand, and stockout costs. The project resolves two critical backend challenges:
1. **Environment Transition**: Promoting the dashboard from a development/test environment (99 transaction records) to a production database (1,043 transaction records).
2. **Database Engine Migration**: Transitioning the entire data source engine from Microsoft SQL Server to MySQL Database. This migration is executed via Power Query Advanced Editor modifications to avoid rebuilding the semantic model or breaking existing DAX metrics.

---

## Analytical Findings and Risk Drivers

Analysis of the production dataset reveals a significant supply-demand mismatch across the product line:

### Core Inventory Metrics (Production Environment)
* **Average Demand per Day**: 48.65 units
* **Average Availability per Day**: 24.70 units
* **Total Supply Shortage**: 61,000 units (representing cumulative unfulfilled demand)

### Financial Exposure
* **Total Realized Profit**: $301,000 (generated from days where availability successfully met or exceeded demand)
* **Total Revenue Loss**: $8,000,000 (the cumulative value of unfulfilled demand, where demand exceeded availability)
* **Average Daily Loss**: $2,970 per day due to stockouts

The data demonstrates that availability meets less than half of the daily customer demand, resulting in an $8M revenue deficit. Underwriting and inventory teams must re-evaluate procurement cycles and safety stock levels to capture this unfulfilled market.

---

## Actionable Lending Recommendations
Based on the default and supply concentrations identified in the data:
1. **Optimize Procurement Cycles**: Increase safety stock thresholds for products showing chronic daily shortages to capture the $8M in unfulfilled demand.
2. **Dynamic Stock Reallocation**: Implement a dynamic allocation model to shift stock to locations or channels exhibiting highest demand rates.
3. **Underwriting and Collateral Risk Controls**: Align customer credit terms and borrowing limits with the supply availability of financed inventory to mitigate portfolio risk.

---

## Dashboard Structure

The reporting layout adopts a minimalist executive scorecard design. Rather than crowding the canvas with complex visualizations, the interface focuses on three high-impact KPI banners per page. This layout is optimized for rapid mobile and tablet viewing by C-suite executives who require immediate visibility into portfolio health and stockout costs without visual clutter.

### Page 1: Demand and Availability Overview
* **Focus**: High-level KPIs including Average Demand per Day, Average Availability per Day, and Total Supply Shortage.
* **Visuals**: Tracks core operational supply metrics to identify structural shortages, using page-level filters for Product Name and Order Date.

  <img width="1407" height="789" alt="Screenshot 2026-06-06 041819" src="https://github.com/user-attachments/assets/64cbbb43-d5ad-4b35-b258-a83661b5a835" />


### Page 2: Financial Impact Analysis
* **Focus**: Financial outcomes including Total Profit, Total Loss, and Average Daily Loss.
* **Visuals**: Translates physical inventory shortages into financial terms to demonstrate the cost of stockouts, sharing the same page-level filters.

  <img width="1406" height="790" alt="Screenshot 2026-06-06 041836" src="https://github.com/user-attachments/assets/0a9fcc53-aaa2-4e27-be9e-872bc0a06477" />


---

## Data Architecture and Pipeline

The reporting pipeline is designed to handle large datasets and source database migrations:

```
[SQL Server or MySQL Database] 
      │
      ▼ (On-Premises Data Gateway: Df1)
[Power BI Dataflow (Power Query Online)] ──► Centralized Transformations
      │
      ▼ (Advanced M-Query Source Swap)
[Power BI Semantic Model (Import Mode)]  ──► Reconciled against Excel Pivot Tables
      │
      ▼
[Power BI Dashboard Reports]
```

### Gateway and Connection Setup
* **Gateway Connection**: An On-Premises Data Gateway (Standard Mode, named `Df1`) connects the Power BI service to the local databases.
* **Centralized ETL**: Data ingestion, normalization, and type formatting are handled in Power Query Online (Dataflows Gen 1), creating a reusable source of truth.

### Source Migration: SQL Server to MySQL
Due to infrastructure changes, the backend was migrated from SQL Server to MySQL. Swapping the connection type was achieved without breaking the reports using the following steps:

1. **SQL Query Translation**: MySQL does not support the `SELECT INTO` syntax used in SQL Server. The query was translated into a `CREATE TABLE AS SELECT` operation:
   * *SQL Server Query*:
     ```sql
     SELECT a.order_date, a.product_id, a.availability, a.demand, b.product_name, b.unit_price 
     INTO new_table
     FROM prod_environment_inventory_dataset a 
     LEFT JOIN products b ON a.product_id = b.product_id;
     ```
   * *MySQL Query*:
     ```sql
     CREATE TABLE new_table AS 
     SELECT a.order_date AS order_date, a.product_id AS product_id, a.availability AS availability, 
            a.demand AS demand, b.product_name AS product_name, b.unit_price AS unit_price 
     FROM prod_environment_inventory_dataset a 
     LEFT JOIN products b ON a.product_id = b.product_id;
     ```
2. **Connector Dependency**: Swapped connections require the local installation of the **MySQL Connector/NET** component.
3. **Advanced Editor Connection Swap**: The connection string was updated in the Advanced Editor to prevent breaking downstream steps:
   * *Original SQL Server*: `Source = Sql.Database("localhost", "prod", [Query="SELECT * FROM new_table"])`
   * *Updated MySQL*: `Source = MySQL.Database("localhost", "prod", [Query="SELECT * FROM new_table"])`

### Data Quality and Referential Integrity Fixes
Fact tables originally contained invalid product IDs 21 and 22, which did not exist in the dimension table `products` (limited to IDs 1–20). These were mapped to valid IDs at the source database layer before merging:
```sql
UPDATE prod_environment_inventory_dataset SET product_id = 7 WHERE product_id = 21;
UPDATE prod_environment_inventory_dataset SET product_id = 11 WHERE product_id = 22;
```

---

## Analytical DAX Implementation

The semantic model uses row-level iterators (`SUMX`) combined with conditional table filters to calculate financial impact, segregating daily demand shortages from inventory surpluses.

### Profit/Loss Variance (Calculated Column)
Calculates the physical unit surplus or deficit at the transaction grain.
```dax
loss_profit = demand_availability_data[availability] - demand_availability_data[demand]
```

### Total Realized Profit
Filters the dataset to rows with inventory surpluses and multiplies the surplus quantity by the unit price.
```dax
total profit = 
SUMX(
    FILTER(
        demand_availability_data,
        demand_availability_data[loss_profit] > 0
    ),
    demand_availability_data[loss_profit] * demand_availability_data[unit_price]
)
```

### Total Stockout Loss
Isolates rows where demand exceeded availability, calculates the value of the unfulfilled orders, and multiplies by `-1` to represent the loss as a positive value on the dashboard.
```dax
total loss = 
SUMX(
    FILTER(
        demand_availability_data,
        demand_availability_data[loss_profit] < 0
    ),
    demand_availability_data[loss_profit] * demand_availability_data[unit_price]
) * -1
```

### Average Daily Loss
Divides the total calculated loss by the distinct count of days in the transaction period.
```dax
average loss per day = 
DIVIDE(
    [total loss],
    DISTINCTCOUNT(demand_availability_data[order_date]),
    0
)
```
