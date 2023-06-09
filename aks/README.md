# Create AKS

## Check prerequirements
```sh
az provider show -n Microsoft.OperationsManagement -o table
az provider show -n Microsoft.OperationalInsights -o table
```

```sh
az provider register --namespace Microsoft.OperationsManagement
az provider register --namespace Microsoft.OperationalInsights
```


## Create AKS
```sh
# Set Cluster Name
AKS_CLUSTER=aksprod1
AKS_RESROURCE_GROUP=rg-aks-${AKS_CLUSTER}
AKS_VNET=aks-vnet
AKS_VNET_ADDRESS_PREFIX=10.0.0.0/8
AKS_VNET_SUBNET_DEFAULT=aks-subnet-default
AKS_VNET_SUBNET_DEFAULT_PREFIX=10.240.0.0/16
AKS_VNET_SUBNET_VIRTUALNODES=vnet-aks-${AKS_CLUSTER}-subnet-virtual-nodes
AKS_VNET_SUBNET_VIRTUALNODES_PREFIX=10.241.0.0/16
LOCATION="westeurope"

# Upgrade az CLI  (To latest version)
az --version
az upgrade

# Create resource group
az group create  \
    --name  ${AKS_RESOURCE_GROUP} \
    --location ${LOCATION}

# Create Virtual Network & default Subnet
az network vnet create -g ${AKS_RESOURCE_GROUP} \
    -n ${AKS_VNET} \
    --address-prefix ${AKS_VNET_ADDRESS_PREFIX} \
    --subnet-name ${AKS_VNET_SUBNET_DEFAULT} \
    --subnet-prefix ${AKS_VNET_SUBNET_DEFAULT_PREFIX} 

# Create Virtual Nodes Subnet in Virtual Network
az network vnet subnet create \
    --resource-group ${AKS_RESOURCE_GROUP} \
    --vnet-name ${AKS_VNET} \
    --name ${AKS_VNET_SUBNET_VIRTUALNODES} \
    --address-prefixes ${AKS_VNET_SUBNET_VIRTUALNODES_PREFIX}

# Get Virtual Network default subnet id
AKS_VNET_SUBNET_DEFAULT_ID=$(az network vnet subnet show \
    --resource-group ${AKS_RESOURCE_GROUP} \
    --vnet-name ${AKS_VNET} \
    --name ${AKS_VNET_SUBNET_DEFAULT} \
    --query id \
    -o tsv)

echo ${AKS_VNET_SUBNET_DEFAULT_ID}

# Create Azure AD Group
AKS_AD_AKSADMIN_GROUP_ID=$(az ad group create --display-name aksadmins --mail-nickname aksadmins --query objectId -o tsv)    
echo $AKS_AD_AKSADMIN_GROUP_ID

# Create Azure AD AKS Admin User 
# Replace with your AD Domain - aksadmin1@stacksimplifygmail.onmicrosoft.com
AKS_AD_AKSADMIN1_USER_OBJECT_ID=$(az ad user create \
  --display-name "AKS Admin1" \
  --user-principal-name aksadmin1@stacksimplifygmail.onmicrosoft.com \
  --password @AKSDemo123 \
  --query objectId -o tsv)

echo $AKS_AD_AKSADMIN1_USER_OBJECT_ID

# Associate aksadmin User to aksadmins Group
az ad group member add --group aksadmins --member-id $AKS_AD_AKSADMIN1_USER_OBJECT_ID

# Make a note of Username and Password
Username: aksadmin1@stacksimplifygmail.onmicrosoft.com
Password: @AKSDemo123

# Create Folder
mkdir $HOME/.ssh/aks-prod-sshkeys

# Create SSH Key
ssh-keygen \
    -m PEM \
    -t rsa \
    -b 4096 \
    -C "azureuser@myserver" \
    -f ~/.ssh/aks-prod-sshkeys/aksprodsshkey \
    -N mypassphrase

# List Files
ls -lrt $HOME/.ssh/aks-prod-sshkeys

# Set SSH KEY Path
AKS_SSH_KEY_LOCATION=/Users/kalyanreddy/.ssh/aks-prod-sshkeys/aksprodsshkey.pub
echo $AKS_SSH_KEY_LOCATION

# Create Log Analytics Workspace
AKS_MONITORING_LOG_ANALYTICS_WORKSPACE_ID=$(az monitor log-analytics workspace create \
    --resource-group ${AKS_RESOURCE_GROUP} \
    --workspace-name ${AKS_CLUSTER}-loganalytics-workspace1 \
    --query id \
    -o tsv)

echo $AKS_MONITORING_LOG_ANALYTICS_WORKSPACE_ID

# Create AKS cluster 
az aks create --resource-group ${AKS_RESOURCE_GROUP} \
    --name ${AKS_CLUSTER} \
    --enable-managed-identity \
    --ssh-key-value  ${AKS_SSH_KEY_LOCATION} \
    --admin-username aksnodeadmin \
    --node-count 1 \
    --enable-cluster-autoscaler \
    --min-count 1 \
    --max-count 10 \
    --network-plugin azure \
    --service-cidr 10.0.0.0/16 \
    --dns-service-ip 10.0.0.10 \
    --vnet-subnet-id ${AKS_VNET_SUBNET_DEFAULT_ID} \
    --enable-aad \
    --aad-admin-group-object-ids ${AKS_AD_AKSADMIN_GROUP_ID}\
    --aad-tenant-id ${AZURE_DEFAULT_AD_TENANTID} \
    --node-osdisk-size 30 \
    --node-vm-size Standard_DS2_v2 \
    --nodepool-labels nodepool-type=system nodepoolos=linux app=system-apps \
    --nodepool-name systempool \
    --nodepool-tags nodepool-type=system nodepoolos=linux app=system-apps \
    --enable-addons monitoring \
    --workspace-resource-id ${AKS_MONITORING_LOG_ANALYTICS_WORKSPACE_ID} \
    --enable-ahub \
    --zones {1,2,3}


```

## Get credentials

```sh
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
```
