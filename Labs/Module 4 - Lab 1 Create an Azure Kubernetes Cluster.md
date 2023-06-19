# Module 4 - Lab 1: Create an Azure Kubernetes Service Cluster

In this lab, you will use the Azure CLI (command line interface) create an AKS cluster that uses several nodes to provide the scale needed to support the application using AKS. The cluster architecture will consist of a single control plane and multiple nodes to enable management of workload resources. You will also create an Azure Container Registry that will be used to store container images

AKS supports both Linux and Windows node pools via the Portal or Azure CLI. In this lab you will create the cluster to support Linux.

The lab instructions below use Powershell. You can also use CMD or bash, but you may need to adjust the commands to be compliant with those shells.

## Create the AKS Cluster and Container Registry

### Create variables for the configuration values you reuse throughout the exercises.

Create the environment variables you will use throughout this lab. Update the LOCATION variable with the region closest to you. For example: "East US" or "eastus".

Powershell

```console
$env:RESOURCE_GROUP="rg-containers-workshop"
$env:CLUSTER_NAME="aks-containers-workshop"
$env:ACR_NAME="acrcontainersworkshop"
$env:LOCATION="<myLocation>"    
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
    --enable-addons http_application_routing \
    --generate-ssh-keys \
    --node-vm-size Standard_B2s \
    --network-plugin azure \
    --attach-acr $env:ACR_NAME
```

The above command creates a new AKS cluster named aks-containers-workshop within the rg-containers-workshop resource group. The cluster has two nodes defined by the --node-count parameter. To minimize costs of this lab, only two nodes are being used. These nodes are system nodes used to host system pods for critical services such as DNS and metrics services. The --node-vm-size parameter configures the cluster nodes as Standard_B2s-sized VMs. The HTTP application routing add-on is enabled via the --enable-addons flag. These nodes are going to be part of System mode.

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
az aks get-credentials --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP
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
