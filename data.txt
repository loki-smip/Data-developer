# Executive Summary  
Cloud data platforms offer diverse tools for ingestion, transformation, orchestration, storage, and analytics. We explore Azure Data Factory (ADF), Azure Databricks (PySpark/Spark SQL), Azure Data Lake Storage (ADLS), Azure SQL (and SQL Server), Logic Apps, Key Vault, Azure Synapse Analytics, and Microsoft Fabric. For each, we present a use case, architecture (Mermaid diagram), code examples (Python, SQL, CLI, JSON), configuration notes (linked services, cluster settings, secrets), best practices/pitfalls, sample data/outputs, and references. We also compare Databricks vs. Synapse vs. Fabric and illustrate an end-to-end pipeline (ADF → Databricks → ADLS → Azure SQL) orchestrated by Logic Apps. All examples use current Azure best practices and official documentation.

## Azure Data Factory (ADF)  
**Use Case:** Cloud ETL orchestration for moving and transforming data between diverse sources (on-premises and cloud). ADF pipelines can use **Copy** activities, **Mapping Data Flows** (for Spark-based transformations), and **Databricks/Synapse activities**. It’s ideal for scheduled, parameterized data pipelines【14†L54-L58】【25†L520-L528】.

```mermaid
flowchart LR
  SourceSQL[(On-Prem SQL Server)] -->|Copy Activity| Blob[Azure Blob/ADLS Storage]
  SourceSQL --> ADF[Azure Data Factory]
  ADF --> Databricks[Azure Databricks / Spark]
  Databricks --> ADLS[(ADLS Gen2)]
  ADLS --> Synapse[Azure Synapse / Warehouses]
  Synapse --> PowerBI[Power BI]
```

**Sample Pipeline JSON (Copy SQL→Blob):**  
```json
{
  "name": "CopySQLToBlobPipeline",
  "properties": {
    "activities": [
      {
        "name": "CopySQLToBlob",
        "type": "Copy",
        "inputs": [ { "referenceName": "CustomerTableDataset", "type": "DatasetReference" } ],
        "outputs": [ { "referenceName": "BlobOutputDataset", "type": "DatasetReference" } ],
        "typeProperties": {
          "source": { "type": "AzureSqlSource" },
          "sink":   { "type": "BlobSink" }
        }
      }
    ]
  }
}
```  
This **Copy** activity reads from Azure SQL and writes to Blob Storage【25†L431-L439】【25†L450-L458】.

**Code (Python SDK):** Use the Azure SDK to create ADF resources. For example, to create a copy-pipeline:  
```python
from azure.identity import ClientSecretCredential
from azure.mgmt.datafactory import DataFactoryManagementClient
from azure.mgmt.datafactory.models import *
# (Initialize credentials and clients omitted for brevity)
# Create linked storage service
storage_string = SecureString(value='DefaultEndpointsProtocol=https;AccountName=acct;AccountKey=<key>')
ls_azure_storage = LinkedServiceResource(properties=AzureStorageLinkedService(connection_string=storage_string))
adf_client.linked_services.create_or_update(rg, df, 'AzureStorageLS', ls_azure_storage)
# Create datasets and copy activity
ds_in = DatasetResource(properties=AzureSqlTableDataset(linked_service_name=LinkedServiceReference('AzureSqlLS'), table_name='Customers'))
ds_out = DatasetResource(properties=AzureBlobDataset(linked_service_name=LinkedServiceReference('AzureStorageLS'), folder_path='output', file_name='customers.csv', format=DelimitedTextFormat(column_delimiter=',')))
adf_client.datasets.create_or_update(rg, df, 'SQLDataset', ds_in)
adf_client.datasets.create_or_update(rg, df, 'BlobDataset', ds_out)
copy_act = CopyActivity(name='CopySQLToBlob', inputs=[DatasetReference('SQLDataset')], outputs=[DatasetReference('BlobDataset')],
                        source=AzureSqlSource(), sink=BlobSink())
pipeline = PipelineResource(activities=[copy_act])
adf_client.pipelines.create_or_update(rg, df, 'CopyPipeline', pipeline)
```
*(Based on Azure SDK examples.)*  

