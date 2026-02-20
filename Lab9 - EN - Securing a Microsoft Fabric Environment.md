# Introduction & General Architecture

The goal of this step-by-step guide is to propose a working environment within Microsoft Fabric using the most restrictive access security measures. We will discover how to provide access to resources only from an organization's internal network, enabling the development of a Spark Notebook within Microsoft Fabric that needs to access data stored in an Azure Data Lake. The exchange between the user and the platform is private, and the exchange between Microsoft Azure and Microsoft Fabric is also private.

Step by step, we will deploy an environment that meets the following constraints:
  - **Use of Private Link**: Communications with Power BI and/or Microsoft Fabric are done via the Azure network, not via the Internet. This allows control over the entry point for users and interactions with data flowing to and from Microsoft Fabric.
  - **Disabling public Internet access**: Not only is access done via internal Azure services, but access to the portal can no longer be made via the service's public IP.
  - **Disabling public access to Azure resources**: By disabling public network access to Key Vault (container for identification keys) or ADLS Gen2 (container for data), access from anywhere is also prohibited.
  - **Use of private endpoints for Azure services**: An approved entry point is defined for accessing Azure resources, strictly reserved for a service or usage.
  - **Consumption of Azure services via Microsoft Fabric privately**: Once portal access is restricted and exchanges between the service and resources are controlled, users and developers will operate in a segregated environment.

**Notes & Limitations:**
  - Managed Private Network: When an administrator enables Private Link, launching a Spark notebook will automatically use a Managed Virtual Network to instantiate Spark compute resources. This cluster is not accessible from the public network. Instantiating resources within this Managed Virtual Network takes longer than public usage; session startup increases from about 30 seconds to 4 minutes.
     
  - Starter Pools: It is possible to launch a Spark Notebook without worrying about the compute resource configuration. In this case, the so-called _Starter Pool_ will be instantiated with a standard but non-optimal size for computations. When resources are privatized, this Starter Pool is no longer available, so you must create one with a defined size directly within the Workspace to run your Notebook.

Once these prerequisites are validated, we will see how to use Microsoft Fabric to transform our data and evolve it within our environment.

**General Architecture:**

<img src="https://github.com/user-attachments/assets/ab850c9f-8417-478f-bf26-a90a1baad66a">

Using a VM in Azure is not necessary.

# Prerequisites

For this step-by-step, we need:
  - **Roles**:
    - In Azure, rights to create resources in a test subscription or resource group
    - In Microsoft Fabric, administrator rights on the environment
  - **Resources**:
    - An **Azure** environment with the ability to create:
      - A Microsoft Fabric capacity
      - A Virtual Network and Bastion
      - A Virtual Machine
      - A Private Link Service
      - A Key Vault
      - An Azure Data Lake Storage Gen 2
    - A **Microsoft Fabric** environment with administrator role:
      - To enable/disable features in the portal
      - Ability to create a workspace containing a Notebook

# Step-by-step: Part 1: Setting up private portal access

Microsoft documentation exists for creating a Private Link for Power BI and Microsoft Fabric. The following details are based on: https://learn.microsoft.com/en-us/fabric/security/security-private-links-use

## 1. Enabling features in the Power BI portal:

_In this part, we will enable the Private Link feature to access resources via the Azure Backbone._

In the Power BI admin portal, enable the feature to use Private Links: Go to the admin portal, search for advanced network settings, and enable the feature:

<img src="https://github.com/user-attachments/assets/638cbb9f-7586-41c0-a6fb-2809fddb171d" width="300">
<img src="https://github.com/user-attachments/assets/4fcba74a-4e9e-4aff-b505-18f8c48cb04f" width="400">

Also, collect your Microsoft Fabric tenant ID: Click "?" in the header > About Power BI > and copy the GUID after _ctid=_. **Note this ID for later.**

<img src="https://github.com/user-attachments/assets/eda6e8f3-7528-4452-acbd-5e5cd6cccd8c" width="300">
<img src="https://github.com/user-attachments/assets/02293db2-0d6b-4358-afbc-13b49ba1d705" width="300">

## 2. Creating the resource group in Azure:

_In this part, we will create the container for our resources and the Private Link service on Azure._

From the Azure portal: ```https://portal.azure.com```:
Create a resource group to contain all Azure services. In the sidebar > create a resource > search for ```resource group``` (preferably filter only Azure services) > Create > Choose the subscription and enter the resource group name. **Note this resource group name for later.**

<img src="https://github.com/user-attachments/assets/3a2f0f70-98b0-4bee-889b-f590af92e954" width="500">
<img src="https://github.com/user-attachments/assets/ef9ced03-cb39-47ce-9876-d7cadd99b1b5" width="500">

