# Azure Kubernetes Fleet Manager - Update Runs with GitHub Actions

This repository demonstrates how to set up Azure Kubernetes Fleet Manager with multiple member clusters and automate update runs using GitHub Actions.

## What you'll create

- 1 Fleet Manager (no hub cluster)
- 3 AKS member clusters in different regions
- GitHub Actions workflows to run update runs (manual + scheduled)

## Prerequisites

Before you begin, ensure you have the following tools installed:

- [Azure CLI](https://learn.microsoft.com/cli/azure/?view=azure-cli-latest)
- [GitHub CLI](https://cli.github.com/)
- POSIX-compliant shell
- [jq](https://jqlang.org/)

## Variables at a glance

| Variable | Purpose |
| --- | --- |
| `RAND` | Random suffix to avoid name collisions |
| `LOCATION` | Fleet Manager region |
| `RG_NAME` | Fleet Manager resource group |
| `FLEET_NAME` | Fleet Manager name |
| `LOCATIONS` | AKS member regions |
| `REPO_NAME` | New GitHub repo name |

## Azure Setup

### Permissions

- Your Azure identity must be able to create resource groups, AKS clusters, and Fleet resources.
- The service principal created for GitHub Actions is granted `Contributor` at subscription scope.

### Log into Azure

Authenticate to your Azure account so subsequent CLI commands can create resources.

```bash
az login
```

### Install Azure CLI extensions

Install required AKS and Fleet extensions that provide the needed commands.

```bash
az extension add --name aks-preview
az extension add --name fleet
```

### Set a random suffix

Generate a random suffix to avoid naming collisions across resources.

```bash
RAND=$RANDOM
export RAND
echo "Random resource identifier will be: ${RAND}"
```

### Set fleet location

Choose the Azure region where the Fleet Manager will be created.

```bash
LOCATION=westcentralus
```

### Create fleet resource group

Create a resource group to contain the Fleet Manager.

```bash
RG_NAME=rg-$RAND-fleet
az group create --name $RG_NAME --location $LOCATION
```

### Create Fleet Manager (no hub cluster)

Create the Fleet Manager resource without a hub cluster (not needed for this sample).

```bash
FLEET_NAME=fl-$RAND-west
az fleet create --resource-group $RG_NAME --name $FLEET_NAME --location $LOCATION
```

### Set member cluster locations

Select regions where the AKS member clusters will be created.

```bash
LOCATIONS=("westus" "westus2" "westus3")
```

### Create member clusters and add to fleet

Create an AKS cluster per region and register each one as a fleet member.

```bash
for ((i=1; i<=${#LOCATIONS[@]}; i++)); do
  # create resource group
  az group create --name "rg-$RAND-$i" --location "${LOCATIONS[$i]}"
  
  # get k8s minor version
  K8S_MINOR_VERSION=$(az aks get-versions --location "${LOCATIONS[$i]}" --query "values[?contains(capabilities.supportPlan[], 'KubernetesOfficial')].version | sort(@) | [0]" --output tsv)
  
  # get k8s patch version
  K8S_PATCH_VERSION=$(az aks get-versions --location "${LOCATIONS[$i]}" --output json | jq -r --arg ver "$K8S_MINOR_VERSION" '.values[] | select(.version == $ver) | .patchVersions | keys[0]')
  
  # create aks cluster
  AKS_ID=$(az aks create --name "aks-$RAND-$i" --resource-group "rg-$RAND-$i" --node-count 1 --kubernetes-version "$K8S_PATCH_VERSION" --node-vm-size standard_b2ms --query id --output tsv)
  
  # add aks cluster to fleet
  az fleet member create --resource-group "$RG_NAME" --fleet-name "$FLEET_NAME" --name "aks-$RAND-$i" --member-cluster-id "$AKS_ID"
done
```

### Verify member clusters

Expected output: a list of member cluster names with `kubernetesVersion` and `nodeImageVersion` fields populated.

List each member cluster and print its Kubernetes and node image versions to confirm setup.

```bash
az fleet member list -g $RG_NAME --fleet-name $FLEET_NAME | jq -r '.[].clusterResourceId' | while read -r cluster_id; do
  az resource show --ids $cluster_id --query "{name:name,nodeImageVersion:properties.agentPoolProfiles[0].nodeImageVersion,kubernetesVersion:properties.kubernetesVersion}"
done
```

## GitHub Setup

### Log into GitHub

Ensure you have the scopes needed to create/delete repositories, manage secrets/variables, and workflows.

Authenticate to GitHub with the scopes required for repo and workflow management.

```bash
gh auth login --scopes repo,workflow,delete_repo
```

### Set repo name

Define the new repository name and full owner/name path.

```bash
REPO_NAME=akfm-${RAND}
GITHUB_REPO=$(gh api user --jq .login)/${REPO_NAME}
```

### Create repo from template

Create a private repo from the template and clone it locally.

```bash
gh repo create $GITHUB_REPO --template pauldotyu/akfm-update-runner --private --clone
cd $REPO_NAME
```

### Capture subscription and tenant IDs

Capture the subscription and tenant IDs for use with the service principal and secrets.

```bash
SUBSCRIPTION_ID=$(az account show --query id --output tsv)
TENANT_ID=$(az account show --query tenantId --output tsv)
```

### Create service principal

Create a service principal with Contributor access at subscription scope for GitHub Actions.

```bash
CLIENT_ID=$(az ad sp create-for-rbac --name sp-$RAND --role Contributor --scopes "/subscriptions/${SUBSCRIPTION_ID}" --create-password false --query appId --output tsv)
```

### Create federated credential

Allow GitHub Actions to authenticate to Azure using workload identity federation.

```bash
az ad app federated-credential create \
  --id $CLIENT_ID \
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

### Set GitHub secrets

Store Azure identity values as GitHub Secrets for the workflows.

```bash
gh secret set AZURE_CLIENT_ID --body $CLIENT_ID --repo $GITHUB_REPO
gh secret set AZURE_SUBSCRIPTION_ID --body $SUBSCRIPTION_ID --repo $GITHUB_REPO
gh secret set AZURE_TENANT_ID --body $TENANT_ID --repo $GITHUB_REPO
```

### Set GitHub variables

Store fleet identifiers as GitHub Variables for reuse in workflows.

```bash
gh variable set FLEET_NAME --body $FLEET_NAME --repo $GITHUB_REPO
gh variable set FLEET_RESOURCE_GROUP --body $RG_NAME --repo $GITHUB_REPO
```

## Test manual run

With the setup complete, you can now test a manual update run using GitHub Actions. Trigger the manual update run workflow and provide the fleet resource group and fleet name as inputs.

Trigger the manual update workflow for a one-off update run. This will bump all member clusters to the next available Kubernetes and latest node image versions.

```bash
gh workflow run manual-update.yml -f resource_group=$RG_NAME -f fleet_name=$FLEET_NAME
```

Watch the update run. This command retrieves the latest run ID for the manual update workflow and watches its progress.

Follow the latest workflow run until it completes.

```bash
gh run watch $(gh run list --workflow="manual-update.yml" --json databaseId --jq '.[0].databaseId')
```

## Additional testing

- Vary member cluster Kubernetes versions to test rollout behavior.
- Vary node image versions to test node-image-only updates.
- Adjust the schedule or add additional scheduled workflows for specific update types.

## Troubleshooting

- `az fleet` or `az aks` commands fail: confirm `aks-preview` and `fleet` extensions are installed.
- `K8S_MINOR_VERSION` or `K8S_PATCH_VERSION` is empty: check region support and output of `az aks get-versions`.
- GitHub Actions auth fails: confirm `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, and `AZURE_SUBSCRIPTION_ID` are set and the federated credential `subject` matches the repo/branch.
- Workflow run times out: GitHub-hosted runners can execute for up to 6 hours. If execution take longer than 6 hours, look to run the workflow on [self-hosted runners](https://docs.github.com/en/actions/concepts/runners/self-hosted-runners). Also see [GitHub Action limits](https://docs.github.com/actions/reference/limits) for more info.

## Costs

This creates three AKS clusters and supporting resources. Remember to clean up to avoid ongoing charges.

## Cleanup

Once you have completed your testing, clean up the resources created during this setup to avoid incurring unnecessary costs.

Delete all Azure resources, the service principal, and the GitHub repo created for this sample.

```bash
az group delete --name rg-$RAND-1 -y --no-wait
az group delete --name rg-$RAND-2 -y --no-wait
az group delete --name rg-$RAND-3 -y --no-wait
az group delete --name rg-$RAND-fleet -y --no-wait
az ad sp delete --id $CLIENT_ID
gh repo delete $GITHUB_REPO
```
