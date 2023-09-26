# Running Spring Apps with minimum permissions 
The below walkthrough is aimed for customers who want to run Azure Spring Apps in a custom VNET and custom route table with minimum required permissions. This is just a temporary workaround until the product team adds support for custom roles.

## Workaround Summary 

1.  "User Access Administrator" and "Network Contributor" are needed for 3 operations as of now
- During deployment
- If you start/stop the whole service instance https://learn.microsoft.com/en-us/azure/spring-apps/how-to-start-stop-service?tabs=azure-portal
- During deletion
2. We will create 3 new custom role defentions which we will assign to the ASA service along with the above roles
- a role for the resource group where ASA will reside
- a role for the VNET access
- a role for the route table access
3. This custom role will be used during creation
4. Once the instance is created, we can modify the roles assigned to the service by deleting the "User Access Administrator" and "Network Contributor" roles, and then run our validations
5. in case we need to start/stop the whole service instance or we need to delete it, we need to modfiy the assigned roles by adding the "User Access Administrator" and "Network Contributor" roles again


## Define variables 

### Define variables 
```bash
RESOURCE_GROUP=rg-spring-cloud
VNET_RESOURCE_GROUP=rg-spring-cloud-vnet
SPRING_CLOUD_APP_NAME=az-spring-cloud-app
VNET_NAME=az-spring-cloud-vnet
RUNTIME_SUBNET_NAME=spring-runtime-subnet
APPS_SUBNET_NAME=spring-apps-subnet
RUNTIME_SUBNET_ROUTE_TABLE_NAME=spring-runtime-subnet-routetable
APPS_SUBNET_ROUTE_TABLE_NAME=spring-apps-subnet-routetable
LOCATION=northeurope

### create 2 resource group (one for ASA and one for the VNET)
az group create --name $RESOURCE_GROUP --location $LOCATION
az group create --name $VNET_RESOURCE_GROUP --location $LOCATION

### create a vnet
az network vnet create --name $VNET_NAME --resource-group $VNET_RESOURCE_GROUP --location $LOCATION --address-prefixes 192.168.0.0/16

### create RunTime Subnet subnet
az network vnet subnet create --name $RUNTIME_SUBNET_NAME --vnet-name $VNET_NAME --resource-group $VNET_RESOURCE_GROUP --address-prefixes 192.168.0.0/24

### create apps subnet 
az network vnet subnet create --name $APPS_SUBNET_NAME --vnet-name $VNET_NAME --resource-group $VNET_RESOURCE_GROUP --address-prefixes 192.168.1.0/24

### get the ASA resource group id 
RESOURCE_GROUP_ID=$(az group show --name $RESOURCE_GROUP --query id --output tsv)


### get the vnet id 
VNET_ID=$(az network vnet show --name $VNET_NAME --resource-group $VNET_RESOURCE_GROUP --query id --output tsv)

### get the runtime subnet id 
RUNTIME_SUBNET_ID=$(az network vnet subnet show --name $RUNTIME_SUBNET_NAME --vnet-name $VNET_NAME --resource-group $VNET_RESOURCE_GROUP --query id --output tsv)

### get the apps subnet id 
APPS_SUBNET_ID=$(az network vnet subnet show --name $APPS_SUBNET_NAME --vnet-name $VNET_NAME --resource-group $VNET_RESOURCE_GROUP --query id --output tsv)

###  if you have a custom route table (you need to do the same for both subnets) 
### get the route table id
RUNTIME_SUBNET_ROUTE_TABLE_ID=$(az network route-table show --name $RUNTIME_SUBNET_ROUTE_TABLE_NAME --resource-group $VNET_RESOURCE_GROUP --query id --output tsv)

APPS_SUBNET_ROUTE_TABLE_ID=$(az network route-table show --name $RUNTIME_SUBNET_ROUTE_TABLE_NAME --resource-group $VNET_RESOURCE_GROUP --query id --output tsv)
```
# Create Custom Role Definitions 

We will create 3 custom roles, one for the resource group where ASA will reside, one for the route table, and one for the virtual network, given that ASA is managing AKS clusters underneath, the roles required are similar to the below  
https://learn.microsoft.com/en-us/azure/aks/concepts-identity#aks-cluster-identity-permissions