**Configuration:** ADF pipelines use **Linked Services** to connect to data sources. For SQL Server or Azure SQL, use `AzureSqlDatabase` linked service; for ADLS/Blob, use `AzureBlobStorage` or `AzureDataLakeStorageGen2`. Store sensitive credentials in Azure **Key Vault** and reference secrets (see below). Choose an **Integration Runtime**: Azure IR for cloud data or Self-Hosted IR for on-prem sources【14†L65-L73】【11†L60-L68】. Enable **managed identity** for secure access. Example linked service JSON (Azure Key Vault reference):
```json
{
  "name": "AzureSqlLS",
  "properties": {
    "type": "AzureSqlDatabase",
    "typeProperties": {
      "connectionString": "Server=tcp:myserver.database.windows.net;Database=mydb;User ID=azureuser;Password=<password>;"
    }
  }
}
```
Or with Key Vault secret:
```json
"password": {
  "type": "AzureKeyVaultSecret",
  "secretName": "SqlPassword",
  "store": { "referenceName": "AzureKeyVaultLS", "type": "LinkedServiceReference" }
}
```  
【11†L169-L177】.

**Best Practices & Pitfalls:**  
- **Parameterize** pipelines and datasets for reusability (e.g. date partitions).  
- Use **Incremental loads** (watermark columns) to avoid full-copy.  
- Leverage **Mapping Data Flows** or external compute (Databricks/Synapse) for complex transformations.  
- Monitor runs via ADF’s monitoring, and handle failures with **try/catch** and alerts.  
- **Pitfalls:** Large-scale transformations in Copy activity can fail; switch to Data Flows/Spark. Beware of multiple concurrent copy tasks hitting limits. Secure credentials with Key Vault【11†L74-L83】.

**Sample Data & Output:** In one example, ADF copies a `customers` table to blob as CSV【25†L487-L492】:  
```
id,name,city
1,Atul,Pune
2,Rahul,Mumbai
3,Neha,Delhi
```  
(See 【25†L487-L492】 for expected output.)  

**References:** Microsoft docs on Copy activity and pipelines【14†L54-L58】【11†L120-L128】; sample GitHub ADF project【25†L431-L439】【25†L487-L492】; official ADF Python SDK Quickstart【9†L269-L276】【9†L420-L427】.

## Azure Databricks (PySpark/Spark SQL)  
**Use Case:** Big data analytics and ETL using Apache Spark in a managed environment. Databricks is ideal for **distributed processing**, machine learning, and Delta Lake (unified data format) on Azure. Often used to process data in ADLS and load results into a warehouse.  

```mermaid
flowchart LR
  ADF_Pipeline[ADF Pipeline] --> Databricks[Azure Databricks Cluster]
  Databricks --> ADLS[(ADLS Gen2)]
  Databricks --> AzureSQL[(Azure SQL Database)]
```

**Code Snippet (PySpark in Databricks):**  
```python
# Create a DataFrame manually and display
data = [[2021, "test", "Albany", "M", 42]]
df1 = spark.createDataFrame(data, schema="Year int, First_Name STRING, County STRING, Sex STRING, Count int")
display(df1)  # Databricks 'display' for rich output【30†L269-L272】

# Load CSV from ADLS into DataFrame
df_csv = spark.read.csv("abfss://mycontainer@myaccount.dfs.core.windows.net/path/data.csv",
                        header=True, inferSchema=True, sep=",")
display(df_csv)  # show contents【30†L322-L326】

# Example Spark SQL query on DataFrame
df1.createOrReplaceTempView("sales")
result = spark.sql("SELECT County, SUM(Count) AS TotalCount FROM sales GROUP BY County")
result.show()
```
(Example based on Databricks tutorials【30†L269-L272】【30†L322-L326】.)  

**Spark SQL (SQL on Spark):** Within a notebook you can run `%sql` or `spark.sql()`. For example:  
```sql
%sql
SELECT country, SUM(sales) AS total_sales FROM sales_data GROUP BY country;
```  
Or with `spark.sql`:  
```python
spark.sql("SELECT Name, COUNT(*) AS cnt FROM dbo.Customers GROUP BY Name").show()
```
This uses Spark’s distributed engine for SQL queries.

