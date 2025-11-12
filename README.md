# Insight-Engine

A robust **SQL Server Data Warehouse** solution that implements a multi-layered ETL pipeline to transform raw sales data from disparate sources into actionable business intelligence. The system follows the **Bronze-Silver-Gold** architectural pattern to progressively clean, transform, and model data for enterprise analytics and reporting.

## Table of Contents
- [Project Overview](#project-overview)
- [High-Level Architecture](#high-level-architecture)
- [Low-Level Design](#low-level-design)
- [Directory and Code Structure](#directory-and-code-structure)
- [Data Flow and Layers](#data-flow-and-layers)
- [Scope and Future Expansion](#scope-and-future-expansion)
- [Database Migration to PostgreSQL](#database-migration-to-postgresql)
- [Setup Instructions](#setup-instructions)
- [Quality Assurance](#quality-assurance)
- [Contributing](#contributing)

---

## Project Overview

### Purpose
**Insight-Engine** is a comprehensive data warehouse solution designed to consolidate and analyze sales data from multiple heterogeneous source systems (CRM and ERP). The project addresses the challenge of fragmented business data by creating a unified, clean, and analytics-ready data model.

### Problem Statement
Organizations often struggle with:
- **Data Silos**: Sales data scattered across CRM and ERP systems with different formats and naming conventions
- **Data Quality Issues**: Inconsistent formatting, missing values, future dates, and non-standardized categorical values
- **Complex Integration**: Difficulty joining customer, product, and sales information from disparate sources
- **Analytics Barriers**: Raw data not suitable for direct business intelligence and reporting

### Solution
Insight-Engine implements a **medallion architecture** (Bronze → Silver → Gold) that:
1. **Ingests** raw data from CSV files into Bronze layer tables
2. **Cleanses and transforms** data in the Silver layer (standardization, validation, enrichment)
3. **Models** data into a **Star Schema** in the Gold layer for optimized analytics
4. **Enables** business users to generate insights through clean dimensional models

### Key Features
- ✅ **Multi-Source Integration**: Combines data from CRM and ERP systems
- ✅ **Data Quality**: Automated cleansing, validation, and standardization
- ✅ **Star Schema Design**: Optimized dimensional model (Facts & Dimensions)
- ✅ **Incremental ETL**: Stored procedures for Bronze, Silver, and Gold layer loading
- ✅ **Quality Checks**: Built-in data validation and integrity tests
- ✅ **Scalable Architecture**: Modular design supports future data sources
- ✅ **Documentation**: Comprehensive data catalog and naming conventions

---

## High-Level Architecture

### System Architecture Diagram

### Architectural Layers

The system implements a **three-tier medallion architecture**:

```
┌─────────────────────────────────────────────────────────────────┐
│                        DATA SOURCES                             │
│  ┌──────────────────┐              ┌──────────────────┐         │
│  │   CRM System     │              │   ERP System     │         │
│  │  - Customer Info │              │  - Location Data │         │
│  │  - Product Info  │              │  - Customer Demo │         │
│  │  - Sales Details │              │  - Categories    │         │
│  └──────────────────┘              └──────────────────┘         │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                     BRONZE LAYER (Raw Data)                     │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Stored Procedure: bronze.load_bronze                      │ │
│  │  • BULK INSERT from CSV files                              │ │
│  │  • Exact copy of source data (no transformations)          │ │
│  │  • Tables: crm_*, erp_*                                    │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   SILVER LAYER (Cleansed Data)                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Stored Procedure: silver.load_silver                      │ │
│  │  • Data cleansing and standardization                      │ │
│  │  • Business logic transformations                          │ │
│  │  • Date conversions and validation                         │ │
│  │  • Key derivation and enrichment                           │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                  GOLD LAYER (Business Model)                    │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Views: Star Schema for Analytics                          │ │
│  │  • dim_customers  (Customer Dimension)                     │ │
│  │  • dim_products   (Product Dimension)                      │ │
│  │  • fact_sales     (Sales Fact Table)                       │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
                   BI Tools & Reporting
```

### Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Database** | Microsoft SQL Server | Data warehouse platform |
| **Language** | T-SQL | ETL logic, stored procedures, views |
| **Data Format** | CSV | Source data files |
| **Architecture** | Medallion (Bronze-Silver-Gold) | Data quality and progressive refinement |
| **Schema Design** | Star Schema | Dimensional modeling for analytics |

---

## Low-Level Design

### Data Flow Diagram
### ETL Process Flow

#### 1. Bronze Layer Loading (`bronze.load_bronze`)
```
Source CSV → BULK INSERT → Bronze Tables
```
- **Input**: Raw CSV files from `datasets/source_crm/` and `datasets/source_erp/`
- **Process**:
  1. Truncate existing Bronze tables
  2. Execute BULK INSERT for each source file
  3. Track load duration for monitoring
- **Output**: Raw data stored exactly as received
- **Tables Created**:
  - `bronze.crm_cust_info`
  - `bronze.crm_prd_info`
  - `bronze.crm_sales_details`
  - `bronze.erp_loc_a101`
  - `bronze.erp_cust_az12`
  - `bronze.erp_px_cat_g1v2`

#### 2. Silver Layer Transformation (`silver.load_silver`)
```
Bronze Tables → Data Cleansing → Business Logic → Silver Tables
```
- **Input**: Bronze layer tables
- **Process**:
  1. **Data Cleansing**:
     - Remove unwanted prefixes (e.g., 'AW-', 'NAS')
     - Trim whitespace from strings
     - Standardize categorical values (gender, marital status, product lines)
  2. **Data Validation**:
     - Convert integer dates to proper DATE format (YYYYMMDD → DATE)
     - Set future birthdates to NULL
     - Calculate product end dates using LEAD window function
  3. **Data Enrichment**:
     - Extract category IDs from product keys
     - Map product line codes to descriptive names (M→Mountain, R→Road)
     - Handle NULL values with ISNULL/COALESCE
- **Output**: Cleaned, validated, and enriched data
- **Transformation Examples**:
  ```sql
  -- Customer ID normalization
  CASE 
    WHEN cid LIKE 'AW-%' THEN SUBSTRING(cid, 4, LEN(cid))
    WHEN cid LIKE 'NAS%' THEN SUBSTRING(cid, 4, LEN(cid))
    ELSE cid
  END
  
  -- Date conversion (from integer YYYYMMDD to DATE)
  CAST(CAST(sls_order_dt AS NVARCHAR) AS DATE)
  
  -- Product line standardization
  CASE 
    WHEN UPPER(TRIM(prd_line)) = 'M' THEN 'Mountain'
    WHEN UPPER(TRIM(prd_line)) = 'R' THEN 'Road'
    ELSE 'n/a'
  END
  ```

#### 3. Gold Layer Modeling (Views)
```
Silver Tables → Joins & Aggregations → Dimensional Views
```
- **Input**: Silver layer tables
- **Process**:
  1. Create **Dimension Views**:
     - Join related Silver tables (e.g., CRM + ERP customer data)
     - Add surrogate keys using ROW_NUMBER()
     - Apply business naming conventions
  2. Create **Fact Views**:
     - Join fact data with dimensions via surrogate keys
     - Preserve transactional grain
- **Output**: Star schema ready for BI tools
- **Schema**:
  ```
  fact_sales (Fact)
      ├── customer_key → dim_customers (Dimension)
      └── product_key → dim_products (Dimension)
  ```

### Data Model

**Star Schema Components:**

1. **Fact Table**: `gold.fact_sales`
   - Grain: One row per sales order line item
   - Measures: sales_amount, quantity, price
   - Foreign Keys: customer_key, product_key

2. **Dimension Tables**:
   - `gold.dim_customers`: Customer demographics and geographic info
   - `gold.dim_products`: Product hierarchy with categories and attributes

---

## Directory and Code Structure

### Project Structure
```
Insight-Engine/
│
├── datasets/                      # Source data files
│   ├── source_crm/               # CRM system CSV exports
│   │   ├── cust_info.csv        # Customer master data
│   │   ├── prd_info.csv         # Product master data
│   │   └── sales_details.csv    # Sales transactions
│   │
│   └── source_erp/               # ERP system CSV exports
│       ├── LOC_A101.csv         # Customer location data
│       ├── CUST_AZ12.csv        # Customer demographics
│       └── PX_CAT_G1V2.csv      # Product categories
│
├── scripts/                      # SQL scripts for ETL
│   ├── init_database.sql        # Database and schema creation
│   │
│   ├── bronze/                   # Bronze layer scripts
│   │   ├── ddl_bronze.sql       # Create Bronze tables
│   │   └── proc_load_bronze.sql # Load Bronze data (BULK INSERT)
│   │
│   ├── silver/                   # Silver layer scripts
│   │   ├── ddl_silver.sql       # Create Silver tables
│   │   └── proc_load_silver.sql # Transform Bronze → Silver
│   │
│   └── gold/                     # Gold layer scripts
│       └── ddl_gold.sql         # Create Gold views (Star Schema)
│
├── tests/                        # Data quality checks
│   ├── quality_checks_silver.sql # Silver layer validation
│   └── quality_checks_gold.sql   # Gold layer validation
│
├── docs/                         # Documentation and diagrams
│   ├── naming_conventions.md    # Naming standards
│   ├── data_catalog.md          # Data dictionary for Gold layer
│   └── *.pdf                    # Additional documentation
│
└── README.md                     # This file
```

### Component Descriptions

#### `/datasets`
Contains raw CSV files exported from source systems. These files are ingested into the Bronze layer.
- **CRM files**: Customer, product, and sales transaction data
- **ERP files**: Additional customer demographics, location, and product categorization

#### `/scripts`
SQL scripts organized by data layer:

1. **`init_database.sql`**:
   - Creates `DataWarehouse` database
   - Creates schemas: `bronze`, `silver`, `gold`
   - **WARNING**: Drops existing database if present

2. **`bronze/` Scripts**:
   - **`ddl_bronze.sql`**: Defines raw data table structures matching source formats
   - **`proc_load_bronze.sql`**: Stored procedure to BULK INSERT CSV data
   - **Execution**: `EXEC bronze.load_bronze;`

3. **`silver/` Scripts**:
   - **`ddl_silver.sql`**: Defines cleansed table structures (adds `dwh_create_date`)
   - **`proc_load_silver.sql`**: Implements data cleansing, standardization, and transformation logic
   - **Execution**: `EXEC silver.load_silver;`

4. **`gold/` Scripts**:
   - **`ddl_gold.sql`**: Creates dimension and fact views implementing Star Schema
   - Views: `dim_customers`, `dim_products`, `fact_sales`

#### `/tests`
SQL scripts for data quality validation:
- **`quality_checks_silver.sql`**: Validates data cleansing (NULL checks, duplicates, standardization)
- **`quality_checks_gold.sql`**: Validates dimensional model integrity (surrogate keys, referential integrity)

#### `/docs`
Project documentation including:
- **Naming conventions**: Standards for tables, columns, procedures
- **Data catalog**: Data dictionary for Gold layer
- **Visual diagrams**: Architecture, flow, and data model

---

## Data Flow and Layers

### Layer Interactions

### Bronze Layer (Raw/Landing Zone)
**Purpose**: Store raw data exactly as received from source systems

**Characteristics**:
- No data transformations applied
- Direct BULK INSERT from CSV files
- Preserves original data types and formats
- Temporary storage for initial ingestion

**Tables**:
- Source system prefix retained (e.g., `crm_`, `erp_`)
- No business logic or data quality rules applied

**Usage**:
```sql
-- Load Bronze layer
EXEC bronze.load_bronze;
```

### Silver Layer (Cleansed/Conformed)
**Purpose**: Apply data quality rules and business transformations

**Characteristics**:
- Data cleansing (trim spaces, remove prefixes)
- Data validation (date ranges, NULL handling)
- Data standardization (categorical value mapping)
- Adds technical columns (`dwh_create_date`)

**Transformations Applied**:
1. **ID Normalization**: Remove system-specific prefixes
2. **Date Conversion**: Convert integer dates (YYYYMMDD) to DATE type
3. **Categorical Standardization**: Map codes to business terms
4. **NULL Handling**: Replace invalid values with NULL or defaults
5. **Derived Fields**: Calculate end dates, extract category IDs

**Usage**:
```sql
-- Load Silver layer (runs transformations)
EXEC silver.load_silver;
```

### Gold Layer (Business/Presentation)
**Purpose**: Provide business-ready dimensional model for analytics

**Characteristics**:
- Implemented as **Views** (not materialized)
- Star Schema design (Facts & Dimensions)
- Surrogate keys for dimension tables
- Joins Silver tables to create unified views

**Objects**:
- **`dim_customers`**: Customer master with demographics and location
- **`dim_products`**: Product catalog with categories and attributes
- **`fact_sales`**: Sales transactions with foreign keys to dimensions

**Usage**:
```sql
-- Query Gold layer for analytics
SELECT 
    c.first_name,
    c.last_name,
    p.product_name,
    f.sales_amount
FROM gold.fact_sales f
JOIN gold.dim_customers c ON f.customer_key = c.customer_key
JOIN gold.dim_products p ON f.product_key = p.product_key
WHERE c.country = 'Australia';
```

---

## Scope and Future Expansion

### Current Scope

#### What the System Does:
1. **Data Integration**:
   - Ingests data from 2 source systems (CRM, ERP)
   - Handles 6 source tables across 3 data layers
   - Processes customer, product, and sales data

2. **Data Quality**:
   - Automated cleansing and standardization
   - Date validation and format conversion
   - Categorical value normalization
   - Built-in quality checks and tests

3. **Analytics Support**:
   - Star Schema for efficient querying
   - Pre-joined dimension and fact tables
   - Supports customer, product, and sales analysis

4. **ETL Automation**:
   - Stored procedures for each layer
   - Truncate-and-load pattern (full refresh)
   - Performance monitoring (load duration tracking)

#### Current Limitations:
1. **No Incremental Loading**: Full truncate-and-load approach (not suitable for large datasets)
2. **No SCD Implementation**: No historical tracking of dimension changes
3. **Manual Execution**: Stored procedures must be run manually
4. **Single Database**: No distributed architecture
5. **CSV Only**: Limited to file-based sources
6. **No Real-time**: Batch processing only

### Future Enhancements

####  Phase 1: Performance & Scalability
- [ ] **Incremental Loading**: Implement CDC (Change Data Capture) or watermark-based incremental loads
- [ ] **Indexing Strategy**: Add clustered and non-clustered indexes on frequently queried columns
- [ ] **Partitioning**: Implement table partitioning for large fact tables (e.g., by date)
- [ ] **Materialized Views**: Convert Gold views to indexed views for performance

####  Phase 2: Advanced Features
- [ ] **Slowly Changing Dimensions (SCD)**:
  - Implement Type 2 SCD for tracking historical changes
  - Add effective date columns (`valid_from`, `valid_to`)
  - Add `is_current` flag for active records
- [ ] **Data Lineage**: Track data provenance and transformation history
- [ ] **Error Handling**: Implement logging tables for ETL errors and rejections
- [ ] **Audit Trail**: Add audit columns (created_by, modified_by, modified_date)

####  Phase 3: Automation & Orchestration
- [ ] **Job Scheduling**: SQL Server Agent jobs or external orchestrator (e.g., Apache Airflow)
- [ ] **Alerting**: Email notifications for ETL failures or data quality issues
- [ ] **Dynamic ETL**: Metadata-driven framework for adding new sources
- [ ] **CI/CD Pipeline**: Automate deployment of schema changes

####  Phase 4: Expanded Integration
- [ ] **Additional Sources**:
  - Marketing data (campaigns, leads)
  - Financial data (invoices, payments)
  - Web analytics (clickstream, e-commerce)
- [ ] **Real-time Streaming**: Integrate Kafka or Azure Event Hub for real-time data
- [ ] **API Integration**: REST/SOAP APIs for cloud-based sources
- [ ] **Cloud Migration**: Azure SQL Database or Synapse Analytics

####  Phase 5: Advanced Analytics
- [ ] **Aggregation Tables**: Pre-aggregated summary tables for dashboards
- [ ] **Predictive Models**: Integrate ML models for forecasting
- [ ] **Data Quality Dashboards**: Real-time monitoring of data quality metrics
- [ ] **Self-Service BI**: Power BI or Tableau integration with semantic layer

---

## Database Migration to PostgreSQL

### Migration Considerations

If migrating from **SQL Server to PostgreSQL**, the following changes would be required:

#### 1. SQL Dialect Changes

| SQL Server Feature | PostgreSQL Equivalent | Migration Effort |
|-------------------|----------------------|------------------|
| `GETDATE()` | `CURRENT_DATE` or `NOW()` | **Low** - Find and replace |
| `NVARCHAR(n)` | `VARCHAR(n)` | **Low** - Data types compatible |
| `DATETIME2` | `TIMESTAMP` | **Low** - Direct mapping |
| `ISNULL()` | `COALESCE()` | **Low** - Standard SQL function |
| `BULK INSERT` | `COPY` command | **Medium** - Syntax differs |
| `LEAD() OVER()` | `LEAD() OVER()` | **None** - Identical |
| `ROW_NUMBER()` | `ROW_NUMBER()` | **None** - Identical |
| `GETDATE()` in DEFAULT | `CURRENT_TIMESTAMP` | **Low** - DDL changes |

#### 2. Schema and Object Changes

**A. Stored Procedures**
- **SQL Server**: `CREATE OR ALTER PROCEDURE`
- **PostgreSQL**: `CREATE OR REPLACE FUNCTION` with `LANGUAGE plpgsql`
- **Changes Required**:
  ```sql
  -- SQL Server
  CREATE OR ALTER PROCEDURE bronze.load_bronze AS
  BEGIN
    -- logic
  END
  
  -- PostgreSQL
  CREATE OR REPLACE FUNCTION bronze.load_bronze()
  RETURNS void AS $$
  BEGIN
    -- logic
  END;
  $$ LANGUAGE plpgsql;
  ```

**B. Data Loading**
- **SQL Server BULK INSERT**:
  ```sql
  BULK INSERT bronze.crm_cust_info
  FROM 'C:\path\to\file.csv'
  WITH (FIRSTROW = 2, FIELDTERMINATOR = ',', TABLOCK);
  ```
- **PostgreSQL COPY**:
  ```sql
  COPY bronze.crm_cust_info
  FROM '/path/to/file.csv'
  WITH (FORMAT csv, HEADER true, DELIMITER ',');
  ```

**C. Error Handling**
- **SQL Server**: `BEGIN TRY...END TRY BEGIN CATCH...END CATCH`
- **PostgreSQL**: `BEGIN...EXCEPTION WHEN...END`
- **Changes Required**:
  ```sql
  -- PostgreSQL
  BEGIN
    -- logic
  EXCEPTION
    WHEN OTHERS THEN
      RAISE NOTICE 'Error: %', SQLERRM;
  END;
  ```

#### 3. Configuration Files to Modify

**A. Connection Strings** (if using application layer):
- Update database driver: `sqlserver` → `postgresql`
- Change port: `1433` → `5432` (default)
- Update authentication method

**B. Environment Variables**:
```bash
# Before (SQL Server)
DB_HOST=localhost
DB_PORT=1433
DB_NAME=DataWarehouse
DB_DRIVER=mssql

# After (PostgreSQL)
DB_HOST=localhost
DB_PORT=5432
DB_NAME=datawarehouse  # PostgreSQL lowercase convention
DB_DRIVER=postgresql
```

#### 4. Script Modifications Required

**Files to Update**:
1. **`/scripts/init_database.sql`**:
   - Remove `USE master; GO`
   - Replace `CREATE DATABASE` syntax
   - PostgreSQL uses `CREATE SCHEMA` without `GO` statements

2. **`/scripts/bronze/ddl_bronze.sql`**:
   - Change `NVARCHAR` → `VARCHAR`
   - Change `DATETIME2` → `TIMESTAMP`
   - Remove `GO` statements

3. **`/scripts/bronze/proc_load_bronze.sql`**:
   - Convert to PostgreSQL function syntax
   - Replace `BULK INSERT` with `COPY`
   - Update error handling blocks
   - Replace `GETDATE()` with `NOW()`
   - Replace `PRINT` with `RAISE NOTICE`

4. **`/scripts/silver/ddl_silver.sql`**:
   - Same DDL changes as Bronze
   - Update DEFAULT constraints: `DEFAULT GETDATE()` → `DEFAULT CURRENT_TIMESTAMP`

5. **`/scripts/silver/proc_load_silver.sql`**:
   - Convert to PostgreSQL function
   - Replace `GETDATE()` → `NOW()`
   - Update `CAST` for date conversions (may differ slightly)
   - Replace `DATEDIFF(SECOND,...)` → `EXTRACT(EPOCH FROM ...)`

6. **`/scripts/gold/ddl_gold.sql`**:
   - Views syntax is mostly compatible
   - Minor adjustments for window functions if needed

#### 5. Testing and Validation

**Migration Checklist**:
- [ ] Update all DDL scripts (data types, defaults)
- [ ] Convert stored procedures to functions
- [ ] Replace BULK INSERT with COPY commands
- [ ] Update error handling blocks
- [ ] Test all date functions and conversions
- [ ] Validate window functions (LEAD, ROW_NUMBER)
- [ ] Run quality check scripts
- [ ] Performance test with sample data
- [ ] Update documentation for PostgreSQL-specific notes

**Estimated Migration Effort**: **Medium (2-3 days)**
- Most SQL is standard and portable
- Main effort in procedure conversion and BULK INSERT replacement
- Window functions and CTEs are identical

---

## Setup Instructions

### Prerequisites
- **Microsoft SQL Server** (2016 or later)
  - Express Edition (free) or higher
  - SQL Server Management Studio (SSMS) recommended
- **Windows Environment** (for BULK INSERT file paths)
- **Sufficient Disk Space**: ~100 MB for sample data

### Installation Steps

#### 1. Clone the Repository
```bash
git clone https://github.com/MonalBarse/Insight-Engine.git
cd Insight-Engine
```

#### 2. Prepare Data Files
Ensure CSV files are accessible to SQL Server:
```bash
# Default path in scripts: C:\sql\dwh_project\datasets\
# Copy datasets folder to this location or update paths in proc_load_bronze.sql
```

**Update File Paths** (if different):
Edit `/scripts/bronze/proc_load_bronze.sql` and update:
```sql
-- Change these paths to match your environment
FROM 'C:\sql\dwh_project\datasets\source_crm\cust_info.csv'
-- to your actual path, e.g.:
FROM 'C:\Users\YourName\Insight-Engine\datasets\source_crm\cust_info.csv'
```

#### 3. Create Database and Schemas
Run the initialization script:
```sql
-- Connect to SQL Server using SSMS
-- Execute: scripts/init_database.sql
-- This creates the 'DataWarehouse' database and bronze/silver/gold schemas
```

#### 4. Create Bronze Layer
```sql
-- Execute: scripts/bronze/ddl_bronze.sql
-- Creates Bronze tables for raw data ingestion
```

#### 5. Load Bronze Layer
```sql
-- Execute: scripts/bronze/proc_load_bronze.sql (creates procedure)
-- Then run:
EXEC bronze.load_bronze;
-- Loads data from CSV files into Bronze tables
```

#### 6. Create Silver Layer
```sql
-- Execute: scripts/silver/ddl_silver.sql
-- Creates Silver tables for cleansed data
```

#### 7. Load Silver Layer
```sql
-- Execute: scripts/silver/proc_load_silver.sql (creates procedure)
-- Then run:
EXEC silver.load_silver;
-- Transforms and loads data from Bronze to Silver
```

#### 8. Create Gold Layer
```sql
-- Execute: scripts/gold/ddl_gold.sql
-- Creates Gold layer views (Star Schema)
```

#### 9. Verify Data Quality
```sql
-- Execute: tests/quality_checks_silver.sql
-- Execute: tests/quality_checks_gold.sql
-- All quality checks should return 0 rows (no issues)
```

### Environment Variables (Optional)

If integrating with an application:
```bash
# Database connection
DB_SERVER=localhost
DB_NAME=DataWarehouse
DB_USER=your_username
DB_PASSWORD=your_password

# File paths
DATA_SOURCE_PATH=C:\sql\dwh_project\datasets
```

### Execution Order Summary
```
1. init_database.sql       → Create database & schemas
2. bronze/ddl_bronze.sql   → Create Bronze tables
3. bronze/proc_load_bronze.sql → Create Bronze load procedure
4. EXEC bronze.load_bronze → Load Bronze data
5. silver/ddl_silver.sql   → Create Silver tables
6. silver/proc_load_silver.sql → Create Silver load procedure
7. EXEC silver.load_silver → Load Silver data
8. gold/ddl_gold.sql       → Create Gold views
9. Run quality checks      → Validate data
```

---

## Quality Assurance

### Data Quality Checks

The system includes comprehensive quality validation:

#### Silver Layer Checks (`tests/quality_checks_silver.sql`)
-  **Primary Key Integrity**: No NULL or duplicate keys
-  **Data Cleansing**: No unwanted spaces in string fields
-  **Standardization**: Categorical values are normalized
-  **Date Validation**: No future dates or invalid date ranges
-  **Business Rules**: Sales = Quantity × Price consistency

#### Gold Layer Checks (`tests/quality_checks_gold.sql`)
- **Surrogate Key Uniqueness**: Customer and product keys are unique
- **Referential Integrity**: All fact records link to valid dimensions
- **Data Model Connectivity**: Joins between fact and dimensions succeed

### Running Quality Checks
```sql
-- After loading Silver layer
USE DataWarehouse;
GO
EXEC master.dbo.sp_executesql @statement = N'
  -- Run all checks from quality_checks_silver.sql
';

-- After creating Gold layer
EXEC master.dbo.sp_executesql @statement = N'
  -- Run all checks from quality_checks_gold.sql
';
```

**Expected Result**: All quality check queries should return **0 rows**, indicating no data quality issues.

---

## Contributing

### For Developers

#### Adding a New Data Source
1. **Bronze Layer**:
   - Add table DDL in `scripts/bronze/ddl_bronze.sql`
   - Update `bronze.load_bronze` procedure to include BULK INSERT
2. **Silver Layer**:
   - Add transformation logic in `scripts/silver/proc_load_silver.sql`
   - Define cleansing rules and business logic
3. **Gold Layer**:
   - Update dimension/fact views in `scripts/gold/ddl_gold.sql`
   - Ensure proper joins and surrogate keys
4. **Testing**:
   - Add quality checks in `tests/` directory
   - Validate data integrity

#### Code Standards
- Follow naming conventions in `docs/naming_conventions.md`
- Use snake_case for all database objects
- Document all stored procedures with header comments
- Add DDL comments for complex business logic

#### Pull Request Process
1. Create a feature branch: `git checkout -b feature/new-source`
2. Make changes and test thoroughly
3. Update documentation (README, data catalog)
4. Submit PR with description of changes
5. Ensure all quality checks pass

### Documentation
- **Data Catalog**: Update `docs/data_catalog.md` for new Gold layer objects
- **Diagrams**: Refresh architecture diagrams if structure changes
- **README**: Keep this file current with new features

---

## License

This project is for educational and demonstration purposes.

---

**Project Status**: Active Development

**Last Updated**: 2025

**Version**: 1.0
