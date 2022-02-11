# Azure AppService via Azure Arc
Demo on how to leverage Azure AppService hosted on a generic k8s installation connected via Azure Arc

# Howto use this repository
* [Optional] Clone this repo and launch a GitHub CodeSpaces on it
* Change the content of the variables
* Execute the commands
* [Mandatory] Enjoy!

# Prerequisites
* [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) installed
* Azure Subscription
* Operational Kubernetes Cluster

# Resources
This demo has been constructed after going through the following docs ; 
* https://docs.microsoft.com/en-us/azure/azure-arc/kubernetes/quickstart-connect-cluster?tabs=azure-cli
* https://docs.microsoft.com/en-us/azure/app-service/manage-create-arc-environment?tabs=bash
* https://docs.microsoft.com/en-us/cli/azure/aks?view=azure-cli-latest
* https://docs.microsoft.com/en-us/azure/app-service/quickstart-arc

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
az provider register --namespace Microsoft.ExtendedLocation
```

## Check if Providers got activated
```bash
az provider show -n Microsoft.Kubernetes -o table
az provider show -n Microsoft.KubernetesConfiguration -o table
az provider show -n Microsoft.ExtendedLocation -o table
```

## Enable Arc/k8s Extension
```bash
az extension add --name connectedk8s
```

## Enable AppService on Arc dependencies
```bash
az extension add --upgrade --yes --name k8s-extension
az extension add --upgrade --yes --name customlocation
az extension add --upgrade --yes --name appservice-kube
az provider register --namespace Microsoft.ExtendedLocation --wait
az provider register --namespace Microsoft.Web --wait
az provider register --namespace Microsoft.KubernetesConfiguration --wait
```

# Set Variables
```bash
rgaks="demo-test-aks"
nameaks="demoakscluster"
rgarc="demo-test-arc"
namearc="demoarc"
region="westeurope"
extensionName="appservice-ext"
namespace="appservice-ns"
kubeEnvironmentName="appservice-arc"
workspaceName="appservice-arc"
customLocationName="my-custom-apparc"
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

# Setup Log Analytics for AppService on Arc
```bash
az monitor log-analytics workspace create \
    --resource-group $rgarc \
    --workspace-name $workspaceName

logAnalyticsWorkspaceId=$(az monitor log-analytics workspace show \
    --resource-group $rgarc \
    --workspace-name $workspaceName \
    --query customerId \
    --output tsv)

logAnalyticsWorkspaceIdEnc=$(printf %s $logAnalyticsWorkspaceId | base64 -w0)

logAnalyticsKey=$(az monitor log-analytics workspace get-shared-keys \
    --resource-group $rgarc \
    --workspace-name $workspaceName \
    --query primarySharedKey \
    --output tsv)

logAnalyticsKeyEnc=$(printf %s $logAnalyticsKey | base64 -w0) 
```

# Install the AppService extension for Azure Arc
```bash
az k8s-extension create \
    --resource-group $rgarc \
    --name $extensionName \
    --cluster-type connectedClusters \
    --cluster-name $namearc \
    --extension-type 'Microsoft.Web.Appservice' \
    --release-train stable \
    --auto-upgrade-minor-version true \
    --scope cluster \
    --release-namespace $namespace \
    --configuration-settings "Microsoft.CustomLocation.ServiceAccount=default" \
    --configuration-settings "appsNamespace=${namespace}" \
    --configuration-settings "clusterName=${kubeEnvironmentName}" \
    --configuration-settings "keda.enabled=true" \
    --configuration-settings "buildService.storageClassName=default" \
    --configuration-settings "buildService.storageAccessMode=ReadWriteOnce" \
    --configuration-settings "customConfigMap=${namespace}/kube-environment-config" \
    --configuration-settings "envoy.annotations.service.beta.kubernetes.io/azure-load-balancer-resource-group=${aksClusterGroupName}" \
    --configuration-settings "logProcessor.appLogs.destination=log-analytics" \
    --configuration-protected-settings "logProcessor.appLogs.logAnalyticsConfig.customerId=${logAnalyticsWorkspaceIdEnc}" \
    --configuration-protected-settings "logProcessor.appLogs.logAnalyticsConfig.sharedKey=${logAnalyticsKeyEnc}"

extensionId=$(az k8s-extension show \
    --cluster-type connectedClusters \
    --cluster-name $namearc \
    --resource-group $rgarc \
    --name $extensionName \
    --query id \
    --output tsv)
```

# (Optional) Validate AppService Health
```bash
az resource wait --ids $extensionId --custom "properties.installState!='Pending'" --api-version "2020-07-01-preview"

kubectl get pods -n $namespace
```

# Create Customer Location
```bash
connectedClusterId=$(az connectedk8s show --resource-group $rgarc --name $namearc --query id --output tsv)
az customlocation create \
    --resource-group $rgarc \
    --name $customLocationName \
    --host-resource-id $connectedClusterId \
    --namespace $namespace \
    --cluster-extension-ids $extensionId
customLocationId=$(az customlocation show \
    --resource-group $rgarc \
    --name $customLocationName \
    --query id \
    --output tsv)

```

# (Optional) Validate Customer Location
```bash
az customlocation show --resource-group $rgarc --name $customLocationName
```

# Create App Service Environment
```bash
az appservice kube create \
    --resource-group $rgarc \
    --name $kubeEnvironmentName \
    --custom-location $customLocationId
```

# (Optional) Validate App Service Environment on Arc
```bash
az appservice kube show --resource-group $rgarc --name $kubeEnvironmentName
```
# Create WebApp on Arc enabled App Service Environment
```bash
appname=$(cat /proc/sys/kernel/random/uuid)
az webapp create \
    --resource-group $rgarc \
    --name $appname \
    --custom-location $customLocationId \
    --runtime 'NODE|12-lts'
``` 

# (Variant A) Deploy code to Webapp
```bash
git clone https://github.com/Azure-Samples/nodejs-docs-hello-world
cd nodejs-docs-hello-world
zip -r package.zip .
az webapp deployment source config-zip --resource-group $rgarc --name $appname --src package.zip
```

# (Variant B) Deploy a container to the Webapp
```bash
az webapp create \
    --resource-group $rgarc \
    --name $appname \
    --custom-location $customLocationId \
    --deployment-container-image-name mcr.microsoft.com/appsvc/node:12-lts
```