## 3. Creating the Private Link Service for Power BI (also applies to Microsoft Fabric):

Once the resource group is created > In the notification center, go to the resource group > Click Create in the header > Search for ```Template deployment``` > Create. This template allows resource creation via code instead of menus.

<img src="https://github.com/user-attachments/assets/796576ae-db3a-4b57-b5df-62234418f900" width="300">
<img src="https://github.com/user-attachments/assets/9c076b8e-e2d5-4eb5-b71f-43469ba176d0" width="500">
<img src="https://github.com/user-attachments/assets/3dd14a61-365e-4f5b-85d9-c15de5bec98b" width="500">

Once the code block appears: copy the code below, replacing:
- The value of the name attribute with the private link service name
- The value of the tenantId attribute with the Power BI environment ID

```{
  "$schema": "http://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {},
  "resources": [
      {
          "type":"Microsoft.PowerBI/privateLinkServicesForPowerBI",
          "apiVersion": "2020-06-01",
          "name" : "<resource-name>",
          "location": "global",
          "properties" : 
          {
               "tenantId": "<tenant-object-id>"
          }
      }
  ]
}
```

After entering the details, go through the creation tabs, enter resource group info, then validate creation:

<img src="https://github.com/user-attachments/assets/4d72d13c-183e-436c-8328-205af0a96310" width="500">
<img src="https://github.com/user-attachments/assets/550dd9a7-6823-45a1-bb5a-98af24b5096d" width="400">
<img src="https://github.com/user-attachments/assets/a4777ba1-5507-4316-b382-6f416a26fb23" width="500">

## 4. Creating network services

The current network architecture does not follow the Hub N Spoke topology. Ideally, this architecture should be favored by splitting elements into different networks.

Also, in this case, there is no enterprise network to connect to the infrastructure. So, a virtual machine will be created directly in the test network. Ideally, a local machine connected to the enterprise network, itself connected to Azure via Express Route, would be used.

_In this part, we will create the necessary network resources for grouping our services._

**Creating the VNET**

From the resource group: Create > search _Virtual Network_ > Create a resource.

In the **Basics** tab, choose the previously created resource group, set the name and region of the virtual network.

<img src="https://github.com/user-attachments/assets/2355d410-83c6-4711-9c69-5134efa75b6a" width="500">

In the **Security** tab, enable the ```Azure Bastion``` feature, allowing access to the VM from a web tab, avoiding opening a remote control port.

<img src="https://github.com/user-attachments/assets/e7170bdb-dd44-40ed-b6fe-3738a641e106" width="500">

In the **IP Addresses** tab, set the network ranges needed for the infrastructure. In this case, the address count is small, so limit the reserved addresses for the test.

<img src="https://github.com/user-attachments/assets/48b68d16-2f6b-4607-908d-bf8c6e09c2f8" width="500">

**Creating the Private Endpoint for Power BI & Microsoft Fabric**

This service enables private communication with Microsoft Fabric.

From the resource group: Create > search _Private Endpoint_ > Create a resource.

In the **Basics** tab, enter the resource group and private endpoint name:

<img src="https://github.com/user-attachments/assets/9ed19018-0238-40d5-9ce9-e89f62d761fe" width="500">

In the Resource tab, choose Microsoft.PowerBI/privateLinkServicesForPowerBI for the resource type, then select the service created earlier via the ARM template. For sub-resource, choose _tenant_.

<img src="https://github.com/user-attachments/assets/63ee729e-056e-47b3-8043-3bcac9c7780d" width="500">

In the **Virtual Network** tab, select the VNET created earlier and keep the default subnet:

<img src="https://github.com/user-attachments/assets/ba171ea8-73a1-4a3f-8754-4e12e684ca1a" width="500">

In the **DNS** tab, set the 3 Private DNS Zones for the service (should be pre-filled):
  - _(New)privatelink.analysis.windows.net_
  - _(New)privatelink.pbidedicated.windows.net_
  - _(New)privatelink.prod.powerquery.microsoft.com_

<img src="https://github.com/user-attachments/assets/569ead0d-1dc0-4604-820b-39b043f03384" width="500">

## 5. Creating the virtual machine

Once the virtual network is created, create a VM to simulate a user workstation integrated into a private network. This is the entry point for accessing Microsoft Fabric.

From the resource group: Create > search _Virtual Machine_ > Create a resource.

In the **Basics** tab, enter the resource group and VM name.
For testing, choose ```No Infrastructure redundancy required``` for availability, > ```Standard``` for Security Type. For VM size, choose something simple: ```Standard_D2s_V6``` for example, with a ```Windows Server 2022``` image.

<img src="https://github.com/user-attachments/assets/d04620bb-1cce-4764-8185-85198e3439f1" width="500">

