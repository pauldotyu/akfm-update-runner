# Azure Kubernetes Fleet Manager Update Runner

## Prerequisites

- Azure CLI
- GitHub CLI
- POSIX-compliant shell
- jq

## Azure Setup

Log into Azure subscription

```
az login
```

Ensure Azure CLI extensions are installed

```
az extension add --name aks-preview
az extension add --name fleet
```

Set random variable for resource naming

```
RAND=$RANDOM
export RAND
echo "Random resource identifier will be: ${RAND}"
```

Set location for fleet resource

```
LOCATION=westcentralus
```

Create fleet manager resource group

```
RG_NAME=rg-$RAND-fleet
az group create --name $RG_NAME --location $LOCATION
```

Create fleet manager without a hub cluster

```
FLEET_NAME=fl-$RAND-west
az fleet create --resource-group $RG_NAME --name $FLEET_NAME --location $LOCATION
```

Set locations where member clusters will be created

```
LOCATIONS=("westus" "westus2" "westus3")
```

For each location, find the oldest available Kubernetes version, create resource group, AKS cluster, and add as a member of the fleet

```
for ((i=0; i<${#LOCATIONS[@]}; i++)); do
  # create resource group
  az group create -n "rg-$RAND-$((i+1))" -l "${LOCATIONS[$i]}"
  
  # get k8s minor version
  K8S_MINOR_VERSION=$(az aks get-versions -l "${LOCATIONS[$i]}" --query "values[?contains(capabilities.supportPlan[], 'KubernetesOfficial')].version | sort(@) | [0]" --output tsv)
  
  # get k8s patch version
  K8S_PATCH_VERSION=$(az aks get-versions -l "${LOCATIONS[$i]}" -o json | jq -r --arg ver "$K8S_MINOR_VERSION" '.values[] | select(.version == $ver) | .patchVersions | keys[0]')
  
  # create aks cluster
  AKS_ID=$(az aks create -n "aks-$RAND-$((i+1))" -g "rg-$RAND-$((i+1))" --node-count 1 --kubernetes-version "$K8S_PATCH_VERSION" --node-vm-size standard_b2ms --query id -o tsv)
  
  # add aks cluster to fleet
  az fleet member create --resource-group "$RG_NAME" --fleet-name "$FLEET_NAME" --name "aks-$RAND-$((i+1))" --member-cluster-id "$AKS_ID"
done
```

Confirm cluster setup is complete

```
az fleet member list -g $RG_NAME --fleet-name $FLEET_NAME | jq -r '.[].clusterResourceId' | while read -r cluster_id; do
  az resource show --ids $cluster_id --query "{name:name,nodeImageVersion:properties.agentPoolProfiles[0].nodeImageVersion,kubernetesVersion:properties.kubernetesVersion}"
done
```

## GitHub Setup

Log into GitHub

```
gh auth login --scopes repo,workflow,delete_repo
```

Set new repo name

```
REPO_NAME=akfm-${RAND}
GITHUB_REPO=$(gh api user --jq .login)/akfm-${RAND}
```

Create new private repo

```
gh repo create $GITHUB_REPO --template pauldotyu/akfm-updaterunner --private --clone
cd $REPO_NAME
```

Set subscription and tenant variables

```
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
TENANT_ID=$(az account show --query tenantId -o tsv)
```

Create a service principal and assign Contributor role to subscription scope

```
APP_ID=$(az ad sp create-for-rbac -n sp-$RAND --role Contributor --scopes "/subscriptions/${SUBSCRIPTION_ID}" --create-password false --query appId -o tsv)
```

Create a federated credential

```
az ad app federated-credential create \
  --id "$APP_ID" \
  --parameters "$(cat <<EOF
{
    "name": "$APP_ID",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:${GITHUB_REPO}:ref:refs/heads/main",
    "audiences": [
        "api://AzureADTokenExchange"
    ]
}
EOF
)"
```

Set repo secrets for workload identity authentication

```
gh secret set AZURE_CLIENT_ID --body $APP_ID --repo $GITHUB_REPO
gh secret set AZURE_SUBSCRIPTION_ID --body $SUBSCRIPTION_ID --repo $GITHUB_REPO
gh secret set AZURE_TENANT_ID --body $TENANT_ID --repo $GITHUB_REPO
```

Set repo variables

```
gh variable set FLEET_NAME --body $FLEET_NAME --repo $GITHUB_REPO
gh variable set FLEET_RESOURCE_GROUP --body $RG_NAME --repo $GITHUB_REPO
```
## Cleanup

```
az group delete -n rg-$RAND-1 -y --no-wait
az group delete -n rg-$RAND-2 -y --no-wait
az group delete -n rg-$RAND-3 -y --no-wait
az group delete -n rg-$RAND-fleet -y --no-wait
az ad sp delete -n sp-$RAND -y --no-wait
gh repo delete akfm-$RAND
```