*Note* Please discard anything VMAS related in the above link, ASA uses VMSS only 
*Note* Update the $YOUR_AZURE_SUB_ID to your subscription ID in the JSON files

```bash
### create a role definition for the resource group where ASA will reside (IPs, LBs, Disks, Storage, VMSS, etc...)
az role definition create --role-definition ./asa_permissions_resource_group.json

### create a role definition for the virtual network access 
az role definition create --role-definition ./asa_permissions_vnet.json

### create a role definition for the route table access
az role definition create --role-definition ./asa_permissions_route_table.json



## grant spring apps service permissions on the VNET, Route Tables, and Resource group 

### grant spring apps service permissions on the VNET
az role assignment create --assignee e8de9221-a19c-4c81-b814-fd37c6caf9d2 --scope $VNET_ID --role "ASA Permissions VNET"

### grant spring apps service permissions on the Resource Group where the ASA object will reside 
az role assignment create --assignee e8de9221-a19c-4c81-b814-fd37c6caf9d2 --scope $RESOURCE_GROUP_ID --role "ASA Permissions Role"

### grant spring apps service permissions on the Route Table (you will need to do this twice for both subnets)
az role assignment create --assignee e8de9221-a19c-4c81-b814-fd37c6caf9d2 --scope $RUNTIME_SUBNET_ROUTE_TABLE_ID --role "ASA Permissions Route Table"

az role assignment create --assignee e8de9221-a19c-4c81-b814-fd37c6caf9d2 --scope $APPS_SUBNET_ROUTE_TABLE_ID --role "ASA Permissions Route Table"

### below 2 roles are needed for creation, we will drop them right after creation ASA deployment 
az role assignment create --assignee e8de9221-a19c-4c81-b814-fd37c6caf9d2 --scope $VNET_ID --role "User Access Administrator"
az role assignment create --assignee e8de9221-a19c-4c81-b814-fd37c6caf9d2 --scope $VNET_ID --role "Network Contributor"

### list role assignments for ASA service 
az role assignment list --all --assignee e8de9221-a19c-4c81-b814-fd37c6caf9d2 --output table
```


# ASA Deployment 

```bash
### deploy an an azure spring apps instance in the vnet
az spring create \
--name $SPRING_CLOUD_APP_NAME \
--resource-group $RESOURCE_GROUP \
--location $LOCATION \
--vnet $VNET_ID \
--service-runtime-subnet $RUNTIME_SUBNET_ID \
--app-subnet $APPS_SUBNET_ID \
--enable-java-agent \
--sku standard 


### validate that the deployment was successful
az spring show --name $SPRING_CLOUD_APP_NAME --resource-group $RESOURCE_GROUP --output table




## Delete role assignments that were needed for creation 
As explained above "User Access Administrator" and "Network Contributor" roles are needed for creation, we will drop them right after creation ASA deployment

### delete role assignments that were needed for creation
az role assignment delete --assignee e8de9221-a19c-4c81-b814-fd37c6caf9d2 --scope $VNET_ID --role "User Access Administrator"
az role assignment delete --assignee e8de9221-a19c-4c81-b814-fd37c6caf9d2 --scope $VNET_ID --role "Network Contributor"

### list role assignments for ASA service
az role assignment list --all --assignee e8de9221-a19c-4c81-b814-fd37c6caf9d2 --output table

### validate that the deployment is still successful
az spring show --name $SPRING_CLOUD_APP_NAME --resource-group $RESOURCE_GROUP --output table
```

# Validations
In order to validate that deleting role assignment didn't break the installtion, we will be deploying an APP, try to scale which should add a new route table. in your case run the validations required for your use case.
.....


# Clean Up 
When time comes for deletion you will need to add the roles again to be able to delete the ASA instance 

```bash
### add the roles again to be able to delete the ASA instance
az role assignment create --assignee e8de9221-a19c-4c81-b814-fd37c6caf9d2 --scope $VNET_ID --role "User Access Administrator"
az role assignment create --assignee e8de9221-a19c-4c81-b814-fd37c6caf9d2 --scope $VNET_ID --role "Network Contributor"

### delete the ASA instance
az spring delete --name $SPRING_CLOUD_APP_NAME --resource-group $RESOURCE_GROUP --yes



### delete the resource groups
az group delete --name $RESOURCE_GROUP --yes
az group delete --name $VNET_RESOURCE_GROUP --yes
```