**Configuration:** Use the Azure Databricks workspace to create clusters (choose Spark runtime version, VM size, auto-scaling, etc.). Ensure **cluster permissions** and **network** (VNet injection or public). Use **Databricks Secrets** (Secret Scopes) or Azure Key Vault-backed scopes for database credentials, storage keys, etc. For Delta Lake on ADLS Gen2, configure an **Azure Service Principal** or managed identity for DBFS and mount storage or use abfss URI (see Data Lake access).  

**Best Practices & Pitfalls:**  
- Use **Delta Lake format** for reliability (transactional writes, schema enforcement) and enable `spark.databricks.delta.optimizeWrite` and `autoCompact` for performance【28†L129-L137】.  
- Avoid using `collect()` on large datasets – rely on distributed actions (`write`, `count`, etc.).  
- Cache/persist intermediate DataFrames only when reused.  
- For notebooks, **parameterize** using `dbutils.widgets` (as shown above or in [24] L239-242).  
- Monitor jobs in the Databricks Jobs UI.  
- Pitfalls: Unbounded shuffles (e.g. groupBy on high-cardinality) can OOM. Use appropriate partitions. Secure secrets with scopes (no hard-coding passwords).

**Sample Data & Output:** For instance, if a CSV of baby names is processed, the displayed DataFrame might be:  
| Year | First_Name | County | Sex | Count |
|------|------------|--------|-----|-------|
| 2021 | test       | Albany | M   | 42    |  
(This is created by `spark.createDataFrame`【30†L269-L272】.)  

When writing out (e.g. `df.write.format("delta").save("/mnt/adls/output/sales")`), you get Delta files in ADLS.

**References:** Azure Databricks documentation and tutorials【30†L269-L272】【30†L322-L326】; Microsoft guidance on Delta Lake and Spark configuration【28†L129-L137】. Official databricks blog and docs provide similar examples (not cited directly here).

## Spark SQL (within Databricks/Synapse)  
**Use Case:** Run SQL queries on big data tables. Use Spark SQL for analytical queries or CTAS (Create Table As Select) operations on DataFrames or Hive tables.  

**Code Example:** In PySpark:  
```python
# Register DataFrame as a view and query it with SQL
df1.createOrReplaceTempView("df1_view")
spark.sql("SELECT Year, COUNT(*) AS cnt FROM df1_view GROUP BY Year").show()
```
Or in a notebook cell with SQL magic:  
```sql
%sql
SELECT country, SUM(amount) AS total_amount 
FROM big_sales_data 
GROUP BY country;
```
These commands leverage Spark’s distributed SQL engine.  

**Best Practices:** Persist views as tables (e.g. Delta tables) for repeated queries. Use **CTAS** for large result sets to avoid repeated scans. Push down filters to data source when possible.  

**Pitfalls:** Shuffles on GROUP BY can be heavy; ensure adequate cluster size. 

**References:** Spark SQL is covered in Spark docs; Databricks notebooks support `%sql` magics (implied from [30] code context). For example, creating a table and querying it in Synapse was shown in [47†L179-L187] and [47†L218-L220], similar to Spark SQL usage.

## Azure Data Lake Storage Gen2 (ADLS)  
**Use Case:** Scalable storage for big data (files, logs, raw datasets). ADLS Gen2 provides hierarchical namespaces on Blob Storage, enabling directory-like structure. Typical pattern: raw data ingested into ADLS, processed by Databricks/Synapse, and results stored back or loaded into warehouses.  

```mermaid
flowchart LR
  SourceSystems[Source Systems] --> Ingestion[Ingestion via ADF or Tools]
  Ingestion --> ADLS[(Azure Data Lake Gen2)]
  ADLS --> Databricks[Azure Databricks / Spark]
  ADLS --> Synapse[Azure Synapse]
  Databricks --> ProcessedData[Processed Data in ADLS]
  ProcessedData --> PowerBI[Power BI (via Synapse or Direct Lake)]
```

**Setup (CLI):** Create an ADLS-enabled storage account with hierarchical namespace:  
```bash
az storage account create \
  --name mystorageacct \
  --resource-group myResourceGroup \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2 \
  --hierarchical-namespace true
```  
【39†L145-L153】  
Create a filesystem (container):  
```bash
az storage container create --account-name mystorageacct --name rawdata
```  
Store credentials in Key Vault or use managed identity for access.

