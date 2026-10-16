# ADF-STEPBYSTEP-20261015
### Step 1: Create Azure Data Lake
------------------------------------------------

Do this now
1. Open the Azure portal.
2. Create "Resource groups" and open it. "rg-retail-data-dev" or "rg-retail-data-sit" or "rg-retail-data-uat" or other appropriate name.
3. Create "Storage account" and open it. "strretaildata202610" (Follow st-<business>-yyyymmdd) or other appropriate name.
4. Open the storage account > Data storage > Containers. create raw, processed, archive, and config.
5. Open the "raw" container and verify that the retail CSV files are present in their actual folders [Source Files](./SourceFiles/)
6. Storage Structure

## Raw Layer

```text
raw/
├── #customers/
│   ├── customers.csv
│   └── incremental files
├── #products/
│   └── products.csv
├── #stores/
│   └── stores.csv
├── #orders/
│   └── orders.csv
└── other retail files
```
## Create the similar structure in processed container as well.
```text
processed/
├── #customers/
├── #products/
├── #stores/
├── #orders/
```

### Step 2: Create Azure Data Factory

1. Open the [Azure Portal](https://portal.azure.com).
2. Search for **Data factories** and click **Create**.
3. Configure the following details under the **Basics** tab:

| Field | Value |
|---|---|
| Subscription | Azure subscription 1 |
| Resource group | `rg-retail-data-dev` |
| Name | `adf-retail-data-dev202610` |
| Region | India South Central |
| Version | V2 |

4. Review the **Networking** settings:
   - Connect via: Public endpoint
5. Review the **Git configuration**, **Advanced**, and **Tags** tabs as required.
6. Navigate to **Review + create**.
7. Verify that the configuration matches the details above.
8. Click **Create** to deploy the Azure Data Factory resource.
9. Wait for the deployment to complete successfully.
10. Launch "AZURE Data Factory"


### Step 3: Enable Managed Identity

1. Open Azure Data Factory `adf-retail-data-dev202610` in Azure Portal.
2. Navigate to **Settings → Managed identities → System assigned**.
3. Set Status to **On** and click **Save**. if it is already **On** keep it as is and proceed next.

### Step 4: Grant Storage Permissions in Azure data lake to Data factory managed Identity

1. Open the ADLS Gen2 storage account in Azure Portal → **Access control (IAM)**.
2. Assign the **Storage Blob Data Contributor** role to the Data Factory managed identity `adf-retail-data-dev202610`.
3. Complete the role assignment.

   ### Step 5: Create ADLS Gen2 Linked Service

1. Go to **Manage → Linked services → + New** and select **Azure Data Lake Storage Gen2**.
2. Enter name `LS_ADLS_Retail` and select **Managed Identity** authentication.
3. Select your subscription and storage account `stretaildata202610`.
4. Click **Test connection**; after success, click **Create**.

   ### Step 6: Create Source Dataset

1. Create a dataset using **Azure Data Lake Storage Gen2 → DelimitedText**.
2. Set the name to `DS_Retail_Source`, linked service to `LS_ADLS_Retail`, and container path to `raw`.
3. Leave **Directory** and **File name** blank, keep **First row as header** checked, and select **None** for Import schema.
4. Click **OK** to create the dataset.

   ###  Step 7: Create the Destination Dataset
1. In ADF Studio, select Author → + → Dataset.
2. Select Azure Data Lake Storage Gen2 → DelimitedText.
3. Set the name to DS_Retail_Processed, linked service to LS_ADLS_Retail, and container path to processed. Leave Directory and File name blank.
4. Keep First row as header checked, select None for Import schema, and click OK.
This dataset will be used as the destination for files copied from raw to processed.

## Step 8: Recursive File Copy Using Azure Data Factory

1. **Create Datasets:** Create source dataset `DS_Retail_Raw_Folder` pointing to the `raw` container and destination dataset `DS_Retail_Processed` pointing to the `processed` container.
2. **Add Copy Data Activity:** Create pipeline `PL_Retail_File_Ingestion` and add a Copy Data activity named `CP_Copy_Raw_To_Processed`.
3. **Configure Source:** Select `DS_Retail_Raw_Folder`, choose **Wildcard file path**, leave Wildcard folder path blank, set Wildcard file name to `*.csv`, and enable **Recursive**.
4. **Configure Destination:** Select `DS_Retail_Processed` and set **Copy behavior** to **Preserve hierarchy**.
5. **Run and Verify:** Click **Validate**, then **Debug**. Verify that the CSV files are copied from `raw` to `processed` with their folder structure preserved.

##  Step 9:  Dynamic File Copy Using Get Metadata, ForEach and Copy Data

1. **Create Source Folder:** Create `SourceFiles` inside the `Raw` container and upload `Customers.csv`, `Products.csv`, `Stores.csv`, and `Sales.csv`.
2. **Create Destination Folder:** Create `Destination` inside the `Processed` container.
3. **Create Datasets:** Create `DS_Retail_Raw_Folder` for listing source files, `DS_Raw_SourceFiles` for reading individual files, and `DS_Processed_Destination` for writing destination files.
4. **Configure Get Metadata:** Add `GM_Get_Source_Files`, select `DS_Retail_Raw_Folder`, and choose `Child items` to retrieve the file list.
5. **Configure ForEach:** Add `FE_Process_Files` after Get Metadata and set Items to `@activity('GM_Get_Source_Files').output.childItems`.
6. **Configure Copy Data:** Add `CP_Copy_File` inside ForEach. Set the source and destination dataset filename parameters to `@item().name`.
7. **Configure Dataset Parameters:** Add the string parameter `pFileName` to the source and destination datasets. Set each dataset's file name to `@dataset().pFileName`.
8. **Validate and Run:** Validate the pipeline, debug it, and verify that all four CSV files are copied to `Processed/Destination/` with their original filenames.

# <span style="background-color: yellow; color: black;"> Step 10: Azure Data Factory – Incremental File Loading</span>

## 1. Objective

Copy CSV files from the `Raw/SourceFiles` folder to the `Processed/Destination` folder using Azure Data Factory. Use an Azure SQL watermark table to track processed files and avoid copying the same files repeatedly.

## 2. Source and Destination

- **Source container:** `Raw`
- **Source folder:** `SourceFiles`
- **Source files:** `Customers.csv`, `Products.csv`, `Stores.csv`, `Sales.csv`
- **Destination container:** `Processed`
- **Destination folder:** `Destination`
- **Azure SQL database:** `sqldb-retail-watermark`
- **Watermark table:** `dbo.FileIngestionWatermark`
- **Pipeline:** `PL_Retail_File_Ingestion_Dynamic`

## 3. Create the Source and Destination Folders

1. Open the ADLS Gen2 storage account.
2. Create the `SourceFiles` folder inside the `Raw` container.
3. Upload `Customers.csv`, `Products.csv`, `Stores.csv`, and `Sales.csv`.
4. Create the `Destination` folder inside the `Processed` container.

## 4. Create the Watermark Table

Create the following table in the Azure SQL database:

```sql
CREATE TABLE dbo.FileIngestionWatermark
(
    FileName VARCHAR(255) PRIMARY KEY,
    ProcessedOn DATETIME2 NOT NULL
        DEFAULT SYSUTCDATETIME()
);
```

The table stores each successfully processed filename and its processing timestamp.

## 5. Configure Get Metadata

1. Add a Get Metadata activity named `GM_Get_Source_Files`.
2. Select the folder-level dataset pointing to `Raw/SourceFiles`.
3. Under **Field list**, select `Child items`.
4. Connect Get Metadata to the ForEach activity using the Success dependency.

The activity returns the list of files available in the source folder.

## 6. Configure ForEach

1. Add a ForEach activity named `FE_Process_Files`.
2. Under **Settings → Items**, enter:

   ```text
   @activity('GM_Get_Source_Files').output.childItems
   ```

3. Add the Lookup activity inside the ForEach loop.

ForEach processes each file returned by Get Metadata.

## 7. Configure the Lookup Activity

1. Add a Lookup activity named `LK_Check_Watermark`.
2. Select the Azure SQL dataset connected to `sqldb-retail-watermark`.
3. Select **Use query**.
4. Enter the query:

   ```sql
   SELECT COUNT(*) AS FileCount
   FROM dbo.FileIngestionWatermark
   WHERE FileName = '@{item().name}';
   ```

5. Enable **First row only**.
6. Connect Lookup to the If Condition using the Success dependency.

The Lookup checks whether the current filename already exists in the watermark table.

## 8. Configure the If Condition

1. Add an If Condition named `IF_File_Not_Processed`.
2. Set the expression to:

   ```text
   @equals(activity('LK_Check_Watermark').output.firstRow.FileCount, 0)
   ```

3. In the **True** branch, add the Copy Data activity.
4. Leave the **False** branch empty.

- **True:** The file is not recorded, so copy it.
- **False:** The file is already recorded, so skip it.

## 9. Configure the Copy Data Activity

1. Add a Copy Data activity named `CP_Copy_File` inside the True branch.
2. Select the source dataset pointing to `Raw/SourceFiles`.
3. Select the destination dataset pointing to `Processed/Destination`.
4. If the datasets use a `pFileName` parameter, set the source and destination parameter values to:

   ```text
   @item().name
   ```

5. In each parameterized dataset's Connection settings, set the file name to:

   ```text
   @dataset().pFileName
   ```

6. Confirm that both datasets use the correct CSV format.

The Copy Data activity copies the current file using its original filename.

## 10. Update the Watermark Table

After Copy Data succeeds, add an Azure SQL Script or Stored Procedure activity inside the True branch.

1. Name the activity `SQL_Record_Processed_File`.
2. Connect `CP_Copy_File` to this activity using the Success dependency.
3. Insert the current filename and processing timestamp into the watermark table.
4. Use a parameterized SQL operation to pass the filename safely.

Example SQL logic:

```sql
INSERT INTO dbo.FileIngestionWatermark
    (FileName, ProcessedOn)
VALUES
    (@FileName, SYSUTCDATETIME());
```

Configure `@FileName` as a parameter in the SQL activity or stored procedure. Do not hardcode a filename.

**Important:** The watermark insert must execute only after the file copy succeeds.

## 11. Validate and Run the Pipeline

1. Click **Validate all** and resolve any validation errors.
2. Click **Publish all** to publish the pipeline changes.
3. Click **Debug** to run the pipeline.
4. Open the pipeline run output.
5. Verify that Get Metadata, ForEach, Lookup, If Condition, Copy Data, and the SQL watermark activity succeed.

## 12. Verify the Watermark Table

Execute the following query in the Azure SQL Query Editor:

```sql
SELECT *
FROM dbo.FileIngestionWatermark
ORDER BY ProcessedOn DESC;
```

Verify that successfully processed filenames appear in the table.

## 13. Test Incremental Loading

1. Run the pipeline with the four original CSV files.
2. Verify that the files are copied to `Processed/Destination`.
3. Confirm that their filenames are recorded in the watermark table.
4. Run the pipeline again without adding new files.
5. Verify that previously processed files are skipped.
6. Upload a new CSV file into `Raw/SourceFiles`.
7. Run the pipeline again.
8. Verify that the new file is copied and recorded in the watermark table.

## 14. Expected Output

**Source:**

```text
Raw/
└── SourceFiles/
    ├── Customers.csv
    ├── Products.csv
    ├── Stores.csv
    └── Sales.csv
```

**Destination:**

```text
Processed/
└── Destination/
    ├── Customers.csv
    ├── Products.csv
    ├── Stores.csv
    └── Sales.csv
```

**Watermark table:**

| FileName | ProcessedOn |
|---|---|
| Customers.csv | Processing timestamp |
| Products.csv | Processing timestamp |
| Stores.csv | Processing timestamp |
| Sales.csv | Processing timestamp |

## 15. Implementation Status

- Get Metadata: **Succeeded**
- ForEach: **Succeeded**
- Lookup: **Succeeded**
- If Condition: **Succeeded**
- Copy Data: **Succeeded**

**Pending verification:** Confirm that `SQL_Record_Processed_File` is configured and succeeds. Verify that the watermark table contains the processed filenames and that subsequent pipeline runs skip existing files.

A successful pipeline run alone does not confirm that incremental loading is complete. The watermark update and repeat-run test must also succeed.