Further down, enter the administrator account and password.
Disable all inbound ports by selecting "None".

<img src="https://github.com/user-attachments/assets/1a7a8bbe-6c90-47eb-9f5d-00fd91b7a506" width="500">

Skip the **Disks** tab, and in **Networking**, select the previously created VNET.

<img src="https://github.com/user-attachments/assets/954cd52f-28b7-457e-9005-839e20162fba" width="500">

In the **Advanced** tab, enable Auto Shutdown to avoid unnecessary resource consumption:

<img src="https://github.com/user-attachments/assets/8a30b3af-460f-4ac1-bb69-63d73cf4c345" width="500">

**Connect via Bastion**

Once the VM is created, connect via Azure Bastion: Find the resource in the portal, in the sidebar go to Connect > Bastion > enter username & password > Click connect.

<img src="https://github.com/user-attachments/assets/5535616a-3938-4f2d-9f67-d863170f839d" width="500">

Once connected to the VM, start a command prompt: Windows Menu > type _cmd_.

Return to the Azure portal, find the _private endpoint_ resource created earlier, go to Settings > DNS Configuration: copy the value in the FQDN column:

<img src="https://github.com/user-attachments/assets/a68193ae-8274-4697-9874-5c8957cbcd9a" width="800">

From the Bastion tab for the VM, test the DNS FQDN:
Replace the FQDN value with the one from the portal: ```nslookup e171a59a21664df89256ed18abb8ee91-api.analysis.windows.net```

<img src="https://github.com/user-attachments/assets/528c56c5-54a0-43e6-8998-eb5dad5cefb3" width="500">

The goal is to see that the returned address is within our network (in 10.X.X.X) and not a public IP.

Once portal access via a private IP is possible, disable portal access via the Internet. From the VM, connect to the Power BI portal, go to tenant settings > Enable Block Public Internet Access:

<img src="https://github.com/user-attachments/assets/aef0c1ea-e111-4771-b57a-9420f2bac3ec" width="500">

**This setting cannot be disabled from a public IP for security reasons.**

<img src="https://github.com/user-attachments/assets/8c3ae0d1-2626-4f4f-be85-97356105645d" width="800">

If you try to access the portal directly from a browser outside the VM:

<img src="https://github.com/user-attachments/assets/aad07028-6c18-41ad-80cd-cfbe7529ab2a" width="600">

# Part 2: Accessing Azure services and data from Microsoft Fabric

# 1. Creating a storage account

To continue, we need to store data in Azure Data Lake Storage Gen 2.
From the portal, Create a resource > Search _Storage Account_ > Create.

In the **Basics** tab, enter the resource group, region, and resource name (no uppercase or special characters).

<img src="https://github.com/user-attachments/assets/967bc46a-c23c-48f0-a285-430ec8f39bc8" width="600">

In the **Advanced** tab, enable _Enable Storage Account Key Access_ to use the account key later, then Enable Hierarchical Namespace to convert the storage account to a data lake:

<img src="https://github.com/user-attachments/assets/99c078b6-8009-482a-a211-cf278d342f99" width="500">

In the networking tab, disable public access to the storage account.

<img src="https://github.com/user-attachments/assets/63a69146-5ef2-434c-aae4-9ae169d8364f" width="500">

Finish creating the storage account. Once created, from the sidebar, Data Storage > Containers > create a container. To go further, you can populate the storage account with data (either from inside the network or by temporarily re-enabling public network access).

<img src="https://github.com/user-attachments/assets/7abcac83-4384-4aa9-8764-102ba547e8cf" width="500">

# 2. Creating the workspace in Microsoft Fabric:

From the Fabric portal, create a workspace and assign it to a Fabric capacity from the **Advanced** menu.

<img src="https://github.com/user-attachments/assets/c24cd867-8b60-432b-a6d6-98e0f1979bf9" width="500">

# 3. Creating the Private Endpoint to ADLS Gen 2:

Once the workspace is created, go to the workspace and click Settings. In the sidebar, choose Network Security > then in Managed Private Endpoint > Create. Create a private endpoint from your workspace to the previously created ADLS Gen 2.

<img src="https://github.com/user-attachments/assets/6db69e94-632f-4422-8d5f-ef9358f10dd0" width="500">

Set the private endpoint name, then enter the storage account details, replacing the values:

```/subscriptions/{subscription-id}/resourceGroups/{resource-group-name}/providers/Microsoft.Storage/storageAccounts/{storage-account-name}```