**Config Notes:** Use Azure RBAC (e.g. Storage Blob Data Owner role) or ACLs for fine-grained access. Mount ADLS in Databricks or access via `abfss://`. For Azure Functions/Databricks reading ADLS, often a **Service Principal** or Managed Identity is configured in Spark config【28†L107-L115】. 

**Best Practices:**  
- Organize data by **hierarchy and partitions** (e.g. `/year=2026/month=04/` folders).  
- Use **Lifecycle Management** rules for cold data.  
- Enable **Encryption** (server-side by default) and secure with Firewall/VNet.  
- Use **Access Keys sparingly**; prefer OAuth/Managed Identity.  

**Sample Data:** Example hierarchy:  
```
raw/
  sales/2026/04/01/data.csv
  logs/2026/04/01/log.json
processed/
  sales_summary.parquet
```
No specific output snippet – data can be any CSV/Parquet in ADLS.  

**References:** Setting up ADLS Gen2 described in Azure docs【39†L145-L153】. For end-to-end scenarios, see [28†L107-L115] on configuring Databricks access to ADLS.

## Azure SQL (Azure SQL Database / Managed Instance / SQL Server)  
**Use Case:** Relational data storage (OLTP) or as a sink for processed data. Azure SQL Database (PaaS) or Managed Instance provides SQL Server compatibility in cloud. Used for operational data or as a data warehouse (smaller workloads) and for feeding Power BI.  

```mermaid
flowchart LR
  Databricks[Databricks/Spark] --> AzureSQL[(Azure SQL Database)]
  ADF_Pipeline[ADF Pipeline] --> AzureSQL
  PowerApps[Apps] --> AzureSQL
```

**Setup (CLI and SQL):**  
```bash
# Create SQL Server and Database
az sql server create --name myserver --resource-group myrg --location eastus \
    --admin-user azureuser --admin-password P@ssw0rd!
az sql db create --resource-group myrg --server myserver \
    --name salesdb --service-objective S0
```  
(See [2†L317-L324] example.)  
Then connect via SSMS or code. Example T-SQL:  
```sql
CREATE TABLE dbo.Customers (
   CustomerId INT NOT NULL,
   Name NVARCHAR(50) NOT NULL,
   City NVARCHAR(50) NOT NULL
);
INSERT INTO dbo.Customers VALUES (1, 'Atul', 'Pune'), (2, 'Rahul', 'Mumbai'), (3, 'Neha', 'Delhi');
SELECT * FROM dbo.Customers;
```  
(Cf. [47†L179-L186] for table creation; [47†L203-L210] for inserts.)  

**Configuration:** Configure server **firewall** rules to allow your clients/ADF. Use **Azure AD authentication** or SQL auth as needed. For ADF/Databricks access, use **Virtual Network** or trusted services. Keep connection strings in Key Vault and use **Managed Identity** when possible. Scale by service tiers (DTUs or vCores).

**Best Practices & Pitfalls:**  
- **Index** and **Partition** large tables for performance.  
- Use **Parameterization** in queries/stored procedures to prevent injection.  
- Leverage **Azure AD** auth and **TDE** (transparent data encryption) for security.  
- **Pitfall:** Max concurrency and size differ from on-prem; plan sizing. Use **Failover Groups** for HA.  

**Sample Data & Output:** From [25], the sample CSV output after ADF copy was:  
```
id,name,city
1,Atul,Pune
2,Rahul,Mumbai
3,Neha,Delhi
```  
After loading into SQL, a SELECT returns the same rows.  

**References:** Official quickstart for connecting/querying Azure SQL【47†L179-L186】【47†L218-L220】; Azure CLI examples【2†L317-L324】; Python connectivity guides【49†L243-L251】. 

## Azure Logic Apps  
**Use Case:** Event-driven orchestration and automation. Logic Apps can trigger on schedules, HTTP, or other events, and call Azure services (including ADF pipelines) via connectors or REST. Example: a daily Logic App run invokes an ADF pipeline or Databricks notebook, or reacts to a Blob upload.

```mermaid
flowchart LR
  Schedule((Schedule)) --> LogicApp[Logic App]
  LogicApp --> ADF[Trigger ADF Pipeline (via Azure Data Factory connector or HTTP)]
  ADF --> Databricks[Run Databricks Notebook]
  Databricks --> ADLS[(Store results in ADLS)]
  LogicApp --> Email[Send Notification]
```

