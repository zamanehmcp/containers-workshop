# Module 4 - Lab 1: Create an Azure Kubernetes Service Cluster

In this lab, you will use the Azure CLI (command line interface) create an Azure Kubernetes Service (AKS) cluster that uses several nodes to provide the scale needed to support the application. The cluster architecture will consist of a single control plane and multiple nodes to enable management of workload resources. You will also create an Azure Container Registry (ACR), which will be used to store container images of the sample app, and attach it to the AKS cluster to enable app deployments.

AKS supports both Linux and Windows node pools via the Portal or Azure CLI. In this lab you will create the cluster to support Linux.

The lab instructions below use Powershell. You can also use CMD or Bash, but you may need to adjust the commands to be compliant with those shells.

## Create the AKS Cluster and Container Registry

### Initialize your shell environment

Create the environment variables you will use throughout this lab. Update each variable with the values appropriate to your environment.

Powershell

```console
$env:RESOURCE_GROUP="rg-containers-workshop"
$env:LOCATION="eastus"
$env:CLUSTER_NAME="aks-containers-workshop"
$env:CLUSTER_SUBNET_NAME="aks-subnet"
$env:ACR_NAME="azcrcontainersworkshop"
$env:AS_DBSRV_NAME="as-dbs-containers-workshop"
$env:AS_DBSRV_SKU="S0"
$env:VNET_RESOURCE_GROUP="MC_rg-containers-workshop_aks-containers-workshop_eastus"
$env:VNET_NAME="aks-vnet-30725576"
$env:VNET_ADDR_PREFIX="10.224.0.0/12"
$env:ASQL_SUBNET_NAME="asql-subnet"
$env:SUBNET_PREFIX="10.226.0.0/24"
$env:AS_VNET_RULE="asql-vnet-rule"
$env:AS_ENDPOINT_NAME="asql-subnet-endpoint"
$env:APPGW_NAME="appgw-containers-workshop"
```

**NOTE:** Unlike most Azure services, Azure Container Registry names cannot contain any special characters.

### Create the Azure Container Registry

```console
az acr create \
    --name $env:ACR_NAME \
    --resource-group $env:RESOURCE_GROUP \
    --sku standard \
    --admin-enabled true
```

### Create the AKS cluster

Create the cluster and attach the container registry created in the previous step.

```console
az aks create \
    --resource-group $env:RESOURCE_GROUP \
    --name $env:CLUSTER_NAME \
    --node-count 2 \
    --enable-addons http_application_routing,ingress-appgw \
    --node-vm-size Standard_B2s \
    --network-plugin azure \
    --enable-managed-identity \
    --appgw-name $env:APPGW_NAME \
    --appgw-subnet-cidr "10.225.0.0/16" \
    --generate-ssh-keys \
    --attach-acr $env:ACR_NAME
```

The above command creates a new AKS cluster named aks-containers-workshop within the rg-containers-workshop resource group. The cluster has two nodes defined by the --node-count parameter. To minimize costs of this lab, only two nodes are being used. These nodes are system nodes used to host system pods for critical services such as DNS and metrics services. The --node-vm-size parameter configures the cluster nodes as Standard_B2s-sized VMs. The HTTP application routing add-on is enabled via the --enable-addons flag. These nodes are going to be part of System mode. An additional 2 nodes will be created in the next step to host the deployed application. An Application Gateway will also be created to expose the single public IP address that is used to access the app on the AKS cluster and provide L7 load-balancing. The cluster will include an Application Gateway Ingress Controller (AGIC) that allows AKS to leverage the Application Gateway.

### Add a Nodepool to the AKS Cluster

Run the az aks nodepool add command to add another node pool that uses the default Linux operating system.

Powershell

```console
az aks nodepool add \
    --resource-group $env:RESOURCE_GROUP \
    --cluster-name $env:CLUSTER_NAME \
    --name userpool \
    --node-count 2 \
    --node-vm-size Standard_B2s
```

The command adds a new node pool (User mode) to the existing AKS cluster (created in the previous command). This new node pool can be used to host applications and workloads, instead of using the System node pool that was created in the previous step using az aks create.

### Link with kubectl

kubectl is a command line tools used to manage your cluster via the Kubernetes API. Link your Kubernetes cluster with kubectl by running `az aks get-credentials` command.

Powershell

```console
az aks get-credentials --name $env:CLUSTER_NAME --resource-group $env:RESOURCE_GROUP
```

This command adds an entry to your local ~/.kube/config file, which holds all the information to access your clusters. Kubectl enables you to manage multiple clusters from a single command-line interface.

Run the `kubectl get nodes` command to check that you can connect to your cluster, and confirm its configuration.

Powershell

```console
kubectl get nodes
```

The output should list four available nodes for two node pools.

Output

```console
NAME                                STATUS   ROLES   AGE    VERSION
aks-nodepool1-21895026-vmss000000   Ready    agent   245s   v1.23.12
aks-nodepool1-21895026-vmss000001   Ready    agent   245s   v1.23.12
aks-userpool-21895026-vmss000000    Ready    agent   105s   v1.23.12
aks-userpool-21895026-vmss000001    Ready    agent   105s   v1.23.12
```

## Add the Azure SQL database server to the AKS VNET

### Create a subnet for Azure SQL

```console
az network vnet subnet create \
    --name $env:$env:ASQL_SUBNET_NAME \
    --resource-group $env:RESOURCE_GROUP \
    --vnet-name $env:VNET_NAME \
    --address-prefixes $env:SUBNET_PREFIX
```

### Create a Service Endpoint for the Azure SQL database server

The Service Endpoint will be used to access the Azure SQL database by the apps deployed in AKS.

```console
az network vnet subnet update \
    --resource-group $env:RESOURCE_GROUP \
    --vnet-name $env:VNET_NAME \
    --name $env:ASQL_SUBNET_NAME \
    --service-endpoints Microsoft.Sql
```

### Create an Azure SQL VNET rule to allow access from the AKS subnet to the Azure SQL subnet

The rule will allow traffic between the app in AKS and the Azure SQL database. In order to create the rule, run the following command to get the complete resource ID of the AKS subnet. 

```console
$env:CLUSTER_SUBNET_ID=$(az network vnet subnet show \
    --resource-group $env:VNET_RESOURCE_GROUP \
    --name $env:CLUSTER_SUBNET_NAME \
    --vnet-name $env:VNET_NAME 
    --query id -o tsv)
```

Use the AKS cluster subnet ID to create the vnet rule. 

```console
az sql server vnet-rule create \
    --name $env:AS_VNET_RULE \
    --resource-group $env:RESOURCE_GROUP \
    --server $env:AS_DBSRV_NAME \
    --subnet $env:CLUSTER_SUBNET_ID
```
