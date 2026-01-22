# Azure Kubernetes Fleet Manager - Update Runs with GitHub Actions

This repository demonstrates how to set up Azure Kubernetes Fleet Manager (AKFM) with multiple member clusters and automate update runs using GitHub Actions. The setup includes creating a fleet manager, adding member clusters, and configuring a GitHub repository to manage update runs.

## Prerequisites

Before you begin, ensure you have the following tools installed:

- [Azure CLI](https://learn.microsoft.com/cli/azure/?view=azure-cli-latest)
- [GitHub CLI](https://cli.github.com/)
- POSIX-compliant shell
- [jq](https://jqlang.org/)

## Azure Setup

Log into Azure subscription.

```bash
az login
```

Ensure Azure CLI extensions are installed.

```bash
az extension add --name aks-preview
az extension add --name fleet
```

Set random variable for resource naming. This helps avoid naming conflicts.

```bash
RAND=$RANDOM
export RAND
echo "Random resource identifier will be: ${RAND}"
```

Set location for fleet resource. You may choose any supported region.

```bash
LOCATION=westcentralus
```

Create fleet manager resource group.

```bash
RG_NAME=rg-$RAND-fleet
az group create --name $RG_NAME --location $LOCATION
```

Create fleet manager without a hub cluster. For this sample, hub cluster is not required.

```bash
FLEET_NAME=fl-$RAND-west
az fleet create --resource-group $RG_NAME --name $FLEET_NAME --location $LOCATION
```

Set locations where member clusters will be created. You may choose any supported regions.

```bash
LOCATIONS=("westus" "westus2" "westus3")
```

For each location, find the oldest available Kubernetes version, create resource group, AKS cluster, and add as a member of the fleet

```bash
for ((i=0; i<${#LOCATIONS[@]}; i++)); do
  # create resource group
  az group create --name "rg-$RAND-$((i+1))" --location "${LOCATIONS[$i]}"
  
  # get k8s minor version
  K8S_MINOR_VERSION=$(az aks get-versions --location "${LOCATIONS[$i]}" --query "values[?contains(capabilities.supportPlan[], 'KubernetesOfficial')].version | sort(@) | [0]" --output tsv)
  
  # get k8s patch version
  K8S_PATCH_VERSION=$(az aks get-versions --location "${LOCATIONS[$i]}" --output json | jq -r --arg ver "$K8S_MINOR_VERSION" '.values[] | select(.version == $ver) | .patchVersions | keys[0]')
  
  # create aks cluster
  AKS_ID=$(az aks create -n "aks-$RAND-$((i+1))" --resource-group "rg-$RAND-$((i+1))" --node-count 1 --kubernetes-version "$K8S_PATCH_VERSION" --node-vm-size standard_b2ms --query id --output tsv)
  
  # add aks cluster to fleet
  az fleet member create --resource-group "$RG_NAME" --fleet-name "$FLEET_NAME" --name "aks-$RAND-$((i+1))" --member-cluster-id "$AKS_ID"
done
```

Confirm cluster setup is complete. This command will list all member clusters with their Kubernetes versions and node image versions.

```bash
az fleet member list -g $RG_NAME --fleet-name $FLEET_NAME | jq -r '.[].clusterResourceId' | while read -r cluster_id; do
  az resource show --ids $cluster_id --query "{name:name,nodeImageVersion:properties.agentPoolProfiles[0].nodeImageVersion,kubernetesVersion:properties.kubernetesVersion}"
done
```

## GitHub Setup

Log into GitHub. Ensure you have the necessary scopes to create/delete repositories, manage secrets/variables, and workflows.

```bash
gh auth login --scopes repo,workflow,delete_repo
```

Set new repo name.

```bash
REPO_NAME=akfm-${RAND}
GITHUB_REPO=$(gh api user --jq .login)/akfm-${RAND}
```

Create new private repo. The current repo includes sample GitHub workflows and will be used as a template.

```bash
gh repo create $GITHUB_REPO --template pauldotyu/akfm-update-runner --private --clone
cd $REPO_NAME
```

Set subscription and tenant variables which will be used to create service principal and federated credential.

```bash
SUBSCRIPTION_ID=$(az account show --query id --output tsv)
TENANT_ID=$(az account show --query tenantId --output tsv)
```

Create a service principal and assign Contributor role to subscription scope.

```bash
APP_ID=$(az ad sp create-for-rbac --name sp-$RAND --role Contributor --scopes "/subscriptions/${SUBSCRIPTION_ID}" --create-password false --query appId --output tsv)
```

Create a federated credential for the service principal to allow GitHub Actions to authenticate using workload identity federation.

```bash
az ad app federated-credential create \
  --id $APP_ID \
  --parameters "$(cat <<EOF
{
    "name": "${REPO_NAME}",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:${GITHUB_REPO}:ref:refs/heads/main",
    "audiences": [
        "api://AzureADTokenExchange"
    ]
}
EOF
)"
```

Set repo secrets for workload identity authentication. These secrets will be used by GitHub Actions to authenticate to Azure.

```bash
gh secret set AZURE_CLIENT_ID --body $APP_ID --repo $GITHUB_REPO
gh secret set AZURE_SUBSCRIPTION_ID --body $SUBSCRIPTION_ID --repo $GITHUB_REPO
gh secret set AZURE_TENANT_ID --body $TENANT_ID --repo $GITHUB_REPO
```

Set repo variables. This includes fleet name and resource group name which will be used in the GitHub Actions workflows for scheduled update runs.

```bash
gh variable set FLEET_NAME --body $FLEET_NAME --repo $GITHUB_REPO
gh variable set FLEET_RESOURCE_GROUP --body $RG_NAME --repo $GITHUB_REPO
```

## Test manual run

With the setup complete, you can now test a manual update run using GitHub Actions. Trigger the manual update run workflow and provide the fleet resource group and fleet name as inputs.

```bash
gh workflow run manual-update.yml -f resource_group=$RG_NAME -f fleet_name=$FLEET_NAME
```

Watch the update run. This command retrieves the latest run ID for the manual update workflow and watches its progress.

```bash
gh run watch $(gh run list --workflow="manual-update.yml" --json databaseId --jq '.[0].databaseId')
```

## Additional testing

You can test additional scenarios such as testing update runs when member clusters differ in Kubernetes versions or node image versions. You can also modify the scheduled update run workflow to change the schedule frequency or add additional scheduled runs to update just the node image versions or Kubernetes versions.

## Cleanup

Once you have completed your testing, you can clean up the resources created during this setup to avoid incurring unnecessary costs.

```bash
az group delete --name rg-$RAND-1 -y --no-wait
az group delete --name rg-$RAND-2 -y --no-wait
az group delete --name rg-$RAND-3 -y --no-wait
az group delete --name rg-$RAND-fleet -y --no-wait
az ad sp delete --id $APP_ID
gh repo delete $GITHUB_REPO
```