**Sample Workflow:** A Logic App with a Recurrence trigger calls ADF:  
- **Recurrence trigger** (daily) → **Create pipeline run** (Azure Data Factory connector) to start `CopyPipeline` with JSON body containing parameters.  

Sample JSON (Logic Apps definition excerpt):  
```json
{
  "actions": {
    "Run_ADF_Pipeline": {
      "type": "ApiConnection",
      "inputs": {
        "host": { "connection": {"name": "@parameters('$connections')['azuredatafactory']['connectionId']"} },
        "method": "post",
        "path": "/pipelines/CopyPipeline/createRun",
        "queries": { "api-version": "2018-06-01" },
        "body": { "param1": "value1", "param2": "value2" }
      }
    }
  },
  "triggers": { "Recurrence": { "type": "Recurrence", "interval": 1, "frequency": "Day" } }
}
```  
(See [42†L225-L233] for Logic Apps JSON body format.)  

**Configuration:** In Logic Apps designer, use the **Azure Data Factory connector** (“Create Pipeline Run”) or an HTTP action (invoke ADF REST API) to start pipelines. Provide ADF workspace, pipeline name, and parameters. For securing Logic App HTTP endpoints, configure **Access Keys** or OAuth. Manage identity for Logic App to call ADF securely.  

**Best Practices & Pitfalls:**  
- Use **idempotency tokens** if re-running runs.  
- **Error handling:** configure actions for error/failure paths and notifications.  
- **Concurrency:** Logic App runs concurrently by default; control with settings if needed.  
- Pitfalls: Outgoing calls have defaults (timeouts, retries); test endpoints. Ensure Logic App’s managed identity has permissions if using Azure connectors.  

**References:** See Azure docs and examples on invoking ADF from Logic Apps. For parameter passing, see [42†L225-L233] which shows using the “Create a pipeline run” action. 

## Azure Key Vault  
**Use Case:** Securely store secrets, keys, certificates. Used by ADF (and Synapse) to retrieve database passwords, storage keys, service credentials at runtime【11†L59-L68】.  

```mermaid
flowchart LR
  ADF[Azure Data Factory] -->|Managed Identity| AKV[(Azure Key Vault)]
  AKV --> SecretStore[Secret: StorageKey, SQLPwd,...]
  ADF --> LinkedService1[Azure SQL Linked Service] 
  LinkedService1 --> SecretStore
  Databricks --> SecretStore
```

**Configuration:** Create an **Azure Key Vault** and add secrets (e.g. `StorageKey`, `SqlPassword`). Grant the ADF (or Synapse) **managed identity** Get/List permission【11†L78-L87】. In ADF linked service JSON, reference secrets:  
```json
"linkedServiceName": {"referenceName": "AzureKeyVaultLS", "type": "LinkedServiceReference"},
"typeProperties": {
  "secretName": "StorageSecret",
  "store": {
     "referenceName": "AzureKeyVaultLS",
     "type": "LinkedServiceReference"
  }
}
```  
For example, in a SQL linked service, use `"type": "AzureKeyVaultSecret"` for the password field【11†L169-L177】.

**Best Practices:**  
- Enable **soft-delete** and purge-protection on Key Vault.  
- Rotate secrets regularly.  
- Use **Access Policies or RBAC** for fine-grained control (Key Vault secrets user).  
- Limit Key Vault access only to necessary identities (least privilege).  

**Pitfalls:** Failure to update pipeline if secret version changes; ensure correct `secretName`/version or omit version for latest.  

**References:** Key Vault integration guide【11†L120-L128】【11†L169-L177】 shows JSON examples of linked service and secret references.

## Azure Synapse Analytics  
**Use Case:** Enterprise data warehousing and analytics platform. Offers **Dedicated SQL Pools (SQL DW)** for large-scale MPP warehousing, **Serverless SQL Pools** for on-demand queries, **Spark pools** for big data processing, and integrated **pipelines** (formerly Data Factory) for ETL. Use Synapse for large SQL-based analytics and BI workloads.  

```mermaid
flowchart LR
  ADF --> SynapseSQL[(Synapse Dedicated SQL Pool)]
  ADLS[(Data Lake)] --> SynapseSpark[Synapse Spark Pool]
  SynapseSpark --> SynapseSQL
  SynapseSQL --> PowerBI[Power BI Reporting]
```

