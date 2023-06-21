# Module 5 - Lab 1: Deploy the App to Azure Kubernetes Cluster

In this lab, you will use the Azure CLI (command line interface) deploy the sample `todoapp` app to an Azure Kubernetes Service (AKS) cluster that has several nodes to provide the scale needed to support the application.

The lab instructions below use Powershell. You can also use CMD or Bash, but you may need to adjust the commands to be compliant with those shells.

## Prepare the environment and connect to Azure

### Create environment variables

Create the environment variables you will use throughout this lab. Update the variables with the information from your environment.

Powershell

```console
$env:RESOURCE_GROUP="rg-containers-workshop"
$env:LOCATION="eastus"
$env:CLUSTER_NAME="aks-containers-workshop"
$env:ACR_NAME="azcrcontainersworkshop"
$env:ACR_ADMIN_USER="azcrcontainersworkshop"
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

If the conttext already exists in the configuration file, you will be prompted to overwite the existing context. Accept the overwrite by responding with a "y" (yes).

Run the kubectl command to return the running nodes in your AKS cluster to verify access to AKS.

```console
kubectl get nodes
```

## Deploy the app Deployment and internal ingress manifests to the AKS cluster

### Create and apply the app Deployment manifest

Create a new file in the /App/Working/todoapp folder called todoapp.yaml. This file will be the app Deployment manifest. Add the following deployment configuration to the file and save the file.

```console
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todoapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: todoapp
  template:
    metadata:
      labels:
        app: todoapp
    spec:
      nodeSelector:
        "kubernetes.io/os": linux
      containers:
      - name: todoapp
        image: azcrcontainersworkshop.azurecr.io/todoapp:v1
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 250m
            memory: 256Mi
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: todoapp
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: todoapp
```

Use `kubectl` to deploy the app Deployment manifest to the AKS cluster

```console
kubectl apply -f .\todoapp.yaml
```

### Create and apply the internal ingress manifest

```console
#ingress.yaml
apiVersion: v1
kind: Pod
metadata:
  name: todoapp
  labels:
    app: todoapp
spec:
  containers:
  - image: "acrcontainersworkshop.azurecr.io/todoapp:v1"
    name: todoapp-image
    ports:
    - containerPort: 80
      protocol: TCP

---

apiVersion: v1
kind: Service
metadata:
  name: todoapp
spec:
  selector:
    app: todoapp
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80

---

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: todoapp
  annotations:
    kubernetes.io/ingress.class: azure/application-gateway
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          service:
            name: todoapp
            port:
              number: 80
        pathType: Exact
```

Use `kubectl` to deploy the internal ingress manifest to the AKS cluster

```console
kubectl apply -f .\internal-ingress.yaml
```
