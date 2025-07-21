# Daily Full-Refresh ETL Pipeline with Azure Data Factory

This repository documents and supports an automated ETL pattern using Azure Data Factory for daily full-refresh loads from a Data Lake to SQL tables.

---

## Supported File Types

| File Pattern                       | Example                          | Destination Table        | Additional Columns                  |
|-------------------------------------|----------------------------------|-------------------------|-------------------------------------|
| `CUST_MSTR_YYYYMMDD.csv`            | `CUST_MSTR_20191112.csv`         | `CUST_MSTR`             | `Date` (format: `YYYY-MM-DD`)       |
| `master_child_export-YYYYMMDD.csv`  | `master_child_export-20191112.csv`| `master_child`          | `Date` (`YYYY-MM-DD`), `DateKey` (`YYYYMMDD` as integer) |
| `H_ECOM_ORDER.csv`                  | `H_ECOM_ORDER.csv`               | `H_ECOM_Orders`         | *None (load as-is)*                 |

---

## Azure Data Factory Implementation

### 1. **Data Flow Configuration**

- **CUST_MSTR Data Flow**
  - **Source:** Parameterized dataset for dynamic file name.
  - **Derived Column:** Extract date from filename using substring, convert to `yyyy-MM-dd`:
    ```
    formatDateTime(toDate(substring(sourceFileName, 10, 8), 'yyyyMMdd'), 'yyyy-MM-dd')
    ```
  - **Sink:** Mapped to `CUST_MSTR` table.
    - **Table action:** `Truncate table` (configured in Sink transformation).

- **master_child_export Data Flow**
  - **Source:** Parameterized dataset for dynamic file name.
  - **Derived Columns:**
    - `Date`: `formatDateTime(toDate(substring(sourceFileName, 21, 8), 'yyyyMMdd'), 'yyyy-MM-dd')`
    - `DateKey`: `toInteger(substring(sourceFileName, 21, 8))`
  - **Sink:** Mapped to `master_child` table.
    - **Table action:** `Truncate table` (configured in Sink transformation).

- **H_ECOM_ORDER Data Flow**
  - **Source:** Parameterized dataset.
  - **Transformation:** Optional deduplication (e.g., group by OrderID, first() for other columns).
  - **Sink:** Mapped to `H_ECOM_Orders` table.

---

### 2. **Pipeline Orchestration**

- **File Discovery:**  
  Use Get Metadata or List Files activity to enumerate all files in the container.

- **Filtering & Routing:**  
  Use Filter or If Condition activities to separate files by type.

- **ForEach Activity:**  
  Iterate over files for each type and invoke the correct Data Flow with filename parameter.

- **Truncate Logic:**
  - For `CUST_MSTR` and `master_child` tables, the Data Flow Sink action handles truncation automatically.
  - For `H_ECOM_Orders`, add a Script activity **before** the Data Flow:
    ```sql
    TRUNCATE TABLE dbo.H_ECOM_Orders;
    ```
    > *If foreign key constraints exist, use `DELETE FROM dbo.H_ECOM_Orders;` instead.*

- **Data Loading:**  
  Each Data Flow loads its output to the corresponding SQL table after truncation.

---

### 3. **Daily Operation**

- The pipeline is scheduled for daily execution.
- Each run performs a **full-refresh**: destination tables are cleared and new data is loaded.

---

## Best Practices

- **Permissions:** Ensure your ADF managed identity or linked service has permissions for table truncation/deletion.
- **Testing:** Run pipelines in debug mode and validate target tables after each load.
- **Error Handling:** Monitor for truncation or PK violation errors, especially in the Script activity for `H_ECOM_Orders`.
- **Parameterization:** Use dataset and pipeline parameters to handle dynamic file names and support additional file types easily.

