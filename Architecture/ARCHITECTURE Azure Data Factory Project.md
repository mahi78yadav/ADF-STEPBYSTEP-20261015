# 🏗️ ADF Retail Data Ingestion – End-to-End Architecture

> **Project**: `ADF-STEPBYSTEP-20261015`  
> **Goal**: Build a production-grade, incremental, fault-tolerant file ingestion pipeline from Azure Data Lake (Raw) → Processed using Azure Data Factory + Azure SQL Watermark & Error Logging.

---

## 1. High-Level End-to-End Architecture

```mermaid
flowchart TB
    subgraph Azure["Azure Cloud"]
        direction TB

        subgraph RG["Resource Group: rg-retail-data-dev"]
            direction TB

            subgraph Storage["Azure Data Lake Storage Gen2<br/>strretaildata202610"]
                direction LR
                RAW["📦 raw container"]
                PROC["📦 processed container"]
                ARCH["📦 archive container"]
                CONF["📦 config container"]
            end

            subgraph ADF["Azure Data Factory V2<br/>adf-retail-data-dev202610"]
                direction TB
                MI["🔐 System-Assigned<br/>Managed Identity"]
                LS["🔗 Linked Services"]
                DS["📄 Datasets"]
                PL["⚙️ Pipelines"]
            end

            subgraph SQL["Azure SQL Database<br/>sqldb-retail-watermark"]
                direction TB
                WM["📊 dbo.FileIngestionWatermark"]
                ERR["🚨 dbo.FileIngestionErrorLog"]
                SP["⚙️ dbo.LogFileIngestionFailure"]
            end
        end
    end

    %% Data Flow
    RAW -->|CSV Files| ADF
    ADF -->|Copy + Watermark Check| PROC
    ADF -->|Success| WM
    ADF -->|Failure| ERR
    MI -.->|Storage Blob Data Contributor| Storage
    LS --> Storage
    LS --> SQL
    DS --> LS
    PL --> DS
    PL --> SP

    style RAW fill:#e3f2fd,stroke:#1565c0
    style PROC fill:#e8f5e9,stroke:#2e7d32
    style WM fill:#fff3e0,stroke:#ef6c00
    style ERR fill:#ffebee,stroke:#c62828
    style ADF fill:#f3e5f5,stroke:#7b1fa2
```

---

## 2. Storage Layer – Detailed Structure

```mermaid
flowchart LR
    subgraph ADLS["Azure Data Lake Gen2 – strretaildata202610"]
        direction TB

        subgraph Raw["raw /"]
            direction TB
            SF["SourceFiles/"]
            C["customers/"]
            P["products/"]
            S["stores/"]
            O["orders/"]
        end

        subgraph Processed["processed /"]
            direction TB
            DEST["Destination/"]
            C2["customers/"]
            P2["products/"]
            S2["stores/"]
            O2["orders/"]
        end

        subgraph Other["Other Containers"]
            ARCH["archive/"]
            CONF["config/"]
        end
    end

    SF --> |"Customers.csv<br/>Products.csv<br/>Stores.csv<br/>Sales.csv"| DEST

    style SF fill:#bbdefb
    style DEST fill:#c8e6c9
```

### Folder Structure (Exact)

```text
raw/
├── SourceFiles/                  ← Used in Step 9 & 10
│   ├── Customers.csv
│   ├── Products.csv
│   ├── Stores.csv
│   └── Sales.csv
├── customers/
│   └── customers.csv
├── products/
│   └── products.csv
├── stores/
│   └── stores.csv
└── orders/
    └── orders.csv

processed/
├── Destination/                  ← Target for dynamic/incremental copy
│   ├── Customers.csv
│   ├── Products.csv
│   ├── Stores.csv
│   └── Sales.csv
├── customers/
├── products/
├── stores/
└── orders/
```

---

## 3. Linked Services & Datasets Mapping

```mermaid
flowchart TB
    subgraph LinkedServices["Linked Services"]
        LS_ADLS["LS_ADLS_Retail<br/>(Managed Identity)"]
        LS_SQL["Azure SQL Linked Service<br/>(sqldb-retail-watermark)"]
    end

    subgraph Datasets["Datasets"]
        DS1["DS_Retail_Source<br/>→ raw"]
        DS2["DS_Retail_Processed<br/>→ processed"]
        DS3["DS_Retail_Raw_Folder<br/>→ raw/SourceFiles"]
        DS4["DS_Raw_SourceFiles<br/>(parameterized pFileName)"]
        DS5["DS_Processed_Destination<br/>(parameterized pFileName)"]
        DS6["Azure SQL Dataset<br/>→ watermark & error log"]
    end

    LS_ADLS --> DS1
    LS_ADLS --> DS2
    LS_ADLS --> DS3
    LS_ADLS --> DS4
    LS_ADLS --> DS5
    LS_SQL  --> DS6

    style LS_ADLS fill:#e1bee7
    style LS_SQL fill:#ffccbc
```

---

## 4. Pipeline Evolution Overview

