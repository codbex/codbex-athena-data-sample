# <img src="https://www.codbex.com/icon.svg" width="32" style="vertical-align: middle;"> codbex-athena-data-sample

## 📖 Table of Contents
* [📦 Data](#-data)
* [🐳 Local Development with Docker](#-local-development-with-docker)
* [🔍 Sample Reports in SQL format](#-sample-reports-in-sql-format)

## 📦 Data 

* [Companies](https://github.com/codbex/codbex-athena-data-sample/blob/main/codbex-athena-data-sample/companies.csv)
* [Contacts](https://github.com/codbex/codbex-athena-data-sample/blob/main/codbex-athena-data-sample/contacts.csv)
* [Customer payments](https://github.com/codbex/codbex-athena-data-sample/blob/main/codbex-athena-data-sample/customer-payments.csv)
* [Customers](https://github.com/codbex/codbex-athena-data-sample/blob/main/codbex-athena-data-sample/customers.csv)
* [Employees](https://github.com/codbex/codbex-athena-data-sample/blob/main/codbex-athena-data-sample/employees.csv)
* [Journal Entries](https://github.com/codbex/codbex-athena-data-sample/blob/main/codbex-athena-data-sample/journal-entries.csv)
* [Payment Adjustments](https://github.com/codbex/codbex-athena-data-sample/blob/main/codbex-athena-data-sample/payment-adjustments.csv)
* [Purchase Invoice Items](https://github.com/codbex/codbex-athena-data-sample/blob/main/codbex-athena-data-sample/purchase-invoice-items.csv)
* [Purchase Invoices](https://github.com/codbex/codbex-athena-data-sample/blob/main/codbex-athena-data-sample/purchase-invoices.csv)
* [Sales Invoice Items](https://github.com/codbex/codbex-athena-data-sample/blob/main/codbex-athena-data-sample/sales-invoice-items.csv)
* [Sales Invoices](https://github.com/codbex/codbex-athena-data-sample/blob/main/codbex-athena-data-sample/sales-invoices.csv)
* [Supplier Payments](https://github.com/codbex/codbex-athena-data-sample/blob/main/codbex-athena-data-sample/supplier-payments.csv)
* [Suppliers](https://github.com/codbex/codbex-athena-data-sample/blob/main/codbex-athena-data-sample/suppliers.csv)

## 🐳 Local Development with Docker

When running this project inside the codbex Atlas Docker image, you must provide authentication for installing dependencies from GitHub Packages.
1. Create a GitHub Personal Access Token (PAT) with `read:packages` scope.
2. Pass `NPM_TOKEN` to the Docker container:

    ```
    docker run \
    -e NPM_TOKEN=<your_github_token> \
    --rm -p 80:80 \
    ghcr.io/codbex/codbex-atlas:latest
    ```

⚠️ **Notes**
- The `NPM_TOKEN` must be available at container runtime.
- This is required even for public packages hosted on GitHub Packages.
- Never bake the token into the Docker image or commit it to source control.


## 🔍 Sample Reports in SQL format


- [Finance](#finance)
- [Cashflow](#cashflow)
- [Inventory](#inventory)
- [Customers](#customers)

## Finance
### Cashflow

- Net Cashflow
  ```sql
  SELECT 
      SUM(TRANSACTION_AMOUNT) AS CASHFLOW_NET,
      DATE_TRUNC('day', TRANSACTION_DATE) AS CASHFLOW_DATE
  FROM (
      SELECT 
          SALESINVOICE_DATE AS TRANSACTION_DATE,
          SALESINVOICE_NET AS TRANSACTION_AMOUNT
      FROM CODBEX_SALESINVOICE
      UNION ALL
      SELECT 
          PURCHASEINVOICE_DATE AS TRANSACTION_DATE,
          -PURCHASEINVOICE_NET AS TRANSACTION_AMOUNT
      FROM CODBEX_PURCHASEINVOICE
  ) AS CombinedData
  GROUP BY DATE_TRUNC('day', TRANSACTION_DATE)
  ORDER BY CASHFLOW_DATE DESC;
  ```

- VAT each month(sum(sales_invoices.vat) - sum(purchase_invoices.vat))

  ```sql
  SELECT 
      SUM(VAT) AS CASHFLOW_VAT,
      DATE_TRUNC('month', TRANSACTION_DATE) AS CASHFLOW_DATE
  FROM (
      SELECT 
          SALESINVOICE_DATE AS TRANSACTION_DATE,
          SALESINVOICE_VAT AS VAT
      FROM CODBEX_SALESINVOICE
      UNION ALL
      SELECT 
          PURCHASEINVOICE_DATE AS TRANSACTION_DATE,
          -PURCHASEINVOICE_VAT AS VAT
      FROM CODBEX_PURCHASEINVOICE
  ) AS CombinedData
  GROUP BY DATE_TRUNC('month', TRANSACTION_DATE)
  ORDER BY CASHFLOW_DATE DESC;
  ```

### Inventory

- Quantity left in inventory from `STOCK_RECORD`

  ```sql
  SELECT 
      p.PRODUCT_NAME,
      SUM(s.STOCKRECORD_DIRECTION) AS sum_direction
  FROM 
      CODBEX_STOCKRECORD s
  JOIN 
      CODBEX_PRODUCT p ON s.STOCKRECORD_PRODUCT = p.PRODUCT_ID
  GROUP BY 
      p.PRODUCT_NAME;
  ```

- Quantity left in inventory with `STOCKADJUSTMENT` and `STOCKADJUSTMENTITEM`

  ```sql
  SELECT 
      p.PRODUCT_NAME,
      SUM(s.STOCKRECORD_DIRECTION) +
      COALESCE((
          SELECT SUM(si.STOCKADJUSTMENTITEM_ADJUSTEDQUANTITY)
          FROM CODBEX_STOCKADJUSTMENTITEM si
          JOIN CODBEX_STOCKADJUSTMENT sa ON si.STOCKADJUSTMENTITEM_STOCKADJUSTMENT = sa.STOCKADJUSTMENT_ID
          WHERE si.STOCKADJUSTMENTITEM_PRODUCT = p.PRODUCT_ID
      ), 0) AS QUANTITY_LEFT
  FROM 
      CODBEX_PRODUCT p
  LEFT JOIN 
      CODBEX_STOCKRECORD s ON s.STOCKRECORD_PRODUCT = p.PRODUCT_ID
  GROUP BY 
      p.PRODUCT_NAME;
  ```

- Products ranked by number of sales

  ```sql
  SELECT
      p.PRODUCT_NAME,
      COUNT(si.SALESINVOICEITEM_ID) AS order_count
  FROM
      CODBEX_PRODUCT p
  JOIN
      CODBEX_SALESINVOICEITEM si ON p.PRODUCT_ID = si.SALESINVOICEITEM_PRODUCT
  GROUP BY
      p.PRODUCT_ID,
      p.PRODUCT_NAME
  ORDER BY
      order_count DESC;
  ```

- Products availability ordered by number of sales

  ```sql
  SELECT
      p.PRODUCT_NAME,
      COALESCE(SUM(si.SALESINVOICEITEM_QUANTITY), 0) AS total_sold,
      SUM(s.STOCKRECORD_DIRECTION) AS sum_direction
  FROM
      CODBEX_PRODUCT p
          LEFT JOIN
      CODBEX_SALESINVOICEITEM si ON p.PRODUCT_ID = si.SALESINVOICEITEM_PRODUCT
          LEFT JOIN
      CODBEX_STOCKRECORD s ON p.PRODUCT_ID = s.STOCKRECORD_PRODUCT
  GROUP BY
      p.PRODUCT_NAME
  ORDER BY
      total_sold DESC;
  ```

- Product categories ordered by number of sales

  ```sql
  SELECT
      pc.PRODUCTCATEGORY_NAME,
      COUNT(si.SALESINVOICEITEM_ID) AS sales_count
  FROM
      CODBEX_PRODUCTCATEGORY pc
  JOIN
      CODBEX_PRODUCT p ON pc.PRODUCTCATEGORY_ID = p.PRODUCT_CATEGORY
  JOIN
      CODBEX_SALESINVOICEITEM si ON p.PRODUCT_ID = si.SALESINVOICEITEM_PRODUCT
  GROUP BY
      pc.PRODUCTCATEGORY_NAME
  ORDER BY
      sales_count DESC;
  ```

- Manufacturers ordered by number of sales

  ```sql
  SELECT
  m.MANUFACTURER_NAME,
  COUNT(si.SALESINVOICEITEM_ID) AS sales_count
  FROM
  CODBEX_MANUFACTURER m
  LEFT JOIN
  CODBEX_PRODUCT p ON m.MANUFACTURER_ID = p.PRODUCT_MANUFACTURER
  LEFT JOIN
  CODBEX_SALESINVOICEITEM si ON p.PRODUCT_ID = si.SALESINVOICEITEM_PRODUCT
  GROUP BY
  m.MANUFACTURER_NAME
  ORDER BY
  sales_count DESC;
  ```

### Customers

- Customers ranked by number of sales

  ```sql
  SELECT
      c.CUSTOMER_NAME,
      COUNT(si.SALESINVOICE_ID) AS invoice_count
  FROM
      CODBEX_CUSTOMER c
  LEFT JOIN
      CODBEX_SALESINVOICE si ON c.CUSTOMER_ID = si.SALESINVOICE_CUSTOMER
  GROUP BY
      c.CUSTOMER_ID,
      c.CUSTOMER_NAME
  ORDER BY
      invoice_count DESC;
  ```
