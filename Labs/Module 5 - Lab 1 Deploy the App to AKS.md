# Module 5 - Lab 1: Deploy the App to Azure Kubernetes Cluster

In this lab, you will use the Azure CLI (command line interface) deploy the sample `todoapp` app to an Azure Kubernetes Service (AKS) cluster that has several nodes to provide the scale needed to support the application.

The lab instructions below use Powershell. You can also use CMD or bash, but you may need to adjust the commands to be compliant with those shells.

## Prepare our environment and connect to Azure resources

### Create environment variables

Create the environment variables you will use throughout this lab. Update the variables with the information from your environment.

Powershell

```console
$env:RESOURCE_GROUP="rg-containers-workshop"
$env:LOCATION="eastus"
$env:CLUSTER_NAME="aks-containers-workshop"
$env:ACR_NAME="acrcontainersworkshop"
$env:ACR_ADMIN_USER="acrcontainersworkshop"
$env:AS_DBSRV_NAME="as-dbs-containers-workshop"
$env:AS_DBSRV_SKU="S0"
```

**NOTE:** Unlike most Azure services, Azure Container Registry names cannot contain any special characters.

### Login to ACR using Azure CLI

Login to ACR so that you can push your container image to the registry. You will find the admin username and password in the Azure Portal on the `Access Keys` blade of the ACR portal page.

Powershell

```console
az acr login --name $env:ACR_NAME --username $env:ACR_ADMIN_USER
```

### Connect kubectl to AKS using Azure AKS CLI

Connect the `kubectl` command line utility to your AKS cluster. `kubectl` is used to manage AKS.

**NOTE:** If you do not have kubectl installed on your workstation, used the following command to install the Azure AKS CLI on your workstation.

```console
az aks install-cli
```

Connect kubectl to your AKS cluster.

```console
az aks get-credentials --resource-group $env:RESOURCE_GROUP --name $env:CLUSTER_NAME
```

You should see a return message similar to the following:

```console
Merged "aks-containers-workshop" as current context in C:\Users\loublick\.kube\config
```

Run the kubectl command to return the running nodes in your AKS cluster to verify access to AKS.

```console
kubectl get nodes
```