**SQL Example (Synapse Dedicated Pool):** T-SQL via SSMS or Synapse Studio:  
```sql
-- Create table
CREATE TABLE dbo.Customers (
    CustomerId INT NOT NULL,
    Name NVARCHAR(50),
    Location NVARCHAR(50),
    Email NVARCHAR(50)
);
GO
-- Insert sample rows
INSERT INTO dbo.Customers VALUES 
  (1, N'Orlando', N'Australia', N''),
  (2, N'Keith',   N'India',     N'keith0@example.com');
GO
-- Query the table
SELECT * FROM dbo.Customers;
```  
(Example from Synapse docs【47†L179-L186】【47†L218-L220】.)  

**Configuration:** Create a **Synapse workspace** with SQL Pools. Use **Synapse Studio** to author queries. For larger files, use **COPY** or **PolyBase** to load from ADLS. Spark pools require linked storage and Spark cluster setup. Manage access via Azure AD and firewall/VNet. Use **Azure Key Vault** for secrets (supported in linked services, similar to ADF).

**Best Practices:**  
- Distribute tables properly: use **CLUSTERED COLUMNSTORE** and proper distribution (HASH on keys, ROUND_ROBIN for staging)【47†L179-L186】.  
- Use **CTAS** (CREATE TABLE AS SELECT) for performance on big queries.  
- Partition large date/time tables.  
- Scale out (DWUs/vCores) according to workload.  
- Synapse Pipelines (similar to ADF) for orchestration.  

**Pitfalls:** Data skew if wrong distribution key. Long-running queries consume DWU; consider workload management (resource classes).  

**References:** Azure Synapse quickstarts and docs【47†L179-L186】【47†L218-L220】; design recommendations in Synapse docs. 

## Microsoft Fabric  
**Use Case:** Unified SaaS analytics platform (announced 2023) combining Data Factory–like pipelines, OneLake (data lake), Synapse-like SQL Warehouses, and Power BI. Fabric is ideal if you want end-to-end analytics (ingestion, transformation, warehousing, BI) in one UI, especially with Power BI integration.  

```mermaid
flowchart LR
  DataSources[Data Sources] --> FabricIngestion[Dataflows (Fabric)]
  FabricIngestion --> OneLake[(OneLake Fabric)]
  OneLake --> FabricNotebooks[Notebooks (Spark)]
  FabricNotebooks --> FabricWarehouse[(Fabric Warehouse)]
  FabricWarehouse --> PowerBI[Power BI]
```

**Key Points:** Fabric includes **Data Factory-compatible pipelines**, **Dataflows**, **Lakehouses (OneLake)**, **Notebooks**, and **Power BI Data Warehouses**. It targets low-code data engineering. Workloads that require tight Power BI integration and a unified experience are well-suited to Fabric【51†L151-L157】.

**Configuration:** Use the Microsoft Fabric portal to create a workspace, establish a OneLake (fabric-managed data lake), and enable Data Factory pipelines and Warehouse. Security uses Azure AD and OneLake access controls. Migrating ADF workloads to Fabric is a path forward【5†L51-L59】.

**Best Practices:**  
- Leverage **Power BI dataflows** and **Lakehouse** for data lake operations.  
- Use Fabric Warehouses for hosting datasets used by Power BI.  
- Fabric pipelines support many of ADF features plus built-in AI tasks (preview).  
- Expect Fabric to evolve; for now, use as complementary to ADF if already committed to Power BI.  

**Pitfalls:** Fabric is new; check feature availability. It may not replace all Synapse features yet【51†L151-L157】.  

**References:** Official Fabric overview【51†L153-L157】, and the ADF docs note Fabric as “next-gen” Data Factory【5†L51-L59】. 

## RDBMS: SQL Server  
**Use Case:** On-premises (or IaaS) SQL database workloads. Similar to Azure SQL but in customer-managed VMs or data centers. Used for legacy OLTP and as a source/destination in pipelines.  

**Example (T-SQL):**  
```sql
CREATE DATABASE SalesDB;
GO
USE SalesDB;
CREATE TABLE Inventory(ProductID INT PRIMARY KEY, Qty INT, Warehouse VARCHAR(50));
INSERT INTO Inventory VALUES (1, 100, 'NY'), (2, 200, 'SF');
SELECT * FROM Inventory;
```  
Standard SQL Server features (stored procs, triggers, etc.) apply.  

