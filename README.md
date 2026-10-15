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
