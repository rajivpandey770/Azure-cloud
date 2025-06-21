

Solution Proposal: Tiered Storage Architecture for Cost Optimization
The most effective strategy to address this challenge is to implement a tiered storage architecture. This approach leverages the strengths of different 
Azure services to store data based on its access frequency, significantly reducing costs while maintaining performance and data availability.

Recent, frequently accessed data ("hot data") will remain in Azure Cosmos DB for fast reads, while older, infrequently accessed data ("cold data") will be 
archived to the much more cost-effective Azure Blob Storage.

Proposed Architecture
This solution involves three core components:

Azure Cosmos DB (Hot Tier): Continues to store billing records for the most recent three months. This ensures low-latency access for the read-heavy, current 
data as required.
Azure Blob Storage (Cold Tier): Used to archive billing records older than three months. We will use the Cool access tier, which is optimized for storing 
data that is infrequently accessed but must be immediately available when requested, aligning with the "response time in the order of seconds" requirement.
Azure Functions (Logic Layer): Two serverless functions will automate the data lifecycle and provide a unified data access layer.
Archival Function: A time-triggered function that periodically moves data from Cosmos DB to Blob Storage.
Read-Through API Function: The existing API endpoint, modified to intelligently fetch data from either Cosmos DB or Blob Storage.
Detailed Implementation Plan
Here is a step-by-step plan to implement this solution seamlessly with no downtime.

Step 1: Set Up Azure Blob Storage
Create a Storage Account: Provision a new Azure Storage Account in the same region as your Cosmos DB instance to minimize data transfer latency and costs.
Create a Container: Inside the storage account, create a new blob container (e.g., billing-records-archive).
Set Access Tier: Configure the default access tier for the container to Cool. This is crucial for balancing storage cost and retrieval time. The Archive 
tier is not suitable here due to its multi-hour retrieval latency.
Step 2: Update the API Read Logic
The key to meeting the "No Changes to API Contracts" and "No Downtime" requirements is to first update your existing read API (likely an Azure Function) to
be aware of both storage locations.

The new logic will be as follows:

The API receives a request for a specific billing record ID.
It first attempts to retrieve the record from Azure Cosmos DB.
If the record is found (i.e., it's recent data), it is returned to the client immediately.
If the record is not found in Cosmos DB (which implies it may be an older, archived record), the function then attempts to retrieve it from Azure 
Blob Storage. The blob can be named after the record ID (e.g., {recordId}.json) for direct lookups.
If found in Blob Storage, the data is returned. If it's not found in either location, a 404 Not Found error is returned.
This "read-through cache" pattern is transparent to the end-user and requires no changes to the API contract.

Step 3: Develop the Data Archival Function
Create a new time-triggered Azure Function to automate the archival process.

Trigger: Schedule the function to run periodically (e.g., daily or weekly).
Logic:
The function queries Cosmos DB for records with a creation date older than three months.
It iterates through the results. For each record: a. It writes the record's data as a new JSON blob to the Azure Blob Storage container created in

Step 1. b. Upon successful confirmation of the blob write, it deletes the original record from Cosmos DB. This two-step process ensures no data is lost 
during migration.
Incorporate robust error handling and logging (e.g., using Application Insights) to manage any potential failures during the move.

Step 4: Phased, Zero-Downtime Deployment
Deploy Updated API: Deploy the new version of the read API (from Step 2). At this stage, since all data is still in Cosmos DB, the API's behavior will be 
unchanged.
Perform Initial Backfill: Manually trigger the archival function (from Step 3) to begin moving all existing records older than three months to Blob Storage. 
This process runs in the background. As records are moved, the updated read API will automatically start fetching them from their new location in 
Blob Storage 
without any service interruption.
Enable Scheduled Archiving: Once the initial backfill is complete, enable the time-trigger schedule for the archival function. It will now run automatically 
to archive data as it ages, ensuring the system remains optimized.
Meeting the Requirements
This solution is tailored to meet every requirement outlined in the challenge:

Cost Optimization:

The primary driver of cost reduction is moving the bulk of the data (over 2 million records, potentially >500 GB) from Cosmos DB's more expensive storage to Blob Storage's Cool tier, which is up to 90% cheaper for storage.
Reducing the total data in Cosmos DB will also lower the provisioned Request Units (RUs) needed, further cutting operational costs.
Simplicity & Ease of Implementation:

The solution uses standard, fully-managed Azure services that are designed to work together. The logic for the functions is straightforward and easy to 
maintain.
No Data Loss & No Downtime:

The phased deployment ensures the read API is always available.
The "write-then-delete" archival logic guarantees that data is safely persisted in Blob Storage before being removed from Cosmos DB, preventing data loss.
No Changes to API Contracts:

The logic to handle the two data tiers is encapsulated entirely within the backend service. The external API signature and the data format of the responses 
remain identical.
Access Latency:

Recent records from Cosmos DB will be served with millisecond latency.
Older records from Blob Storage (Cool tier) will be served within seconds, which aligns perfectly with the specified access latency requirement for old 
records.


