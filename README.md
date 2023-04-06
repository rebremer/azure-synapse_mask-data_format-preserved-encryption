## Azure Synapse, mask data using format preserved encryption (fpe)
Azure Synapse capability to mask PII data using [format preserved encryption](https://en.wikipedia.org/wiki/Format-preserving_encryption) at scale. In this, the following is taken care for:
 
- **Masking functionality**: Extending on Python libraries like [ff3]((https://github.com/mysto/python-fpe), fpe can be achieved for IDs, names, phone numbers and emails (similar what can be done with [k2view masking tools](https://www.k2view.com/solutions/data-masking-tools/)). For example:
  - Bremer => 6paCYa
  - Bremer & Sons!, LTD. => OsH0*VlF(dsIGHXkZ4dK
  - 06-23112312 => 48-78322271
  - bremersons@hotmail.com => l-YLt53VGs@VDIdçQ4.com
- **Security**: Synapse Analytics workspace that used the following capabilities:
  - Network: Private endpoints to connect to storage Account and Azure SQL (public access can be disabled)
  - Authentication: Managed Identity to authenticate to Storage account, Azure SQL and Azure Key Vault in which the secrets are stored that are used by ff3 for encryption, 
  - Authorization: RBAC for authorization to Azure Storage, Azure SQL and Azure Key Vault
  - Data exfiltration protection enabled. Using [Synapse data exfiltration protection capability](https://learn.microsoft.com/en-us/azure/synapse-analytics/security/workspace-data-exfiltration-protection), it can be prevented that data leaves the Rabobank tenant by a malicious insider
- **Performance**: A test is created as follows:
  - Size: dataset with 1M records (total size of dataset 250MB). In this dataset, 6 columns are to be encrypted (id, firstname, lastname, companyname, email, phone) 
  - Compute: Medium sized Spark pool with 2 executors (VMs) and 8 vcores (threads) per executor (16 vcores/threads in total) 
  - => Able to mask the data and write it to a storage account in 1m45. It is expected that it is possible to scale solution further for larger datasets (either by scaling up with more vcores per executor or scaling out with more executors or just more spark pools)
- **Orchestration**:
  - Synapse pipelines can orchestrate the process end to end. That is, data can be fetched from on-premises using over 100 different connectors, staged to Azure Storage, masked and then sent back to lower environment for tested.

See high level architecture below:

![Architecture](Images/0_Architecture.jpg)

Following steps need to done:

- Deploy resources
- Configure resources
- Test Synapse notebook 
- (Optional) Create Synapse pipeline ingested and propagating data to Azure SQL

### 1. Deploy resources

The following resources need to be deployed:

- Azure Synapse Analytics workspace with data exfiltration prevention enabled: Make sure that a primary storage account is created. Make also sure that Synapse is deployed with 1) Managed VNET enabled, 2) has a private endpoint to the storage account and 3) allow outbound traffic only to approved targets, see also screenshot below:

![Synapse_networking](Images/1_Synapse_networking.png)

- Azure Key vault: This Keyvault will be used to store the secrets that are used by Synapse to create the HMACs in ```Synapse/mask_data_fpe_prefixcipher.ipynb``` and encryption in ```Synapse/mask_data_fpe_ff3.ipynb```

### 2. Configure resources
#### 2a. Configure resources - Storage

- Primary storage account of Azure Synapse: Create a new Filesystem called ```bronze``` and ```gold```. Then upload csv file in ```Data\SalesLT.Customer.txt```. In case you want to do a larger dataset, see [this](https://testhmacmaskstor.blob.core.windows.net/bronze/SalesLT.Customer_1M.txt?sp=r&st=2023-04-06T09:04:43Z&se=2024-04-01T17:04:43Z&spr=https&sv=2021-12-02&sr=b&sig=zYCOdxO40pWoTKBDfGpC%2FsR6ixpUiCneXGHQJSNlxuQ%3D) of 250MB and 1M records

#### 2b. Configure resources - Azure Key Vault

- Azure Key vault - secrets: Create a secret called ```fpekey``` and ```fpetweak```. Make sure that Hexidecimal values are added for both secrets. In case Azure Key vault was deployed with public access enabled (in order to be able to create secrets via Azure Portal), it is now not needed anymore and public access can be disabled (since private link connection will be created between Synapse and Azure Key vault in 2c1.)

- Azure Key vault - access control: Create a secret called ```fpekey``` and ```fpetweak```. Make sure that Hexidecimal values are added for both secrets. Also, create 5 secrets ```secret10```, ```secret100``````secret1000``````secret10000```and ```secret100000```, these can be filled with any value.
.Make sure that in the access policies of the Azure Key Vault the Synapse Managed Identity had get access to secret, see also screenshot below.

![AKV_Synapse_access](Images/2b_AKV_access_policies.png)

#### 2c1. Configure resources - Synapse - Azure Key Vault

- Azure Synapse Analytics - Private link to Azure Key Vault: Create a private endpoint from the Azure Synapse Workspace managed VNET and your key vault. The request is initiated from Synapse and needs to be approved in the AKV networking. See also screenshot below in which private endpoint is approved. After this step, public access can

![Synapse_private_endpoint_keyvault](Images/2c1_1_Synapse_private_endpoint_keyvault.png)

- Azure Synapse Analytics - Linked Service link to Azure Key Vault: Create a linked service from the Azure Synapse Workspace and your key vault, see screenshot below.

![Synapse_linked_services_keyvault](Images/2c1_2_Synapse_lined_service_keyvault.png)

#### 2c2. Configure resources - Synapse - Spark

- Azure Synapse Analytics - Spark Cluster: Create a Spark cluster that is Medium size, has 3 to 10 nodes and can be scaled to 2 to 3 executors, see also screenshot below.

![Synapse_Spark_cluster_config](Images/2c2_1_Synapse_spark_cluster_config.png)

- Azure Synapse Analytics - Library upload: Notebook ```Synapse/mask_data_fpe_ff3.ipynb``` uses [ff3](https://github.com/mysto/python-fpe) to encryption. Since Azure Synapse Analytics is created with data exfiltration enabled, it can be installed using ```pip install ```, since that requires outbound connectivity. Download the pycryptodome wheel [here](https://files.pythonhosted.org/packages/14/58/77278d7a078241b55b515f6073b90108125fb0d197b384a0f372c5f61c80/pycryptodome-3.17-cp35-abi3-manylinux_2_17_x86_64.manylinux2014_x86_64.whl) and ff2 wheel [here](https://files.pythonhosted.org/packages/3a/c1/3550f1b97d6eedb2117521a149f379bb0d92cbb02e242110bb174f12c9a2/ff3-1.0.1-py3-none-any.whl). Then upload the wheels to the Spark cluster,so screenshot below.

![Synapse_Spark_library_upload](Images/2c2_2_Synapse_spark_cluster_libraries.png)

#### 2c3. Configure resources - Synapse - Notebook

- Azure Synapse Analytics - Notebooks upload: Upload the notebooks ```Synapse/mask_data_fpe_prefixcipher.ipynb``` and ```Synapse/mask_data_fpe_ff3.ipynb``` to your Azure Synapse Analytics Workspace. Make sure that in the notebooks, the value of the storage account, filesystem, key vault name and keyvault linked services are substituted.

- Azure Synapse Analytics - Notebooks - Spark session: Open Spark session of notebook ```Synapse/mask_data_fpe_prefixcipher.ipynb```, make sure you choose more than 2 executor and run it using a Managed Identity, see also screenshot below.

![Synapse_Spark_notebook_session](Images/2c3_1_Synapse_spark_notebook_session.png)

### 3. Run notebook

After all resources are deployed and configured, notebook can be run. Notebook ```Synapse/mask_data_fpe_prefixcipher.ipynb``` contains functionality to mask numeric values, alpanumeric values, phone numbers and email addresses, see functionality below.

```
000001 => 359228
Bremer => 6paCYa
Bremer & Sons!, LTD. => OsH0*VlF(dsIGHXkZ4dK
06-23112312 => 48-78322271
bremersons@hotmail.com => l-YLt53VGs@VDIdçQ4.com
```

 In case the 1M dataset is used and 6 columns are encrypted, processing takes around 2 minutes. This can easily be scaled by using 1) scaling up by using more vcores (from medium to large), scaling out by using more executors or just create a 2nd Spark pool. See also screenshot below.

![Synapse_Spark_notebook_session](Images/3_notebook_run.png)

#### 4. (Optional) Create Synapse pipeline ingested and propagating data to Azure SQL

Pipeline can be found in  ```Synapse/synapse_pipeline.json```