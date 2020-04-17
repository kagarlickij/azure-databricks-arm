# Azure Databricks auth Azure Data Lake Storage Gen2

## Ways of access
As per [documentation](https://docs.microsoft.com/en-us/azure/databricks/data/data-sources/azure/azure-datalake-gen2) there are four ways of accessing Azure Data Lake Storage Gen2:  
1. Pass your Azure Active Directory credentials, also known as [credential passthrough](https://docs.microsoft.com/en-us/azure/databricks/administration-guide/access-control/credential-passthrough)  
It requires Azure Databricks [Premium Plan](https://databricks.com/product/azure-pricing) which is 2-3 times more expensive  
And if you want to have this feature for all users you have to use [High concurrency cluster](https://docs.databricks.com/clusters/configure.html#high-concurrency-clusters)  
This scenario is good for per-person clusters and not so good for shared clusters  
Example:
```python
configs = {
  "fs.azure.account.auth.type": "CustomAccessToken",
  "fs.azure.account.custom.token.provider.class": spark.conf.get("spark.databricks.passthrough.adls.gen2.tokenProviderClassName")
}

dbutils.fs.mount(
  source = "wasbs://cont1@kagstorageaccount1.blob.core.windows.net",
  mount_point = "/mnt/cont1",
  extra_configs = configs)

dbutils.fs.ls("/mnt/cont1")

```

2. Mount an Azure Data Lake Storage Gen2 filesystem to DBFS using a service principal and OAuth 2.0  
Service principal allows to get rid of usage storage account access keys, and this allows to revoke access for particular Databricks without any changes on Storage Account side  
This is the most secure way for shared clusters used for production purposes  

3. Use a service principal directly  
The same as previous one, but mount is more convenient in most cases  

4. Use the Azure Data Lake Storage Gen2 storage account access key directly  
Quick and dirty one, so good for test clusters without production data  
If you want to revoke cluster access to storage you'll have to make a change on storage level and update other clusters  
Examples:
```python
configs = {
    "fs.azure.account.key.kagsa2.blob.core.windows.net": "lRm***RBQ=="
}

dbutils.fs.mount(
  source = "wasbs://cont1@kagstorageaccount1.blob.core.windows.net",
  mount_point = "/mnt/cont1",
  extra_configs = configs)

dbutils.fs.ls("/mnt/cont1")
```

..and:
```python
configs = {
    "fs.azure.account.key.kagstorageaccount1.blob.core.windows.net":dbutils.secrets.get(scope = "db-ws-cl1-secscope1", key = "kagstorageaccount1key")
}

dbutils.fs.mount(
  source = "wasbs://cont1@kagstorageaccount1.blob.core.windows.net",
  mount_point = "/mnt/cont1",
  extra_configs = configs)

dbutils.fs.ls("/mnt/cont1")
```

## Mount an Azure Data Lake Storage Gen2 filesystem to DBFS using a service principal and OAuth 2.0 in depth
### Service principal
1. Create Service principal (Azure Portal > Azure Active Directory > App registrations > New registration > type name, e.g. `db-ws-sa1` > keep all other )
2. Type name, e.g. `ws-sa1` and keep all other params as per default
3. Note "Application ID" and "Tenant ID"
4. Create Secret and note it
5. Don't assign any roles!

## Resource group (optional)
1. For this demo purpose I'll create Resource group `db-ws-rg1`

## Azure KeyVault
1. Create Azure KeyVault in Resource group
2. Enter name (e.g. `db-ws-kv1`), resource group to place in and keep all other settings default
3. Create secret named `db-ws-sa1-sec` and enter `db-ws-sa1`'s Secret
4. Set expiration date to date when `db-ws-sa1`'s Secret expires

## Storage account and container
1. Create Storage account with name `dbwssa1` in Resource group
2. Use "Standard" performance, "StorageV2" type, "RA-GRS" replication, "Hot" tier **and** "Enabled" Hierarchical namespace
2. Create container e.g. with name `cont1`
3. In Storage account IAM assign "Storage Blob Data Contributor" role to Service principal  
*ATTENTION*: IAM role can't be assigned on Resource group or Subscription level, only on Storage account  
*ATTENTION* (again): you can assign only "Storage Blob Data *" roles, no other stuff will work:
```
StatusCode=403
StatusDescription=This request is not authorized to perform this operation using this permission.
ErrorCode=AuthorizationPermissionMismatch
ErrorMessage=This request is not authorized to perform this operation using this permission.
```

If you enter wrong creds you'll get different error:
```
Body: {"error":"invalid_client","error_description":"AADSTS7000215: Invalid client secret is provided.\r\nTrace ID: 3b3c6e2a-45e0-450c-adc0-c01579341e00\r\nCorrelation ID: 5ad5d96e-79a6-4a53-acfb-776ede8ea885\r\nTimestamp: 2020-04-09 17:28:46Z","error_codes":[7000215],"timestamp":"2020-04-09 17:28:46Z","trace_id":"3b3c6e2a-45e0-450c-adc0-c01579341e00","correlation_id":"5ad5d96e-79a6-4a53-acfb-776ede8ea885","error_uri":"https://login.microsoftonline.com/error?code=7000215"}
```

## Initialize filesystem
1. Run the following Scala code:
```
spark.conf.set("fs.azure.createRemoteFileSystemDuringInitialization", "true")
dbutils.fs.ls("abfss://<file-system-name>@<storage-account-name>.dfs.core.windows.net/")
spark.conf.set("fs.azure.createRemoteFileSystemDuringInitialization", "false")
```

## Create Databricks workspace and cluster
1. Create Databricks workspace in Resource group
2. Enter name (e.g. `db-ws-cl1`), region and "Standard" for Tier
3. Create new cluster of "Standard" mode and other params as default

## Create an Azure Key Vault-backed secret scope
1. [Azure Key Vault-backed secret scope is more reliable and advanced than Databricks-backed scope](https://docs.microsoft.com/en-gb/azure/databricks/security/secrets/secrets#create-a-secret-in-an-azure-key-vault-backed-scope)
2. Go to `https://<your_azure_databricks_url>#secrets/createScope` (more info [here](https://docs.microsoft.com/en-gb/azure/databricks/security/secrets/secret-scopes#--create-an-azure-key-vault-backed-secret-scope))
3. Enter Scope name, e.g. `db-ws-cl1-secscope1`
4. Select "All users" to manage Principal. If you want "Creator" to be the only who can manage it you have to use Premium tier. This is critical for Databricks-backed scope but not for Azure Key Vault-backed scope
5. You cal check current list of scopes by running `databricks secrets list-scopes`

## Check it!
1. Create Python notebook:
```python
configs = {"fs.azure.account.auth.type": "OAuth",
       "fs.azure.account.oauth.provider.type": "org.apache.hadoop.fs.azurebfs.oauth2.ClientCredsTokenProvider",
       "fs.azure.account.oauth2.client.id": "c76b0be4-bbf8-42f7-8541-37f8df1f8914", # Service principal's Application ID
       "fs.azure.account.oauth2.client.secret": dbutils.secrets.get(scope = "db-ws-cl1-secscope1", key = "db-ws-sa1-sec"), # Azure Key Vault-backed secret scope's name and Vault's secret name
       "fs.azure.account.oauth2.client.endpoint": "https://login.microsoftonline.com/a9f9369c-7127-4e16-b301-8de8e28b309c/oauth2/token", # Service principal's Tenant ID
       "fs.azure.createRemoteFileSystemDuringInitialization": "true"}

dbutils.fs.mount(
source = "abfss://cont1@kagstorageaccount1.dfs.core.windows.net",
mount_point = "/mnt/cont1",
extra_configs = configs)

dbutils.fs.ls("/mnt/cont1")
```

2. Unmount: `dbutils.fs.unmount("/mnt/cont1")`
