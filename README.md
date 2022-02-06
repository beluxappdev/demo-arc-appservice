# Azure AppService via Azure Arc
Demo on how to leverage Azure AppService hosted on a generic k8s installation connected via Azure Arc

# Prerequisites
* [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) installed
* Azure Subscription
* Operational Kubernetes Cluster

# Prepare your Azure Subscription
## Login to your Subscription
```bash
az login [--use-device-code]
az account set --subscription "your-subscription-name/guid"
```

## Activate Providers
```bash
az provider register --namespace Microsoft.Kubernetes
az provider register --namespace Microsoft.KubernetesConfiguration
az provider register --namespace Microsoft.ExtendedLocation
```

## Check if Providers got activated
```bash
az provider show -n Microsoft.Kubernetes -o table
az provider show -n Microsoft.KubernetesConfiguration -o table
az provider show -n Microsoft.ExtendedLocation -o table
az provider register --namespace Microsoft.ExtendedLocation
```

## Enable Arc/k8s Extension
```bash
az extension add --name connectedk8s
```

# Set Variables
```bash
rgaks="demo-test-aks"
nameaks="demoakscluster"
rgarc="demo-test-arc"
namearc="demoarc"
region="westeurope"
```

# Preparing the Kubernetes environment (Optional : AKS Scenario)
```bash
az group create --name $rgaks --location $region --output table
az aks create -g $rgaks -n $nameaks --enable-cluster-autoscaler  --min-count 1 --max-count 5 --enable-addons monitoring --generate-ssh-keys -s "Standard_B4ms"
az aks get-credentials --name $nameaks --resource-group $rgaks
```

# Prepare Azure Arc
```bash
az group create --name $rgarc --location $region --output table
```

# Connect Kubernetes Cluster to Azure Arc
Ensure your kubectl is pointing to the cluster you want to connect
```bash
az connectedk8s connect --name $namearc --resource-group $rgarc
```

# 