**Configuration:** Configure **Always On Availability Groups** or log shipping for HA, **SQL Agent** for scheduling, and on-prem firewalls. For ADF integration, use a **Self-Hosted IR** to connect on-prem to Azure.  

**Best Practices:** Follow SQL Server best practices: indexing, normalization, backup and DR plans. Use **Windows/AD Authentication** if possible.  

**References:** SQL Server documentation for DB design and queries. ADF on-prem integration is covered in Azure docs (not explicitly cited here).

## Programming: Python and SQL (integration examples)  
**Use Case:** Glue code to orchestrate data tasks or run queries programmatically.  

**Python Example (Azure SQL):** Connect and query:  
```python
import pyodbc

conn = pyodbc.connect(
    'Driver={ODBC Driver 18 for SQL Server};'
    'Server=tcp:myserver.database.windows.net,1433;'
    'Database=salesdb;UID=azureuser;PWD=<YourPassword>;Encrypt=yes;TrustServerCertificate=no;'
)
cursor = conn.cursor()
cursor.execute("SELECT CustomerId, Name FROM dbo.Customers;")
for row in cursor.fetchall():
    print(row)
conn.close()
```
This uses `pyodbc` driver. For managed identity, see Microsoft’s `mssql-python` driver supporting Azure AD auth【49†L185-L193】【49†L313-L321】.  

**SQL in Notebooks:** You can also use Python libraries (like Pandas) with SQL, e.g.:  
```python
import pandas as pd
df = pd.read_sql("SELECT * FROM Customers", conn)
print(df)
```
Uses `pandas` with a DB connection.  

**Best Practices:**  
- Always **parameterize** SQL queries in code (use `?` or `{}` params) to avoid injection.  
- Use **environment variables** or Azure Key Vault for credentials (see [49] on `.env`).  
- Close connections and handle exceptions.  

**Pitfalls:** Leaving credentials in code; long-running queries blocking applications.  

**References:** Azure quickstart for Python+SQL connection【49†L185-L193】【49†L313-L321】. 

## Databricks vs Synapse vs Fabric (Comparison)  

| **Feature / Workload**          | **Azure Databricks**                                 | **Azure Synapse Analytics**                         | **Microsoft Fabric**                              |
|---------------------------------|------------------------------------------------------|----------------------------------------------------|---------------------------------------------------|
| **Primary Use Cases**           | Large-scale Spark-based ETL/ML, multi-tenant data processing【51†L145-L153】 | All-in-one analytics with SQL warehouses + Spark【51†L139-L147】 (legacy Synapse workloads) | Unified SaaS data platform (pipelines, lakehouse, BI)【51†L153-L157】 |
| **Transformation Engine**       | Spark (PySpark/Scala) on managed clusters【51†L145-L153】  | Spark + SQL on dedicated pools. SQL-first approach (DDL/CTAS) | Spark (Notebooks) + Dataflows (low-code ETL)      |
| **Data Lake Integration**       | Natively reads/writes ADLS Gen2 (Delta Lake)         | Uses ADLS for external tables, PolyBase, or Spark pools | OneLake (fabric-managed data lake)                |
| **SQL/BI Integration**          | Can run Spark SQL; integrates with Power BI via Warehouse | Strong SQL (MPP); integrates with Power BI (and now Fabric Lakehouse) | Tightest Power BI integration (Fabric Lakes, Warehouses)【51†L151-L157】 |
| **Best Fit**                    | Code-centric data engineering & ML, data science teams【51†L145-L153】 | Legacy data warehouse scenarios, where SQL is dominant | Low-code enterprise ETL/analytics within MS ecosystem【51†L153-L157】 |
| **Skill Level Required**        | High (Spark/Python/Scala, DevOps)                    | Medium (SQL + some Spark/SQL Pools)                 | Low-Medium (drag-drop, some script)               |
| **When to Use**                | - Heavy transformations, ML<br>- Multi-client data pipelines【51†L145-L153】 | - Corporate data warehouse (structured)<br>- Ad-hoc SQL analytics | - End-to-end MS ecosystem solutions<br>- Power BI heavy workloads【51†L153-L157】 |
| **Notes**                       | Delta Lake for reliability; built for engineering    | Includes pipelines (like ADF), but Fabric may supersede some parts【51†L139-L147】 | Newest platform; evolving rapidly.               |

