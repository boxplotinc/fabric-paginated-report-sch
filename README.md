# Fabric Paginated Report Batch Executor

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![GitHub Stars](https://img.shields.io/github/stars/boxplotinc/fabric-paginated-report-sch?style=social)](https://github.com/boxplotinc/fabric-paginated-report-sch/stargazers)
[![GitHub Last Commit](https://img.shields.io/github/last-commit/boxplotinc/fabric-paginated-report-sch)](https://github.com/boxplotinc/fabric-paginated-report-sch/commits)

A production-ready framework for **Microsoft Fabric** and **Power BI** paginated report automation. This Python-based batch executor enables automated generation of paginated reports with dynamic parameters from multiple data sources including **Lakehouse**, **Semantic Models**, **Warehouses**, and **JSON arrays**. Built for enterprise **data engineering** workflows with features like **retry logic**, **OneLake storage integration**, and **Azure pipeline orchestration** via REST API.

## Features

✅ **Four Flexible Parameter Sources**
- **Semantic Model** (Power BI Dataset) - Query data behind reports with DAX
- **Lakehouse** (Delta Lake) - Native Spark SQL integration
- **JSON Array** - Simple, direct input for testing
- **Warehouse** (SQL Server) - T-SQL queries for enterprise scenarios

✅ **Output Options**
- OneLake storage with date-based folder hierarchy
- Multiple export formats (PDF, Excel, Word, PowerPoint, PNG)
- Automatic file archival and organization

✅ **Enterprise-Ready**
- Retry logic with exponential backoff for failed reports
- Continue on failure (processes all parameters even if some fail)
- Managed identity authentication
- Configurable timeouts and retry intervals
- Pipeline integration with detailed logging

✅ **User-Friendly**
- No code changes needed to update parameters
- Business users can maintain parameter lists
- Comprehensive logging with multiple levels
- Detailed error messages with context

---

## Table of Contents

1. [Quick Start](#quick-start)
2. [Architecture](#architecture)
3. [Parameter Sources](#parameter-sources)
4. [Setup Instructions](#setup-instructions)
5. [Usage Examples](#usage-examples)
6. [Pipeline Configuration](#pipeline-configuration)
7. [Advanced Features](#advanced-features)
8. [Troubleshooting](#troubleshooting)
9. [Best Practices](#best-practices)

---

## Quick Start

### 1. Upload the Notebook

1. Open your Fabric workspace
2. Create a new notebook
3. Import `paginated_report_batch_executor.ipynb`
4. Attach to a Lakehouse (if using Lakehouse parameter source)

### 2. Choose Your Parameter Source

Select one of the four parameter sources based on your needs:

| Source | Best For | Setup Effort |
|--------|----------|--------------|
| **Semantic Model** | Report-driven parameters, business users | Low |
| **Lakehouse** | Large lists, data engineers | Low |
| **JSON** | Testing, static lists | Minimal |
| **Warehouse** | Enterprise, complex SQL | Medium |

### 3. Run a Test

1. Create a pipeline using `pipeline/pipeline_definition.json`
2. Configure parameters (see [Usage Examples](#usage-examples))
3. Run manually to test
4. Verify files in OneLake (Files/reports/archive folder)
5. Enable scheduled trigger once working

---

## Architecture

```
Pipeline (Scheduled/Manual Trigger)
  │
  ├─ Parameters: report_id, static_params, report_partitioning_source, etc.
  │
  ▼
Fabric Notebook: Report Batch Executor
  │
  ├─ Cell 1-2: Parameters & Imports
  │
  ├─ Cell 3: Input Validation & ParameterLoader
  │   ├─ InputValidator class
  │   └─ ParameterLoader class (supports 4 sources)
  │
  ├─ Cell 4: Token Manager & Validation
  │   ├─ TokenManager class (automatic refresh)
  │   └─ Parse & validate all parameters
  │
  ├─ Cell 5: Power BI API Functions
  │   ├─ handle_api_response()
  │   ├─ initiate_report_export()
  │   ├─ poll_export_status()
  │   └─ download_report_file()
  │
  ├─ Cell 6: File Storage Functions
  │   ├─ sanitize_filename()
  │   ├─ generate_file_name()
  │   └─ save_to_onelake()
  │
  ├─ Cell 7: Main Execution Function
  │   └─ execute_report_with_retry()
  │
  ├─ Cell 8: Load Special Parameter Values
  │   └─ ParameterLoader.load()
  │       ├─ Semantic Model (DAX query via sempy)
  │       ├─ Lakehouse (Spark SQL)
  │       ├─ JSON (Direct parsing)
  │       └─ Warehouse (pyodbc + T-SQL)
  │
  ├─ Cell 9: Main Execution Loop
  │   └─ FOR EACH partitioning parameter value:
  │       ├─ Merge static + special params
  │       ├─ Execute report (Power BI API)
  │       ├─ Poll & download
  │       ├─ Save to OneLake
  │       └─ Retry 3x on failure
  │
  └─ Cell 10: Results & Exit
      └─ Return summary to pipeline
```

### Execution Flow

1. **Parameter Loading**: Load partitioning parameter values from configured source
2. **Parameter Merging**: Combine static parameters with each special value
3. **Report Generation**: Execute paginated report via Power BI REST API
4. **File Storage**: Save to OneLake with date-based folder structure
5. **Error Handling**: Retry failures with exponential backoff, continue to next parameter
6. **Summary**: Report total successes, failures, and file locations

---

## Parameter Sources

### 1. Semantic Model (Recommended for Business Users)

Query data from Power BI semantic models using DAX.

**Advantages:**
- Uses trusted data from existing reports
- Row-Level Security automatically enforced
- Includes business logic and calculations
- Fast with semantic layer caching

**Configuration:**
```json
{
  "report_partitioning_source": "semantic_model",
  "semantic_model_workspace_id": "workspace-guid",
  "semantic_model_dataset_id": "dataset-guid",
  "semantic_model_dax_query": "EVALUATE FILTER(DISTINCT('DimCustomer'[CustomerName]), 'DimCustomer'[IsActive] = TRUE)"
}
```

**Example DAX Queries:**
```dax
-- Get all active customers
EVALUATE FILTER(
    DISTINCT('DimCustomer'[CustomerName]),
    'DimCustomer'[IsActive] = TRUE
)

-- Get top 10 customers by sales
EVALUATE TOPN(
    10,
    SUMMARIZECOLUMNS(
        'DimCustomer'[CustomerName],
        "TotalSales", SUM('FactSales'[SalesAmount])
    ),
    [TotalSales], DESC
)

-- Get customers with sales in last 12 months
EVALUATE CALCULATETABLE(
    DISTINCT('DimCustomer'[CustomerName]),
    DATESINPERIOD('DimDate'[Date], TODAY(), -12, MONTH)
)
```

See `setup/sample_semantic_model_queries.dax` for more examples.

---

### 2. Lakehouse (Recommended for Data Engineers)

Query Delta tables in Lakehouse using Spark SQL.

**Advantages:**
- Native Spark SQL (no connection strings needed)
- Best performance for large lists (1000+)
- Easy maintenance via Lakehouse UI
- ACID compliance and version control
- Zero authentication overhead

**Setup:**
```sql
-- Run in Lakehouse notebook
CREATE TABLE parameter_config (
    Category STRING,
    ParameterValue STRING,
    IsActive BOOLEAN,
    SortOrder INT
) USING DELTA;

-- Insert sample data
INSERT INTO parameter_config VALUES
    ('MonthlyReportCustomers', 'Acme Corp', true, 1),
    ('MonthlyReportCustomers', 'TechStart Inc', true, 2),
    ('MonthlyReportCustomers', 'Global Solutions', true, 3);
```

**Configuration:**
```json
{
  "report_partitioning_source": "lakehouse",
  "lakehouse_table": "parameter_config",
  "lakehouse_category": "MonthlyReportCustomers",
  "lakehouse_column": "ParameterValue",
  "lakehouse_filter": ""
}
```

See `setup/create_lakehouse_parameter_table.sql` for complete schema.

---

### 3. JSON Array (Recommended for Testing)

Provide parameters directly as JSON array.

**Advantages:**
- Simplest setup (no infrastructure)
- Perfect for testing
- Good for static, small lists
- Version control friendly

**Configuration:**
```json
{
  "report_partitioning_source": "json",
  "report_partitioning_values": "[\"Acme Corp\", \"TechStart Inc\", \"Global Solutions\"]"
}
```

**Use Cases:**
- Testing and development
- Proof of concept
- One-time or ad-hoc reports
- Very small lists (< 10 items)

---

### 4. Warehouse (Recommended for Enterprise)

Query Warehouse tables using T-SQL.

**Advantages:**
- Familiar T-SQL syntax
- Complex SQL logic (joins, CTEs, stored procedures)
- Row-Level Security (RLS)
- Column-Level Security (CLS)
- Integration with SQL Server workflows

**Setup:**
```sql
-- Run in Warehouse
CREATE TABLE dbo.ParameterConfig (
    Category NVARCHAR(100),
    ParameterValue NVARCHAR(500),
    IsActive BIT,
    SortOrder INT
);

-- Insert data
INSERT INTO dbo.ParameterConfig VALUES
    ('MonthlyReportCustomers', 'Acme Corp', 1, 1),
    ('MonthlyReportCustomers', 'TechStart Inc', 1, 2);
```

**Configuration:**
```json
{
  "report_partitioning_source": "warehouse",
  "warehouse_name": "EnterpriseWarehouse",
  "warehouse_table": "dbo.ParameterConfig",
  "warehouse_column": "ParameterValue",
  "warehouse_category": "MonthlyReportCustomers"
}
```

---

## Setup Instructions

### Prerequisites

1. Microsoft Fabric workspace
2. Paginated report published to workspace
3. Lakehouse for OneLake storage (automatically created with workspace)

### Step 1: Import Notebook

1. Navigate to your Fabric workspace
2. Click **New** → **Import notebook**
3. Select `paginated_report_batch_executor.ipynb`
4. If using Lakehouse source, attach notebook to Lakehouse

### Step 2: Set Up Parameter Source

Choose one of the four options:

#### Option A: Semantic Model
1. Identify Power BI semantic model with parameter data
2. Note workspace GUID and dataset GUID
3. Write DAX query (see `setup/sample_semantic_model_queries.dax`)
4. Test query in Power BI Desktop

#### Option B: Lakehouse
1. Attach notebook to Lakehouse
2. Run `setup/create_lakehouse_parameter_table.sql`
3. Run `setup/sample_data.sql` for test data
4. Verify: `SELECT * FROM parameter_config`

#### Option C: JSON
1. Create JSON array with values
2. Validate JSON syntax
3. Ready to use!

#### Option D: Warehouse
1. Create Warehouse in workspace
2. Run table creation SQL (see config examples)
3. Insert parameter data
4. Grant read permissions

### Step 3: Create Pipeline

1. Go to **Pipelines** in Fabric
2. Create new pipeline
3. Add **Notebook activity**
4. Select `paginated_report_batch_executor` notebook
5. Configure parameters (see [Usage Examples](#usage-examples))
6. Add triggers if needed (Daily, Weekly, Monthly)

### Step 4: Test

1. Run pipeline manually
2. Monitor notebook execution
3. Verify files in OneLake (Files/reports/archive folder)
4. Check execution logs for any errors
5. Enable scheduled trigger once working

---

## Usage Examples

### Example 1: Monthly Customer Reports (Semantic Model)

**Scenario:** Generate monthly sales report for each active customer using data from Power BI semantic model.

**Pipeline Parameters:**
```json
{
  "report_id": "12345678-1234-1234-1234-123456789abc",
  "workspace_id": "workspace-guid",
  "output_format": "PDF",
  "static_params": "{\"start_date\": \"2024-01-01\", \"end_date\": \"2024-12-31\"}",
  "report_partitioning_column": "Customer",
  "report_partitioning_source": "semantic_model",
  "semantic_model_workspace_id": "workspace-guid",
  "semantic_model_dataset_id": "dataset-guid",
  "semantic_model_dax_query": "EVALUATE FILTER(DISTINCT('DimCustomer'[CustomerName]), 'DimCustomer'[IsActive] = TRUE)",
  "archive_to_onelake": "true",
  "max_retries": "3"
}
```

**Result:** PDF reports saved to OneLake at `Files/reports/archive/2025/01/30/`, one per customer.

---

### Example 2: Regional Reports (Lakehouse)

**Scenario:** Generate quarterly report for each region using parameter list from Lakehouse.

**Setup Lakehouse:**
```sql
INSERT INTO parameter_config (Category, ParameterValue, IsActive, SortOrder)
VALUES
    ('QuarterlyRegions', 'North America', true, 1),
    ('QuarterlyRegions', 'Europe', true, 2),
    ('QuarterlyRegions', 'Asia Pacific', true, 3),
    ('QuarterlyRegions', 'Latin America', true, 4);
```

**Pipeline Parameters:**
```json
{
  "report_id": "report-guid",
  "workspace_id": "workspace-guid",
  "output_format": "XLSX",
  "static_params": "{\"quarter\": \"Q1\", \"year\": \"2024\"}",
  "report_partitioning_column": "Region",
  "report_partitioning_source": "lakehouse",
  "lakehouse_table": "parameter_config",
  "lakehouse_category": "QuarterlyRegions",
  "lakehouse_column": "ParameterValue"
}
```

**Result:** Excel reports for 4 regions saved to OneLake.

---

### Example 3: Testing with JSON

**Scenario:** Test report generation for 3 specific customers before rolling out to all.

**Pipeline Parameters:**
```json
{
  "report_id": "report-guid",
  "workspace_id": "workspace-guid",
  "output_format": "PDF",
  "static_params": "{\"start_date\": \"2024-01-01\", \"end_date\": \"2024-01-31\"}",
  "report_partitioning_column": "Customer",
  "report_partitioning_source": "json",
  "report_partitioning_values": "[\"Test Customer A\", \"Test Customer B\", \"Test Customer C\"]"
}
```

**Result:** 3 test reports generated quickly without setting up database tables, saved to OneLake.

---

### Example 4: Enterprise Warehouse with RLS

**Scenario:** Generate reports for customers visible to current user based on Row-Level Security.

**Warehouse Setup:**
```sql
-- Create table with RLS
CREATE TABLE dbo.CustomerReporting (
    CustomerName NVARCHAR(200),
    AssignedTo NVARCHAR(100),
    IsActive BIT
);

-- Create RLS policy
CREATE FUNCTION dbo.fn_CustomerSecurityPredicate(@AssignedTo NVARCHAR(100))
RETURNS TABLE WITH SCHEMABINDING AS
RETURN SELECT 1 AS Result
WHERE @AssignedTo = USER_NAME() OR USER_NAME() IN (SELECT UserName FROM dbo.Admins);

CREATE SECURITY POLICY dbo.CustomerReportingPolicy
ADD FILTER PREDICATE dbo.fn_CustomerSecurityPredicate(AssignedTo)
ON dbo.CustomerReporting WITH (STATE = ON);
```

**Pipeline Parameters:**
```json
{
  "report_partitioning_source": "warehouse",
  "warehouse_name": "EnterpriseWarehouse",
  "warehouse_table": "dbo.CustomerReporting",
  "warehouse_column": "CustomerName",
  "warehouse_category": ""
}
```

**Result:** Each user only generates reports for their assigned customers (RLS enforced).

---

## Pipeline Configuration

### Required Parameters

| Parameter | Description | Example |
|-----------|-------------|---------|
| `workspace_id` | Fabric workspace GUID | `"12345678-1234-..."` |
| `report_id` | Paginated report GUID | `"87654321-4321-..."` |
| `output_format` | Export format | `"PDF"`, `"XLSX"`, `"DOCX"` |
| `static_params` | Fixed parameters (JSON) | `"{\"start_date\": \"2024-01-01\"}"` |
| `report_partitioning_column` | Parameter to loop through | `"Customer"` |
| `report_partitioning_source` | Parameter source type | `"semantic_model"`, `"lakehouse"`, `"json"`, `"warehouse"` |

### Source-Specific Parameters

**Semantic Model:**
- `semantic_model_workspace_id`
- `semantic_model_dataset_id`
- `semantic_model_dax_query`

**Lakehouse:**
- `lakehouse_table`
- `lakehouse_category`
- `lakehouse_column`
- `lakehouse_filter` (optional)

**JSON:**
- `report_partitioning_values`

**Warehouse:**
- `warehouse_name`
- `warehouse_table`
- `warehouse_column`
- `warehouse_category`

### Optional Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| `archive_to_onelake` | Save to OneLake | `"true"` |
| `max_retries` | Retry attempts per report | `"3"` |
| `export_timeout_seconds` | Max seconds to wait for export | `"600"` |
| `poll_interval_seconds` | Seconds between status polls | `"5"` |

---

## Advanced Features

### 1. Multiple Parameter Loops

For complex scenarios with multiple loop parameters:

**Option A: Use Semantic Model with SUMMARIZECOLUMNS**
```dax
EVALUATE
SUMMARIZECOLUMNS(
    'DimGeography'[Region],
    'DimProduct'[Category]
)
```

This returns all combinations of Region × Category. The notebook will loop through each row.

**Option B: Pre-compute combinations in Lakehouse**
```sql
INSERT INTO parameter_config (Category, ParameterValue, IsActive, SortOrder)
SELECT
    'RegionCategoryCombo',
    CONCAT(Region, '|', Category),
    true,
    ROW_NUMBER() OVER (ORDER BY Region, Category)
FROM region_category_combinations;
```

Then parse the combined value in your report.

---

### 2. Conditional Parameter Lists

Filter parameters based on date, status, or other criteria:

**Semantic Model (DAX):**
```dax
EVALUATE
FILTER(
    DISTINCT('DimCustomer'[CustomerName]),
    'DimCustomer'[IsActive] = TRUE
    && CALCULATE(SUM('FactSales'[Amount]), DATESINPERIOD('DimDate'[Date], TODAY(), -6, MONTH)) > 10000
)
```

**Lakehouse (SQL):**
```json
{
  "lakehouse_filter": "ValidFrom <= CURRENT_TIMESTAMP() AND (ValidTo IS NULL OR ValidTo >= CURRENT_TIMESTAMP())"
}
```

---

### 3. Dynamic File Naming

Files are automatically named with this pattern:
```
Report_{SpecialParamName}_{SanitizedValue}_{Timestamp}.{format}
```

Example:
```
Report_Customer_AcmeCorp_20250130_143022.pdf
```

Special characters are sanitized for filesystem compatibility.

---

### 4. Error Handling and Retry

**Retry Logic:**
- 3 retry attempts per report (configurable)
- Exponential backoff: 30s, 60s, 120s
- Continues to next parameter on failure

**Error Categories:**
- Power BI API errors (authentication, report not found, etc.)
- Export failures (timeout, invalid parameters, etc.)
- File storage errors (OneLake write failures, etc.)

All errors are logged with detailed messages.

---

### 5. Monitoring and Notifications

**Console Logging:**
Real-time output shows:
- Parameter loading progress
- Each report execution (5 steps per report)
- Success/failure status
- File paths and sizes
- Execution summary

**Pipeline Notifications:**
Configure webhook URL for notifications:
- Success: Includes file count, total size, paths
- Failure: Includes error details

**Example Webhook Payload (Success):**
```json
{
  "status": "success",
  "pipelineRunId": "run-guid",
  "successCount": 5,
  "failCount": 0,
  "totalSize": "12.5 MB",
  "files": [
    "/reports/2025/01/30/Report_Customer_AcmeCorp_20250130_143022.pdf",
    "/reports/2025/01/30/Report_Customer_TechStart_20250130_143035.pdf"
  ]
}
```

---

### 6. Performance Tuning Parameters

Fine-tune the notebook's performance and behavior for specific scenarios:

| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| `download_chunk_size_mb` | 1 | 1-100 | Download chunk size in MB for large files |
| `file_size_warning_mb` | 500 | 10-5000 | File size threshold for warnings |
| `connection_timeout_seconds` | 30 | 5-300 | API connection timeout |
| `download_timeout_seconds` | 120 | 30-600 | File download timeout |
| `param_loader_retry_attempts` | 3 | 1-10 | Parameter loading retry count |
| `param_loader_retry_delay_seconds` | 5 | 1-60 | Delay between parameter loading retries |
| `token_refresh_interval_minutes` | 45 | 5-55 | Auto token refresh interval |

**When to Adjust:**

**Large Files (Reports > 100MB):**
```json
{
  "download_chunk_size_mb": "5",
  "file_size_warning_mb": "1000",
  "download_timeout_seconds": "300"
}
```

**Slow Network Connections:**
```json
{
  "connection_timeout_seconds": "60",
  "download_timeout_seconds": "240"
}
```

**Long-Running Batches (1000+ reports):**
```json
{
  "token_refresh_interval_minutes": "30"
}
```
*Note: Tokens expire after 60 minutes. Refresh interval should be less than 55 minutes.*

**Unreliable Parameter Source:**
```json
{
  "param_loader_retry_attempts": "5",
  "param_loader_retry_delay_seconds": "10"
}
```

**Token Refresh Mechanism:**
- Tokens are automatically refreshed based on `token_refresh_interval_minutes`
- Prevents authentication failures during multi-hour batch executions
- No user intervention required
- Logged with timestamp for audit purposes

---

## Troubleshooting

### Common Issues

#### 1. No Parameter Values Loaded

**Symptoms:** "No parameter values loaded! Check your configuration."

**Solutions:**
- **Semantic Model:** Verify DAX query returns data in Power BI Desktop
- **Lakehouse:** Check table exists and Category matches exactly (case-sensitive)
- **JSON:** Validate JSON syntax at jsonlint.com
- **Warehouse:** Verify table exists and connection works

---

#### 2. Authentication Errors

**Symptoms:** 401 Unauthorized, token errors

**Solutions:**
- Verify managed identity is enabled for workspace
- Check service principal has access to:
  - Report workspace (Contributor or higher)
  - Semantic model (Read permission)
  - Lakehouse (Read permission)
  - Warehouse (SELECT permission)
- Refresh Azure AD token if expired

---

#### 3. Report Export Timeout

**Symptoms:** "Export timeout after 600 seconds"

**Solutions:**
- Simplify report (reduce data, remove complex visuals)
- Increase timeout in notebook (currently 10 minutes)
- Check report parameters are valid
- Verify underlying data sources are accessible

---

#### 4. Slow Performance

**Symptoms:** Reports take very long to generate

**Solutions:**
- **Semantic Model:** Ensure query is optimized, use DISTINCT instead of VALUES
- **Lakehouse:** Add indexes on Category and IsActive columns
- **Report:** Optimize report design, reduce data volume
- **Parallel Processing:** Consider splitting large batches across multiple pipelines

---

### Debug Mode

To run notebook interactively for debugging:

1. Open notebook in Fabric
2. Set default values in Cell 1 for all parameters
3. Run cells one by one
4. Check output of each cell
5. Fix issues and re-run

---

## Best Practices

### 1. Parameter Source Selection

| Scenario | Recommended Source |
|----------|-------------------|
| Business users maintain list | Semantic Model or Lakehouse |
| Need Row-Level Security | Semantic Model or Warehouse |
| Large lists (1000+) | Lakehouse |
| Testing/Development | JSON |
| Complex SQL logic | Warehouse |
| Highest performance | Lakehouse |

---

### 2. File Organization

**OneLake Structure:**
```
Files/
  /reports/
    /archive/
      /{YYYY}/{MM}/{DD}/
        Report_Customer_Value_Timestamp.pdf
```

Example:
```
Files/reports/archive/2025/01/30/Report_Customer_AcmeCorp_20250130_143022.pdf
```

---

### 3. Scheduling

- **Daily Reports:** Run at off-peak hours (e.g., 6:00 AM)
- **Weekly Reports:** Run Monday morning
- **Monthly Reports:** Run on 1st or last day of month
- **Avoid:** Running multiple large batches simultaneously

---

### 4. Security

- ✅ Always use managed identity for authentication
- ✅ Grant minimal required permissions to workspaces and data sources
- ✅ Use Row-Level Security (RLS) when available in semantic models
- ✅ Audit parameter access and report generation
- ✅ Validate and sanitize all input parameters
- ✅ Use proper error handling to avoid exposing sensitive information
- ❌ Never hardcode credentials or connection strings
- ❌ Never commit secrets or GUIDs to version control

---

### 5. Maintenance

- Regularly review and clean up inactive parameters
- Monitor pipeline success rates and execution times
- Archive or delete old reports from OneLake to manage storage
- Update DAX queries when semantic model schema changes
- Test after any Fabric platform updates
- Document parameter meanings and categories for business users
- Review and optimize timeout and retry settings based on actual performance

---

## Support and Contributing

### Getting Help

1. Check [Troubleshooting](#troubleshooting) section
2. Review example configurations in `config/` folder
3. Check Fabric platform status
4. Review notebook execution logs
5. Contact your Fabric admin or support team

### File Structure

```
/
├── paginated_report_batch_executor.ipynb  # Main notebook
├── README.md                               # This file
├── setup/
│   ├── create_lakehouse_parameter_table.sql
│   ├── sample_data.sql
│   └── sample_semantic_model_queries.dax
├── config/
│   ├── example_semantic_model.json
│   ├── example_lakehouse_mode.json
│   ├── example_json_mode.json
│   └── example_warehouse_mode.json
└── pipeline/
    └── pipeline_definition.json
```

---

## Version History

**v1.0** (2025-11-03)
- Production-ready paginated report batch execution framework
- Four flexible parameter sources: Semantic Model (DAX), Lakehouse (Spark SQL), JSON, Warehouse (T-SQL)
- Automatic token refresh for long-running batches (handles 1-hour token expiration)
- OneLake archival with date-based folder hierarchy
- Retry logic with exponential backoff for robust error handling
- Configurable performance tuning parameters for enterprise scenarios
- Comprehensive input validation and SQL injection protection
- Pipeline integration with detailed logging and progress tracking
- Continue-on-failure support for large batch operations
- Managed identity authentication

---

## License

This project is provided as-is for use within Microsoft Fabric environments.

---

## Acknowledgments

- Generated by Claude Code
- Built for Microsoft Fabric
- Uses Power BI REST API
- Leverages semantic-link library for Semantic Model integration

---

## Quick Reference

### Notebook Parameters

```python
# Report configuration
workspace_id = ""
report_id = ""
output_format = "PDF"
static_params = "{}"
report_partitioning_column = "Customer"

# Source configuration
report_partitioning_source = "semantic_model"  # or "lakehouse", "json", "warehouse"

# Semantic Model
semantic_model_workspace_id = ""
semantic_model_dataset_id = ""
semantic_model_dax_query = "EVALUATE DISTINCT('Table'[Column])"

# Lakehouse
lakehouse_table = "parameter_config"
lakehouse_category = "CustomerList"
lakehouse_column = "ParameterValue"
lakehouse_filter = ""

# JSON
report_partitioning_values = "[]"

# Warehouse
warehouse_name = ""
warehouse_table = "dbo.ParameterConfig"
warehouse_column = "ParameterValue"
warehouse_category = ""

# Options
archive_to_onelake = "true"
max_retries = "3"
export_timeout_seconds = "600"
poll_interval_seconds = "5"
```

### Pipeline Triggers

```json
{
  "DailySchedule": "6:00 AM every day",
  "WeeklySchedule": "8:00 AM every Monday",
  "MonthlySchedule": "7:00 AM on 1st of month"
}
```

---

**Ready to generate thousands of reports with ease!** 🚀
