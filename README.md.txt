
                                         Cost Optimization Challenge
                                         ===========================

// Terraform template for Blob Lifecycle Rules
resource "azurerm_storage_account" "example" {
  name                     = "examplestorageacct"
  resource_group_name      = azurerm_resource_group.example.name
  location                 = azurerm_resource_group.example.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

resource "azurerm_storage_management_policy" "lifecycle" {
  storage_account_id = azurerm_storage_account.example.id

  rule {
    name    = "transition-hot-to-archive"
    enabled = true

    filters {
      blob_types   = ["blockBlob"]
      prefix_match = ["logs/"]
    }

    actions {
      base_blob {
        tier_to_cool {
          days_after_modification_greater_than = 30
        }

        tier_to_archive {
          days_after_modification_greater_than = 90
        }
      }
    }
  }
}

// Azure Data Factory JSON Pipeline to Move Cosmos → SQL → Blob (simplified structure)
// Save this under ADF pipeline definition JSON
{
  "name": "TieredStoragePipeline",
  "properties": {
    "activities": [
      {
        "name": "LookupOldCosmosData",
        "type": "Lookup",
        "dependsOn": [],
        "typeProperties": {
          "source": {
            "type": "CosmosDbSqlApiSource",
            "sqlQuery": "SELECT * FROM c WHERE c.timestamp < GetCurrentTimestamp() - 30"
          },
          "dataset": { "referenceName": "CosmosDBDataset", "type": "DatasetReference" }
        }
      },
      {
        "name": "CopyToSQL",
        "type": "Copy",
        "dependsOn": [
          { "activity": "LookupOldCosmosData", "dependencyConditions": ["Succeeded"] }
        ],
        "typeProperties": {
          "source": { "type": "CosmosDbSqlApiSource" },
          "sink": { "type": "SqlSink" }
        },
        "inputs": [ { "referenceName": "CosmosDBDataset", "type": "DatasetReference" } ],
        "outputs": [ { "referenceName": "SQLDataset", "type": "DatasetReference" } ]
      },
      {
        "name": "CopyToBlob",
        "type": "Copy",
        "dependsOn": [
          { "activity": "CopyToSQL", "dependencyConditions": ["Succeeded"] }
        ],
        "typeProperties": {
          "source": { "type": "SqlSource" },
          "sink": { "type": "BlobSink" }
        },
        "inputs": [ { "referenceName": "SQLDataset", "type": "DatasetReference" } ],
        "outputs": [ { "referenceName": "BlobDataset", "type": "DatasetReference" } ]
      }
    ],
    "annotations": []
  }
}

// Proposed Architecture Implementation Plan
// Cosmos DB (Hot Tier), Blob Storage (Cold Tier), Azure Functions (Archival & Read API)
// Step-by-step strategy for zero-downtime and cost-effective data lifecycle

/*
Step 1: Set Up Azure Blob Storage
- Create a Storage Account in the same region as Cosmos DB.
- Create a blob container (e.g., billing-records-archive).
- Set default access tier to Cool.

Step 2: Update Read API (Azure Function)
- Check Cosmos DB for record ID.
- If found, return.
- If not found, check Blob Storage (using {recordId}.json).
- If found, return; else 404.

Step 3: Develop Archival Function
- Trigger: Timer (daily or weekly).
- Query Cosmos DB for records older than 3 months.
- For each:
  a. Write to Blob Storage as JSON.
  b. On success, delete from Cosmos DB.
- Add Application Insights logging.

Step 4: Phased Deployment
- Deploy updated API first.
- Run archival function manually to backfill.
- Once done, enable timer-based archiving.
*/

