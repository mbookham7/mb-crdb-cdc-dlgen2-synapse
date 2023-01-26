# CockroachDB Change Data Capture (CDC) Integration with Synapse via Azure Blob Storage

CockroachDB is an OLTP database with is perfect for supporting critical line of business applications. However, a clients architecture will often contain many tools for analyzing data to help inform better business decisions. This means that a line of business database like CockroachDB must be able integrate with other systems to help perform this analysis. Previously, many data analytical tool would have been out of reach to many businesses due to cost but the raise in popularity of the hyper-scalers many more tools for analyzing data have become within the reach. 

Microsoft Azure is the second most popular cloud provider and has many data analytics tools. In this tutorial we will look at how we can expose data form CockroachDB, which is often used to store data for transactional systems. This data would then be passed to Azure Synapse. 

Azure Synapse Analytics is a limitless analytics service that brings together data integration, enterprise data warehousing, and big data analytics. It gives you the freedom to query data on your terms, using either serverless or dedicated options—at scale. Azure Synapse brings these worlds together with a unified experience to ingest, explore, prepare, transform, manage, and serve data for immediate BI and machine learning needs.

If you would like to follow along then you will need to get started.

List of prerequisites.

- [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)
- A CockroachDB Cluster running in Azure!
- My Cockroach Cluster is running in Kubernetes so I needed [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Cockroach Binary](https://www.cockroachlabs.com/docs/stable/install-cockroachdb-mac.html)

## Step 1: Prepare Your Variables

To standardize the code in further steps a number of variables should be set.

```
loc1="uksouth"
clus1="crdb-aks-uksouth"
rg="mb-crdb-aks-multi-region"
loc1_cdc_storage_account="crdbuksouthcdc"
loc1_cdc_storagecontainer="uksouth-cdc"
```
## Step 2: Create an Azure Storage Account and Container

CockroachDB's Change Data Capture feature has the ability to send row level changes to object storage, in this case Azure blob storage. 

```
az storage account create \
  --name $loc1_cdc_storage_account \
  --resource-group $rg \
  --location $loc1 \
  --sku Standard_RAGRS \
  --kind StorageV2
```

```
az storage container create \
 -n $loc1_cdc_storagecontainer \
 --fail-on-exist \
 --account-name $loc1_cdc_storage_account
```

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

Change data capture (CDC) provides efficient, distributed, row-level changefeeds into a configurable sink for downstream processing such as reporting, caching, or full-text indexing. In the case of this demo we will be configuring CockroachDB Change Feeds (CDC) to sink data with our storage account in Azure. This storage account has now been configured as a data lake.

### Retrieve the storage account keys from Azure

In order to to configure CDC we need to obtain the storage account keys for our storage account this will allow us to authenticate successfully.

Run the following command to display you account key.

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

In this example CockroacDB is running in Azure Kubernetes Service. 

```
kubectl create -f https://raw.githubusercontent.com/cockroachdb/cockroach/master/cloud/kubernetes/multiregion/client-secure.yaml --namespace $loc1
```

```
kubectl exec -it cockroachdb-client-secure -n $loc1 -- ./cockroach sql --certs-dir=/cockroach-certs --host=cockroachdb-public
```

```
SET CLUSTER SETTING kv.rangefeed.enabled = true;
```
```
CREATE CHANGEFEED FOR TABLE rides
    INTO 'azure://uksouth-cdc?AZURE_ACCOUNT_NAME=crdbuksouthcdc&AZURE_ACCOUNT_KEY=fot2mB%2FdHfvPm1PwDvjToArXxQcgDMITR8LQ3fDumqZVLpSUhY7UA7gDuRCsXVk88SMcikpd1AHj%2BASt0jhDQw%3D%3D'
    WITH
        updated;
```

```
kubectl exec -it cockroachdb-client-secure -n $loc1 --context $clus1 -- sh
```

```
cockroach workload init movr 'postgresql://craig:cockroach@cockroachdb-public:26257/movr?sslmode=verify-full&sslrootcert=/cockroach-certs/ca.crt'

cockroach workload run movr --tolerate-errors --duration=99999m 'postgresql://craig:cockroach@cockroachdb-public:26257/movr?sslmode=verify-full&sslrootcert=/cockroach-certs/ca.crt'
```

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