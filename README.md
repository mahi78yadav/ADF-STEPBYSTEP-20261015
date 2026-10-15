# ADF-STEPBYSTEP-20261015
Step by step implementation of Azure datafactory project.
------------------------------------------------

Do this now
1. Open the Azure portal.
2. Create "Resource groups" and open it. "rg-retail-data-dev" or "rg-retail-data-sit" or "rg-retail-data-uat" or other appropriate name.
3. Create "Storage account" and open it. "strretaildata202610" (Follow st-<business>-yyyymmdd) or other appropriate name.
4. Open the storage account > Data storage > Containers. create raw, processed, archive, and config.
5. Open the "raw" container and verify that the retail CSV files are present in their actual folders.
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