*Data drawn from Microsoft guidance【51†L145-L153】【51†L153-L157】.*

## End-to-End Pipeline Example: ADF → Databricks → ADLS → Azure SQL (via Logic Apps)  
**Scenario:** A Logic App triggers an ADF pipeline daily. The pipeline runs a Databricks notebook that processes raw data from ADLS and writes results to ADLS and Azure SQL.  

```mermaid
flowchart LR
    LogicApp[Logic App\n(Daily Trigger)] --> ADF[ADF Pipeline]
    ADF --> RunNotebook[Databricks Notebook Activity]
    RunNotebook --> ADLS[(ADLS Gen2)]
    RunNotebook --> AzureSQL[(Azure SQL Database)]
    ADLS --> AzureSQL
    LogicApp --> Notify[Email Notification]
```

- **Logic App:** Scheduled trigger (e.g. Recurrence) calls ADF’s **Create pipeline run** (via Azure connector or HTTP) to start the pipeline with any parameters.
- **ADF Pipeline:** Contains a **DatabricksNotebook** activity. Example JSON snippet【55†L98-L107】【55†L108-L112】:
  ```json
  {
    "name": "RunDatabricksNotebook",
    "type": "DatabricksNotebook",
    "linkedServiceName": { "referenceName": "AzureDatabricks_LinkedService", "type": "LinkedServiceReference" },
    "typeProperties": {
      "notebookPath": "/adftutorial/mynotebook",
      "baseParameters": { "input": "@pipeline().parameters.Name" }
    }
  }
  ```
  (This invokes the `/adftutorial/mynotebook` notebook with an input parameter from ADF【24†L239-L244】【55†L98-L107】.)
- **Databricks Notebook:** Reads raw data from ADLS (e.g. JSON/CSV), transforms it (PySpark), and writes output:
  - Writes **refined files** to ADLS (e.g. `abfss://.../processed/`).
  - Loads final aggregates into Azure SQL (using JDBC):
    ```python
    df_result.write.format("com.microsoft.sqlserver.jdbc.spark").options(
        url="jdbc:sqlserver://myserver.database.windows.net:1433;database=salesdb",
        dbtable="dbo.ProcessedResults",
        user="azureuser", password="P@ssw0rd"
    ).mode("overwrite").save()
    ```
- **Azure SQL:** Receives processed data. ADF or notebook JDBC ensure it’s up-to-date for reporting.
- **ADF/Logic App Continuation:** After pipeline succeeds, Logic App can send a success/failure email or trigger downstream tasks.  

**Sample Data Flow:** Raw CSV in ADLS → Databricks cleans and summarizes → writes Parquet to ADLS and updates Azure SQL table → Power BI reads from Azure SQL.  

**References:** The above Databricks activity JSON is from ADF docs【55†L98-L107】. The Logic App invocation pattern follows Azure integration patterns. For complete examples, see the ADF+Databricks tutorial【24†L234-L242】 and Logic Apps + ADF connector (described in [42†L225-L233]).

### References  
- Azure Data Factory docs (pipelines, copy activity)【14†L54-L58】【11†L120-L128】  
- ADF Python SDK Quickstart (pipelines)【9†L269-L276】【9†L420-L427】  
- ADF GitHub sample (Copy from SQL to Blob)【25†L431-L439】【25†L487-L492】  
- Azure Databricks tutorials (PySpark DataFrames)【30†L269-L272】【30†L322-L326】  
- ADF Notebook activity for Databricks【55†L98-L107】【55†L108-L112】  
- Azure Data Lake Gen2 setup (Azure CLI)【39†L145-L153】  
- Synapse SQL pool quickstart (create table/query)【47†L179-L186】【47†L218-L220】  
- Azure SQL Python sample (mssql driver)【49†L185-L193】【49†L313-L321】  
- ADF–Logic Apps integration (passing parameters)【42†L225-L233】  
- Azure Key Vault + ADF integration【11†L120-L128】【11†L169-L177】  
- Platform comparison guidance【51†L145-L153】【51†L153-L157】.  

Each example is drawn from official Azure documentation or authoritative Microsoft blogs/samples to ensure accuracy and current best practices.
