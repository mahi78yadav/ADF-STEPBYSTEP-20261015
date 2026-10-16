# ADF Retail Data Pipeline – Precise Steps

## Step 1: Create Azure Data Lake Storage

1. Open Azure Portal → Create Resource group  
   Name: `rg-retail-data-dev` (or sit/uat as needed)

2. Create Storage account  
   Name: `strretaildata202610` (or `st-<business>-yyyymmdd`)

3. Inside storage account → Data storage → Containers → Create:
   - `raw`
   - `processed`
   - `archive`
   - `config`

4. Create folders inside `raw`:
```text
raw/
├── customers/
├── products/
├── stores/
├── orders/
└── SourceFiles/          ← upload Customers.csv, Products.csv, Stores.csv, Sales.csv here
```

5. Create same folder structure inside `processed`:
```text
processed/
├── customers/
├── products/
├── stores/
├── orders/
└── Destination/
```

---

## Step 2: Create Azure Data Factory

1. Azure Portal → Create → Data factories
2. Basics:
   | Field            | Value                        |
   |------------------|------------------------------|
   | Subscription     | Azure subscription 1         |
   | Resource group   | `rg-retail-data-dev`         |
   | Name             | `adf-retail-data-dev202610`  |
   | Region           | South India                  |
   | Version          | V2                           |
3. Networking: Public endpoint
4. Review + Create → Launch ADF Studio

---

## Step 3: Enable Managed Identity

1. Open ADF → Settings → Managed identities → System assigned
2. Status = **On** → Save

---

## Step 4: Grant Storage Permission

1. Open Storage account → Access control (IAM)
2. Add role assignment:
   - Role: **Storage Blob Data Contributor**
   - Assign access to: Managed identity
   - Select: `adf-retail-data-dev202610`
3. Save

---

## Step 5: Create Linked Service (ADLS Gen2)

1. ADF Studio → Manage → Linked services → + New
2. Select **Azure Data Lake Storage Gen2**
3. Settings:
   - Name: `LS_ADLS_Retail`
   - Authentication: **Managed Identity**
   - Storage account: `strretaildata202610`
4. Test connection → Create

---

## Step 6: Create Source Dataset (Folder Level)

1. Author → + → Dataset → Azure Data Lake Storage Gen2 → DelimitedText
2. Name: `DS_Retail_Raw_Folder`
3. Linked service: `LS_ADLS_Retail`
4. File path: Container = `raw`  (leave Directory & File blank)
5. First row as header: **Checked**
6. Import schema: **None**
7. OK

---

## Step 7: Create Destination Dataset

1. Author → + → Dataset → Azure Data Lake Storage Gen2 → DelimitedText
2. Name: `DS_Retail_Processed`
3. Linked service: `LS_ADLS_Retail`
4. File path: Container = `processed`  (leave Directory & File blank)
5. First row as header: **Checked**
6. Import schema: **None**
7. OK

---

## Step 8: Simple Recursive Copy Pipeline

1. Create pipeline: `PL_Retail_File_Ingestion`
2. Add **Copy Data** activity → name: `CP_Copy_Raw_To_Processed`
3. Source:
   - Dataset: `DS_Retail_Raw_Folder`
   - File path type: **Wildcard file path**
   - Wildcard file name: `*.csv`
   - Recursive: **On**
4. Sink:
   - Dataset: `DS_Retail_Processed`
   - Copy behavior: **Preserve hierarchy**
5. Validate → Debug → Verify files appear in `processed` with same folder structure

---

## Step 9: Dynamic File Copy (Get Metadata + ForEach)

### 9.1 Create Parameterized Datasets

**Source file dataset**
1. New Dataset → ADLS Gen2 → DelimitedText
2. Name: `DS_Raw_SourceFiles`
3. Linked service: `LS_ADLS_Retail`
4. File path: Container=`raw`, Directory=`SourceFiles`
5. Parameters → + New → Name: `pFileName` (String)
6. File name: `@dataset().pFileName`
7. First row as header: Checked | Schema: None

**Destination file dataset**
1. New Dataset → ADLS Gen2 → DelimitedText
2. Name: `DS_Processed_Destination`
3. Linked service: `LS_ADLS_Retail`
4. File path: Container=`processed`, Directory=`Destination`
5. Parameters → + New → Name: `pFileName` (String)
6. File name: `@dataset().pFileName`
7. First row as header: Checked | Schema: None

### 9.2 Build Pipeline

1. Create / open pipeline: `PL_Retail_File_Ingestion_Dynamic`
2. Add **Get Metadata** → name: `GM_Get_Source_Files`
   - Dataset: `DS_Retail_Raw_Folder` (point it to `raw/SourceFiles`)
   - Field list: **Child items**
3. Add **ForEach** → name: `FE_Process_Files`
   - Items: `@activity('GM_Get_Source_Files').output.childItems`
4. Inside ForEach → Add **Copy Data** → name: `CP_Copy_File`
   - Source dataset: `DS_Raw_SourceFiles`  
     Parameter `pFileName` = `@item().name`
   - Sink dataset: `DS_Processed_Destination`  
     Parameter `pFileName` = `@item().name`