For example, in our case: (these values can be found in

```/subscriptions/ecaa47e5-79d1-47e4-ac93-80b1484c3dbf/resourceGroups/rg-private-fabric-all-in/providers/Microsoft.Storage/storageAccounts/safabricprivate```

<img src="https://github.com/user-attachments/assets/afdf748a-13a8-4bf9-8567-6e3173109ce4" width="500">

Once created, it appears in the endpoints list with status _Activating_.

<img src="https://github.com/user-attachments/assets/4d1f3787-4515-49b0-ab56-367d308bc397" width="500">
<img src="https://github.com/user-attachments/assets/ced69224-96ba-422a-a683-e99af82ed63a" width="500">

In Azure, find the storage account to connect to > Networking > Private Endpoint Connections tab and Approve the private endpoint.

<img src="https://github.com/user-attachments/assets/25616761-58eb-4374-b926-027510233a8c" width="800">

Once activated, it is _Approved_ and _Succeeded_ in Fabric:

<img src="https://github.com/user-attachments/assets/893f2e18-2156-4fd3-a90a-b30f7a54a546" width="500">

Note: Once the Private Endpoint is deployed and used, the default Spark Pool cannot be used. No additional cost, but you must create a dedicated one. This is actually good practice: adapt your Spark pool to your needs and do not use the default pool.

<img src="https://github.com/user-attachments/assets/c6557c00-72a7-4118-887b-118fc68f1034" width="600">

To create it, in **Workspace Settings**, expand **Data Engineering**, create and set the default Spark pool:

<img src="https://github.com/user-attachments/assets/3c4b5b94-e748-4231-8996-836927eaec84" width="400">
<img src="https://github.com/user-attachments/assets/97cca32a-8cb0-40de-bf04-d41621b245db" width="400">

# Workspace Identities

Once the link between Azure and the storage account is established, define the method for connecting to the Key Vault. You can use the managed identity of the Fabric workspace: it is the workspace itself, not the notebook developer, that accesses the Key Vault.

To do this, still in **Workspace Settings**, expand **Workspace Identity** and click **+ Workspace Identity**

<img src="https://github.com/user-attachments/assets/c1359d7e-b59a-4308-9df3-38780dcb11d9" width="400">
<img src="https://github.com/user-attachments/assets/b765fe16-f328-4934-9a71-de26299bd597" width="300">

# Key Vault

Create the Key Vault in Azure and disable public access:

<img src="https://github.com/user-attachments/assets/d6136e47-4c03-4fa3-a011-9fadad19517d" width="300">
<img src="https://github.com/user-attachments/assets/efa8ce05-e461-4602-a57d-62597a8baa72" width="500">
<img src="https://github.com/user-attachments/assets/ada19118-87ae-45dc-a7db-e6ffac0ce9dc" width="300">

Create key
<img src="https://github.com/user-attachments/assets/a509871c-1e50-4a61-bd40-50d3f7cd1584" width="300">

Toto

<img src="https://github.com/user-attachments/assets/cf7cc421-399c-40f3-80b7-398a07f8c848" width="300">

<img src="https://github.com/user-attachments/assets/166f30b7-879c-413c-b71c-7c869b7da732" width="300">
<img src="https://github.com/user-attachments/assets/a2682b80-86b0-45fc-8fd6-ee1c3648467f" width="300">

Toto

<img src="https://github.com/user-attachments/assets/44141c0c-f1e7-488b-acb5-26806ee7fb69" width="300">
<img src="https://github.com/user-attachments/assets/d9fa770d-a26c-4dfb-8305-58a0d60f6777" width="300">

Toto

<img src="https://github.com/user-attachments/assets/30ea44ae-4430-4a93-85dd-c173af572cd2" width="300">
<img src="https://github.com/user-attachments/assets/04da8163-6f38-4182-bbeb-39e3a3f5ca41" width="300">

Repeat the operation to create a private endpoint for the Key Vault, which will contain the credentials to access data in the storage account.

```/subscriptions/{subscription-id}/resourceGroups/{resource-group-name}/providers/Microsoft.KeyVault/vaults/{vault-name}```
```/subscriptions/ecaa47e5-79d1-47e4-ac93-80b1484c3dbf/resourceGroups/rg-private-fabric-all-in/providers/Microsoft.KeyVault/vaults/kv-private-gennaker```

<img src="https://github.com/user-attachments/assets/81b4726b-42e7-45cf-b26c-ee9ff2cfe0fd" width="300">
<img src="https://github.com/user-attachments/assets/0c252fd4-f053-490c-87c2-9a496f807e1f" width="300">

# Use Key Vault & Notebook

<img src="https://github.com/user-attachments/assets/f3063673-09e1-4511-9a58-78be4d3c1385" width="300">
<img src="https://github.com/user-attachments/assets/595294b8-4783-41d1-aee8-e4faad48622d" width="300">