| Step | Pipeline Name                        | Pattern                          | Key Activities                          |
|------|--------------------------------------|----------------------------------|-----------------------------------------|
| 8    | `PL_Retail_File_Ingestion`           | Recursive Wildcard Copy          | Copy Data (Recursive + Preserve Hierarchy) |
| 9    | `PL_Retail_File_Ingestion_Dynamic`   | Get Metadata + ForEach           | Get Metadata → ForEach → Copy           |
| 10   | `PL_Retail_File_Ingestion_Dynamic`   | **Incremental + Watermark**      | Get Metadata → ForEach → Lookup → If → Copy → SQL Insert |
| 11   | Same Pipeline                        | **Error Handling**               | + Failure paths to Stored Procedure     |

---

## 5. Detailed Incremental Load Pipeline (Step 10) – Micro Level

```mermaid
flowchart TD
    Start([Pipeline Start]) --> GM

    GM["🔍 GM_Get_Source_Files<br/>Get Metadata<br/>Field: Child Items<br/>Dataset: Raw/SourceFiles"]

    GM --> FE

    FE["🔁 FE_Process_Files<br/>ForEach<br/>Items: @activity('GM_Get_Source_Files').output.childItems"]

    FE --> LK

    LK["🔎 LK_Check_Watermark<br/>Lookup Activity<br/>Query: COUNT of matching FileName<br/>in FileIngestionWatermark table"]

    LK --> IF

    IF{"❓ IF_File_Not_Processed<br/>FileCount equals 0?"}

    IF -->|True - New File| CP
    IF -->|False - Already Processed| Skip([Skip File])

    CP["📋 CP_Copy_File<br/>Copy Data<br/>Source: Raw/SourceFiles/@item().name<br/>Sink: Processed/Destination/@item().name"]

    CP -->|Success| SQL_WM
    CP -->|Failure| SQL_ERR_COPY

    SQL_WM["💾 SQL_Record_Processed_File<br/>Azure SQL / Stored Procedure<br/>INSERT INTO FileIngestionWatermark<br/>(FileName, ProcessedOn)"]

    SQL_ERR_COPY["🚨 SQL_Log_Copy_Failure<br/>Stored Procedure<br/>dbo.LogFileIngestionFailure<br/>FileName = @item().name<br/>FailedActivity = 'CP_Copy_File'"]

    SQL_WM -->|Success| End([Next File / End])
    SQL_WM -->|Failure| SQL_ERR_WM

    SQL_ERR_WM["🚨 SQL_Log_Watermark_Failure<br/>(Optional)<br/>dbo.LogFileIngestionFailure"]

    style GM fill:#bbdefb
    style FE fill:#c5cae9
    style LK fill:#fff9c4
    style IF fill:#ffe0b2
    style CP fill:#c8e6c9
    style SQL_WM fill:#dcedc8
    style SQL_ERR_COPY fill:#ffcdd2
    style SQL_ERR_WM fill:#ffcdd2
```

---

## 6. Activity Dependency Graph (Success vs Failure Paths)

```mermaid
flowchart LR
    GM["GM_Get_Source_Files"] -->|Success| FE["FE_Process_Files"]

    subgraph ForEach["Inside ForEach"]
        LK["LK_Check_Watermark"] -->|Success| IF["IF_File_Not_Processed"]

        IF -->|True| CP["CP_Copy_File"]

        CP -->|Success| WM["SQL_Record_Processed_File"]
        CP -->|Failure| ERR1["SQL_Log_Copy_Failure"]

        WM -->|Failure| ERR2["SQL_Log_Watermark_Failure"]
    end

    style CP fill:#a5d6a7
    style WM fill:#a5d6a7
    style ERR1 fill:#ef9a9a
    style ERR2 fill:#ef9a9a
```

---

## 7. Sequence Diagram – Incremental File Processing

```mermaid
sequenceDiagram
    participant Pipeline
    participant GetMetadata
    participant ForEach
    participant Lookup
    participant IfCondition
    participant Copy
    participant WatermarkSQL
    participant ErrorSQL
    participant ADLS
    participant AzureSQL

    Pipeline->>GetMetadata: List files in Raw/SourceFiles
    GetMetadata->>ADLS: Child Items
    ADLS-->>GetMetadata: [Customers.csv, Products.csv, ...]
    GetMetadata-->>Pipeline: childItems

    loop For each file
        Pipeline->>ForEach: item = current file
        ForEach->>Lookup: Check if FileName exists in Watermark
        Lookup->>AzureSQL: SELECT COUNT(*) WHERE FileName = @item().name
        AzureSQL-->>Lookup: FileCount

        alt New File - FileCount is 0
            Lookup-->>IfCondition: True
            IfCondition->>Copy: Copy file
            Copy->>ADLS: Read from Raw/SourceFiles
            Copy->>ADLS: Write to Processed/Destination
            Copy-->>IfCondition: Success
            IfCondition->>WatermarkSQL: INSERT FileName + Timestamp
            WatermarkSQL->>AzureSQL: Record success
        else File already processed
            Lookup-->>IfCondition: False - Skip
        end

        opt Copy Failed
            Copy-->>ErrorSQL: Log failure
            ErrorSQL->>AzureSQL: INSERT into FileIngestionErrorLog
        end
    end
```