5. Validate → Debug → Confirm 4 files appear in `processed/Destination/`

---

## Step 10: Incremental Loading with Watermark

### 10.1 Create Watermark Table (Azure SQL)

Database: `sqldb-retail-watermark`

```sql
CREATE TABLE dbo.FileIngestionWatermark
(
    FileName    VARCHAR(255) PRIMARY KEY,
    ProcessedOn DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);
```

### 10.2 Create Azure SQL Linked Service & Dataset

1. Linked service: Azure SQL Database → name `LS_AzureSQL_Watermark`  
   (use SQL auth or Managed Identity)
2. Dataset: Azure SQL Database → name `DS_SQL_Watermark`  
   Table: `dbo.FileIngestionWatermark`

### 10.3 Pipeline Logic (inside ForEach)

```
GM_Get_Source_Files
        ↓
FE_Process_Files
        ↓
    LK_Check_Watermark          ← Lookup
        ↓
    IF_File_Not_Processed       ← If Condition
        ├── True branch
        │       CP_Copy_File
        │           ↓ (Success)
        │       SQL_Record_Processed_File
        └── False branch (empty)
```

**Lookup activity** (`LK_Check_Watermark`)
- Dataset: `DS_SQL_Watermark`
- Use query:
```sql
SELECT COUNT(*) AS FileCount
FROM dbo.FileIngestionWatermark
WHERE FileName = '@{item().name}';
```
- First row only: **On**

**If Condition** (`IF_File_Not_Processed`)
- Expression:
```
@equals(activity('LK_Check_Watermark').output.firstRow.FileCount, 0)
```

**Copy Data** (True branch) – same as Step 9

**SQL activity / Stored Procedure** (`SQL_Record_Processed_File`)
- After Copy succeeds
- Insert:
```sql
INSERT INTO dbo.FileIngestionWatermark (FileName, ProcessedOn)
VALUES (@FileName, SYSUTCDATETIME());
```
- Parameter `FileName` = `@item().name`

### 10.4 Test Incremental

1. Run pipeline → 4 files copied + recorded in watermark table
2. Run again → all files skipped
3. Upload new CSV → only new file is copied and recorded

---

## Step 11: Error Handling & Failure Logging

### 11.1 Create Error Log Table & Procedure (once)

```sql
CREATE TABLE dbo.FileIngestionErrorLog
(
    ErrorLogID     INT IDENTITY(1,1) PRIMARY KEY,
    FileName       VARCHAR(255),
    FailedActivity VARCHAR(100),
    ErrorMessage   NVARCHAR(MAX),
    FailedOn       DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);
GO

CREATE PROCEDURE dbo.LogFileIngestionFailure
    @FileName       VARCHAR(255),
    @FailedActivity VARCHAR(100),
    @ErrorMessage   NVARCHAR(MAX)
AS
BEGIN
    INSERT INTO dbo.FileIngestionErrorLog
        (FileName, FailedActivity, ErrorMessage)
    VALUES
        (@FileName, @FailedActivity, @ErrorMessage);
END
```

### 11.2 Add Failure Logging Activities

**Inside True branch of If Condition**

1. After `CP_Copy_File` → add Stored Procedure activity  
   Name: `SQL_Log_Copy_Failure`
   - Linked service: same Azure SQL
   - Stored procedure: `dbo.LogFileIngestionFailure`
   - Parameters:
     | Parameter      | Value                                              |
     |----------------|----------------------------------------------------|
     | FileName       | `@item().name`                                     |
     | FailedActivity | `CP_Copy_File`                                     |
     | ErrorMessage   | `@string(activity('CP_Copy_File').error.message)`  |
   - Connect with **Failure** dependency (red arrow)

2. (Optional) After watermark activity → add another Stored Procedure  
   Name: `SQL_Log_Watermark_Failure`
   - Same procedure
   - Parameters:
     | Parameter      | Value                                                      |
     |----------------|------------------------------------------------------------|
     | FileName       | `@item().name`                                             |
     | FailedActivity | `SQL_Record_Processed_File`                                |
     | ErrorMessage   | `@string(activity('SQL_Record_Processed_File').error.message)` |
   - Connect with **Failure** dependency

### 11.3 Verify

```sql
SELECT * FROM dbo.FileIngestionErrorLog ORDER BY FailedOn DESC;
```

---

## Final Checklist

- [ ] Storage account + containers + folders created
- [ ] ADF created + Managed Identity On
- [ ] Storage Blob Data Contributor role assigned
- [ ] Linked service `LS_ADLS_Retail` works
- [ ] Datasets created (folder + parameterized file)
- [ ] Recursive copy pipeline works
- [ ] Dynamic copy with Get Metadata + ForEach works
- [ ] Watermark table + Lookup + If Condition works
- [ ] Incremental test (skip existing files) succeeds
- [ ] Error log table + procedure + Failure paths configured
- [ ] Pipeline validated and published
