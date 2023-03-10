# CockroachDB Change Data Capture (CDC) Integration with Synapse via Azure Blob Storage

CockroachDB is an Distributed SQL OLTP database which is perfect for supporting critical line of business applications that have scale and resiliency requirements. However, an organization’s data architecture will often contain many tools, including something for transforming, shaping & analyzing data to help inform better business decisions. This means that a line of business database like CockroachDB must be able to integrate with such systems to help customer teams achieve those outcomes for business. 

Microsoft Azure is one of the largest cloud providers and has many popular data analytics tools in its portfolio. In this tutorial we will look at how we can send data using change data capture in CockroachDB to one of those services, Azure Synapse. 

Azure Synapse Analytics is a limitless analytics service that brings together data integration, enterprise data warehousing, and big data analytics. It gives you the freedom to query data on your terms, using either serverless or dedicated options—at scale. Azure Synapse brings these worlds together with a unified experience to ingest, explore, prepare, transform, manage, and serve data for immediate self service BI and machine learning needs. 

If you would like to follow along these are the prerequisites:

- [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
- A CockroachDB Cluster running in Azure!
- My Cockroach Cluster is running in Kubernetes so I needed [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Cockroach Binary](https://www.cockroachlabs.com/docs/stable/install-cockroachdb-mac.html)

## Step 1: Prepare Your Variables

To standardise the code in further steps a number of variables should be set.

```
loc1="uksouth"
clus1="crdb-aks-uksouth"
rg="mb-crdb-aks-multi-region"
loc1_cdc_storage_account="crdbuksouthcdc"
loc1_cdc_storagecontainer="uksouth-cdc"
```
## Step 2: Create an Azure Storage Account and Container

CockroachDB's [Change Data Capture](https://www.cockroachlabs.com/docs/stable/change-data-capture-overview.html) feature has the ability to send row level changes to object storage, in this case Azure blob storage. Create a storage account first.

```
az storage account create \
  --name $loc1_cdc_storage_account \
  --resource-group $rg \
  --location $loc1 \
  --sku Standard_RAGRS \
  --kind StorageV2
```
Then a storage container within that storage account.

```
az storage container create \
 -n $loc1_cdc_storagecontainer \
 --fail-on-exist \
 --account-name $loc1_cdc_storage_account
```

Now we’ve a destination to send the data to from CockroachDB.

## Step 3: Upgrade Azure Blob Storage with Azure Data Lake Storage Gen2 capabilities Validate your storage account by using the following command.

To take advantage of Synapse the storage account needs to be upgraded with Azure Data Lake Storage Gen2 capabilities. Run the command below to upgrade the storage account.

```
az storage account hns-migration start --type validation -n $loc1_cdc_storage_account -g $rg
```

If the validation succeeds, the process completes and no errors appear.

If validation fails, a validation error will appear in the console. For example, the error `(IncompatibleValuesForAccountProperties) Values for account properties are incompatible: Versioning Enabled` indicates that an incompatible feature (Versioning) is enabled on the account. In this case, you would disable the feature and then start the validation process again.

In some cases, the path to a file named error.json appears in the console. You can open that file to determine why the account did not pass the validation step.

The following JSON indicates that an incompatible feature is enabled on the account. In this case, you would disable the feature and then start the validation process again.


Example Output:
```
{
 "startTime": "2021-08-04T18:40:31.8465320Z",
 "id": "45c84a6d-6746-4142-8130-5ae9cfe013a0",
 "incompatibleFeatures": [
     "Blob Delete Retention Enabled"
 ],
 "blobValidationErrors": [],
 "scannedBlobCount": 0,
 "invalidBlobCount": 0,
 "endTime": "2021-08-04T18:40:34.9371480Z"
}
```

After your account has been successfully validated, start the upgrade by running the following command.

```
az storage account hns-migration start --type upgrade -n $loc1_cdc_storage_account -g $rg
```

## Step 4: Enable Change Data Capture

[Change data capture](https://www.cockroachlabs.com/docs/stable/change-data-capture-overview.html) (CDC) provides efficient, distributed, row-level changefeeds into a configurable sink for downstream processing such as reporting, caching, or full-text indexing. In this particular case, we will be configuring the changefeed (CDC) to send data to the above configured storage account in Azure. This storage account has now been configured as a data lake.

### Retrieve the storage account keys from Azure

In order to configure CDC we need to obtain the account keys for our storage account which will allow us to authenticate to it.

Run the following command to display your account key.

```
az storage account keys list -g $rg -n $loc1_cdc_storage_account --query "[0].value" -o tsv
```

The output will look like the example below. As it contains some unsupported characters to use within a URL we need to URL-Encode the key for a future step. There are a number of ways you can URL-Encode a string but I found the easiest way was [here](https://www.urlencoder.org/). 

Example Output:
```
fot2mB/dHfvPm1PwDvjToArXxQcgDMITR8LQ3fDumqZVLpSUhY7UA7gDuRCsXVk88SMcikpd1AHj+ASt0jhDQw==
```
Below is an example of the URL Encoded account.
```
fot2mB%2FdHfvPm1PwDvjToArXxQcgDMITR8LQ3fDumqZVLpSUhY7UA7gDuRCsXVk88SMcikpd1AHj%2BASt0jhDQw%3D%3D
```

### Configure CockroachDB CDC in Kubernetes

In this example CockroachDB is running in Azure Kubernetes Service. If you would like the instructions on how to build a multi region AKS solution to run CockroachDB then please take a look at this [repo](https://github.com/mbookham7/mb-crdb-multi-region-aks).

To connect to the cluster we deploy a pod running the CockroachDB binary with the required certificate. First create the pod using the command below.

```
kubectl create -f https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/multiregion/client-secure.yaml --namespace $loc1
```
Now the pod is running, connect into the pod running the `cockroach sql` command.

```
kubectl exec -it cockroachdb-client-secure -n $loc1 -- ./cockroach sql --certs-dir=/cockroach-certs --host=cockroachdb-public
```

Now you are connected to the pod with the SQL Client we can configure CDC. First, update the cluster settings `kv.rangefeed.enable` to be `true`.

```
SET CLUSTER SETTING kv.rangefeed.enabled = true;
```
Now that we have enabled the feature we can create our first change feed. This change feed will send all changes to the `rides` table to and Azure blob which is also an Azure Datalake.
```
CREATE CHANGEFEED FOR TABLE rides
    INTO 'azure://uksouth-cdc?AZURE_ACCOUNT_NAME=crdbuksouthcdc&AZURE_ACCOUNT_KEY=fot2mB%2FdHfvPm1PwDvjToArXxQcgDMITR8LQ3fDumqZVLpSUhY7UA7gDuRCsXVk88SMcikpd1AHj%2BASt0jhDQw%3D%3D'
    WITH
        updated;
```
To generate some changes to the rides table we are for a second time going to exec int the secure pod we created earlier. However, this time we are going to use `cockroach workload movr` perform `INSERT`, `UPDATE` and `DELETE` actions against the table. Remote into the pods with the command below.

```
kubectl exec -it cockroachdb-client-secure -n $loc1 --context $clus1 -- sh
```
The command below will create the schema required for the workload. In this example there is already an SQL user called `craig` with the password `cockroach` and a new database called `movr`. You will need to create this database and user or change the credentials for a user in your CockroachDB cluster.

```
cockroach workload init movr 'postgresql://craig:cockroach@cockroachdb-public:26257/movr?sslmode=verify-full&sslrootcert=/cockroach-certs/ca.crt'
```

Once the database and schema are created we can generate some load. The command below will do this.
```
cockroach workload run movr --tolerate-errors --duration=99999m 'postgresql://craig:cockroach@cockroachdb-public:26257/movr?sslmode=verify-full&sslrootcert=/cockroach-certs/ca.crt'
```

## Step 5: Run a query against Azure Datalake Gen2 with Synapse

In the Azure Portal create and Synapse workspace. In the search bar at the top and search for Synapse click on the add button and complete wizard that is presented.

![Create Synapse Workspace](./images/synapse-img4.png)

Once created open Open Synapse Studio form the Getting Started tile in the main window.

![Open Synapse Studio](./images/synapse-img2.png)

Once opened it needs to be connected to external storage. This can be done by clicking in `Data` on the left hand menu and by using the `+`. Select `Connect to external data` than complete the wizard to connect the Azure Datalake.

![Connect External Data](./images/synapse-img3.png)

Synapse is now configured and we can run an example SQL Query. The query below looks at all the available files in the Datalake and gives us a revenue total per City.

```
SELECT
    /* --> place the keys that you see in JSON documents in the WITH clause:
    , JSON_VALUE (line, '$.key1') AS header1
    , JSON_VALUE (line, '$.key2') AS header2
    */
    JSON_VALUE (line, '$.after."city"') as City,
    Sum(CAST(JSON_VALUE (line, '$.after."revenue"') as float)) as Total_Revenue
FROM
    OPENROWSET(
        BULK 'https://crdbuksouthcdc.dfs.core.windows.net/uksouth-cdc/2022-12-12/*.ndjson',
        FORMAT = 'csv',
        -- FIELDQUOTE and FIELDTERMINATOR are set to 0x0b as we do not expect to find it in the file.
        FIELDQUOTE = '0x0b',
        FIELDTERMINATOR ='0x0b'
    )
    WITH (
        line varchar(max)
    ) AS [result]
Group by JSON_VALUE (line, '$.after."city"')
```

You are able to see the totals in the image below represented by a bar chart.

![Synapse Quey Result](./images/synapse-img1.png)

# Final Thoughts

In conclusion, combining analytical tools like Azure Synapse with transactional databases like CockroachDB can have a positive impact for many companies. Real-time data replication is made possible via CockroachDB's Change Data Capture (CDC) capability, making it simple to interface with analytical software like Azure Synapse. While guaranteeing that transactional data is highly available and consistent, this integration allows enterprises to obtain insights and make data-driven choices in real-time. The scalability, resilience, and consistency properties of CockroachDB help ensure that the data is correct and dependable despite heavy loads and failures. CockroachDB's data locality feature also facilitates global deployments and low-latency data access. Generally speaking, firms can gain a competitive edge by using data-driven decisions by combining CockroachDB with analytical tools like Azure Synapse.

Code here.
https://github.com/mbookham7/mb-crdb-cdc-dlgen2-synapse