---

## 8. Azure SQL Control Tables

### Watermark Table
```sql
CREATE TABLE dbo.FileIngestionWatermark
(
    FileName     VARCHAR(255) PRIMARY KEY,
    ProcessedOn  DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME()
);
```

### Error Log Table
```sql
-- Columns
ErrorLogID      (Identity)
FileName
FailedActivity
ErrorMessage
FailedOn        (UTC)
```

### Stored Procedure
```sql
dbo.LogFileIngestionFailure
(
    @FileName       VARCHAR(255),
    @FailedActivity VARCHAR(255),
    @ErrorMessage   NVARCHAR(MAX)
)
```

---

## 9. Security & Identity Flow

```mermaid
flowchart LR
    ADF["ADF<br/>adf-retail-data-dev202610"] -->|System Assigned<br/>Managed Identity| MI["Managed Identity"]

    MI -->|Role Assignment| Role["Storage Blob Data Contributor"]
    Role --> ADLS["ADLS Gen2<br/>strretaildata202610"]

    ADF -->|Linked Service<br/>Managed Identity Auth| ADLS
    ADF -->|Linked Service| SQL["Azure SQL<br/>sqldb-retail-watermark"]

    style MI fill:#fff59d
    style Role fill:#ffcc80
```

---

## 10. Complete Component Checklist (Micro Level)

| Layer              | Component                              | Name / Value                              | Status |
|--------------------|----------------------------------------|-------------------------------------------|--------|
| **Resource Group** | Resource Group                         | `rg-retail-data-dev`                      | ✅     |
| **Storage**        | Storage Account                        | `strretaildata202610`                     | ✅     |
|                    | Containers                             | `raw`, `processed`, `archive`, `config`   | ✅     |
| **ADF**            | Data Factory                           | `adf-retail-data-dev202610`               | ✅     |
|                    | Managed Identity                       | System Assigned                           | ✅     |
| **Linked Service** | ADLS                                   | `LS_ADLS_Retail`                          | ✅     |
|                    | Azure SQL                              | (connection to watermark DB)              | ✅     |
| **Datasets**       | Source Folder                          | `DS_Retail_Raw_Folder`                    | ✅     |
|                    | Parameterized Source                   | `DS_Raw_SourceFiles` + `pFileName`        | ✅     |
|                    | Parameterized Destination              | `DS_Processed_Destination` + `pFileName`  | ✅     |
| **Pipeline**       | Main Pipeline                          | `PL_Retail_File_Ingestion_Dynamic`        | ✅     |
| **Activities**     | Get Metadata                           | `GM_Get_Source_Files`                     | ✅     |
|                    | ForEach                                | `FE_Process_Files`                        | ✅     |
|                    | Lookup                                 | `LK_Check_Watermark`                      | ✅     |
|                    | If Condition                           | `IF_File_Not_Processed`                   | ✅     |
|                    | Copy Data                              | `CP_Copy_File`                            | ✅     |
|                    | SQL Insert Watermark                   | `SQL_Record_Processed_File`               | ⚠️     |
|                    | SQL Log Copy Failure                   | `SQL_Log_Copy_Failure`                    | ✅     |
|                    | SQL Log Watermark Failure              | `SQL_Log_Watermark_Failure` (optional)    | ⚪     |
| **SQL Objects**    | Watermark Table                        | `dbo.FileIngestionWatermark`              | ✅     |
|                    | Error Log Table                        | `dbo.FileIngestionErrorLog`               | ✅     |
|                    | Stored Procedure                       | `dbo.LogFileIngestionFailure`             | ✅     |

---

## 11. Expected Runtime Behavior

| Run Scenario                        | Behavior                                      |
|-------------------------------------|-----------------------------------------------|
| First run (4 files)                 | All 4 files copied + recorded in watermark    |
| Second run (same 4 files)           | All files skipped (FileCount > 0)             |
| New file uploaded                   | Only new file copied + watermark updated      |
| Copy fails                          | Error logged in `FileIngestionErrorLog`       |
| Watermark insert fails              | Error logged (if optional path configured)    |

---

## 12. Quick Reference – Key Expressions

| Activity          | Expression / Setting                                      |
|-------------------|-----------------------------------------------------------|
| ForEach Items     | `@activity('GM_Get_Source_Files').output.childItems`      |
| If Condition      | `@equals(activity('LK_Check_Watermark').output.firstRow.FileCount, 0)` |
| File Name Param   | `@item().name`                                            |
| Dataset File Name | `@dataset().pFileName`                                    |
| Error Message     | `@string(activity('CP_Copy_File').error.message)`         |

---

**This document is ready for GitHub.**  
Just push `ARCHITECTURE.md` to the root of your repository. All Mermaid diagrams will render automatically on GitHub.
```
