
# Use network isolation with managed online endpoints

[!INCLUDE [SDK/CLI v2](../../includes/machine-learning-dev-v2.md)]

When deploying a machine learning model to a managed online endpoint, you can secure communication with the online endpoint by using [private endpoints](../private-link/private-endpoint-overview.md).

You can secure the inbound scoring requests from clients to an _online endpoint_. You can also secure the outbound communications between a _deployment_ and the Azure resources it uses. Security for inbound and outbound communication are configured separately. For more information on endpoints and deployments, see [What are endpoints and deployments](concept-endpoints.md#what-are-endpoints-and-deployments).

The following diagram shows how communications flow through private endpoints to the managed online endpoint. Incoming scoring requests from clients are received through the workspace private endpoint from your virtual network. Outbound communication with services is handled through private endpoints to those service instances from the deployment:

:::image type="content" source="./media/how-to-secure-online-endpoint/endpoint-network-isolation-ingress-egress.png" alt-text="Diagram of overall ingress/egress communication.":::

## Prerequisites

* To use Azure machine learning, you must have an Azure subscription. If you don't have an Azure subscription, create a free account before you begin. Try the [free or paid version of Azure Machine Learning](https://azure.microsoft.com/free/) today.

* You must install and configure the Azure CLI and `ml` extension or the AzureML Python SDK v2. For more information, see the following articles:

    * [Install, set up, and use the CLI (v2)](how-to-configure-cli.md).
    * [Install the Python SDK v2](https://aka.ms/sdk-v2-install).

* You must have an Azure Resource Group, in which you (or the service principal you use) need to have `Contributor` access. You'll have such a resource group if you configured your `ml` extension per the above article.

* You must have an Azure Machine Learning workspace, and the workspace must use a private endpoint. If you don't have one, the steps in this article create an example workspace, VNet, and VM. For more information, see [Configure a private endpoint for Azure Machine Learning workspace](./how-to-configure-private-link.md).

    The workspace configuration can either allow or disallow public network access. If you plan on using managed online endpoint deployments that use __public outbound__, then you must also [configure the workspace to allow public access](how-to-configure-private-link.md#enable-public-access).

    Outbound communication from managed online endpoint deployment is to the _workspace API_. When the endpoint is configured to use __public outbound__, then the workspace must be able to accept that public communication (allow public access).

* When the workspace is configured with a private endpoint, the Azure Container Registry for the workspace must be configured for __Premium__ tier. For more information, see [Azure Container Registry service tiers](../container-registry/container-registry-skus.md).

* The Azure Container Registry and Azure Storage Account must be in the same Azure Resource Group as the workspace.

* If you want to use a [user-assigned managed identity](../active-directory/managed-identities-azure-resources/how-manage-user-assigned-managed-identities.md?pivots=identity-mi-methods-azp) to create and manage online endpoints and online deployments, the identity should have the proper permissions. For details about the required permissions, see [Set up service authentication](./how-to-identity-based-service-authentication.md#workspace). For example, you need to assign the proper RBAC permission for Azure Key Vault on the identity.

> [!IMPORTANT]
> The end-to-end example in this article comes from the files in the __azureml-examples__ GitHub repository. To clone the samples repository and switch to the repository's `cli/` directory, use the following commands: 
>
> ```azurecli
> git clone https://github.com/Azure/azureml-examples
> cd azureml-examples/cli
> ```

## Limitations

* The `v1_legacy_mode` flag must be disabled (false) on your Azure Machine Learning workspace. If this flag is enabled, you won't be able to create a managed online endpoint. For more information, see [Network isolation with v2 API](how-to-configure-network-isolation-with-v2.md).

* If your Azure Machine Learning workspace has a private endpoint that was created before May 24, 2022, you must recreate the workspace's private endpoint before configuring your online endpoints to use a private endpoint. For more information on creating a private endpoint for your workspace, see [How to configure a private endpoint for Azure Machine Learning workspace](how-to-configure-private-link.md).

* Secure outbound communication creates three private endpoints per deployment. One to the Azure Blob storage, one to the Azure Container Registry, and one to your workspace.

* When you use network isolation with a deployment, Azure Log Analytics is partially supported. All metrics and the `AMLOnlineEndpointTrafficLog` table are supported via Azure Log Analytics. `AMLOnlineEndpointConsoleLog` and `AMLOnlineEndpointEventLog` tables are currently not supported. As a workaround, you can use the [az ml online-deployment get_logs](/cli/azure/ml/online-deployment#az-ml-online-deployment-get-logs) CLI command, the [OnlineDeploymentOperations.get_logs()](/python/api/azure-ai-ml/azure.ai.ml.operations.onlinedeploymentoperations#azure-ai-ml-operations-onlinedeploymentoperations-get-logs) Python SDK, or the Deployment log tab in the Azure Machine Learning studio instead. For more information, see [Monitoring online endpoints](how-to-monitor-online-endpoints.md).

* You can configure public access to a __managed online endpoint__ (_inbound_ and _outbound_). You can also configure [public access to an Azure Machine Learning workspace](how-to-configure-private-link.md#enable-public-access).

    Outbound communication from a managed online endpoint deployment is to the _workspace API_. When the endpoint is configured to use __public outbound__, then the workspace must be able to accept that public communication (allow public access).

> [!NOTE]
> Requests to create, update, or retrieve the authentication keys are sent to the Azure Resource Manager over the public network.
 
## Inbound (scoring)

To secure scoring requests to the online endpoint to your virtual network, set the `public_network_access` flag for the endpoint to `disabled`:

# [Azure CLI](#tab/cli)

```azurecli
az ml online-endpoint create -f endpoint.yml --set public_network_access=disabled
```

# [Python](#tab/python)

```python
from azure.ai.ml.entities import ManagedOnlineEndpoint

endpoint = ManagedOnlineEndpoint(name='my-online-endpoint',  
                         description='this is a sample online endpoint', 
                         tags={'foo': 'bar'}, 
                         auth_mode="key", 
                         public_network_access="disabled" 
                         # public_network_access="enabled" 
)
```

# [Studio](#tab/azure-studio)

1. Go to the [Azure Machine Learning studio](https://ml.azure.com).
1. Select the **Workspaces** page from the left navigation bar.
1. Enter a workspace by clicking its name.
1. Select the **Endpoints** page from the left navigation bar.
1. Select **+ Create** to open the **Create deployment** setup wizard.
1. Disable the **Public network access** flag at the **Create endpoint** step.

    :::image type="content" source="media/how-to-secure-online-endpoint/endpoint-disable-public-network-access.png" alt-text="A screenshot of how to disable public network access for an endpoint." lightbox="media/how-to-secure-online-endpoint/endpoint-disable-public-network-access.png":::


When `public_network_access` is `Disabled`, inbound scoring requests are received using the [private endpoint of the Azure Machine Learning workspace](./how-to-configure-private-link.md), and the endpoint can't be reached from public networks.

> [!NOTE]
> You can update (enable or disable) the `public_network_access` flag of an online endpoint after creating it.

## Outbound (resource access)

To restrict communication between a deployment and external resources, including the Azure resources it uses, set the deployment's `egress_public_network_access` flag to `disabled`. Use this flag to ensure that the download of the model, code, and images needed by your deployment are secured with a private endpoint. Note that disabling the flag alone is not enough — your workspace must also have a private link that allows access to Azure resources via a private endpoint. See the [Prerequisites](#prerequisites) for more details.

> [!WARNING]
> You cannot update (enable or disable) the `egress_public_network_access` flag after creating the deployment. Attempting to change the flag while updating the deployment will fail with an error.

> [!NOTE]
> For online deployments with `egress_public_network_access` flag set to `disabled`, access from the deployments to Microsoft Container Registry (MCR) is restricted. If you want to leverage container images from MCR (such as when using curated environment or mlflow no-code deployment), recommendation is to push the images into the Azure Container Registry (ACR) which is attached with the workspace. The images in this ACR is accessible to secured deployments via the private endpoints which are automatically created on behalf of you when you set `egress_public_network_access` flag to `disabled`. For a quick example, please refer to this [custom container example](https://github.com/Azure/azureml-examples/tree/main/cli/endpoints/online/custom-container/minimal/single-model).

# [Azure CLI](#tab/cli)

```azurecli
az ml online-deployment create -f deployment.yml --set egress_public_network_access=disabled
```

# [Python](#tab/python)

```python
blue_deployment = ManagedOnlineDeployment(name='blue', 
                                          endpoint_name='my-online-endpoint', 
                                          model=model, 
                                          code_configuration=CodeConfiguration(code_local_path='./model-1/onlinescoring/',
                                                                               scoring_script='score.py'),
                                          environment=env, 
                                          instance_type='Standard_DS2_v2', 
                                          instance_count=1, 
                                          egress_public_network_access="disabled"
                                          # egress_public_network_access="enabled" 
) 
                              
ml_client.begin_create_or_update(blue_deployment) 
```

# [Studio](#tab/azure-studio)

1. Follow the steps in the **Create deployment** setup wizard to the **Deployment** step.
1. Disable the **Egress public network access** flag.

    :::image type="content" source="media/how-to-secure-online-endpoint/deployment-disable-egress-public-network-access.png" alt-text="A screenshot of how to disable the egress public network access for a deployment." lightbox="media/how-to-secure-online-endpoint/deployment-disable-egress-public-network-access.png":::


The deployment communicates with these resources over the private endpoint:

* The Azure Machine Learning workspace
* The Azure Storage blob that is the default storage for the workspace
* The Azure Container Registry for the workspace

When you configure the `egress_public_network_access` to `disabled`, a new private endpoint is created per deployment, per service. For example, if you set the flag to `disabled` for three deployments to an online endpoint, nine private endpoints are created. Each deployment would have three private endpoints to communicate with the workspace, blob, and container registry.

## Scenarios

The following table lists the supported configurations when configuring inbound and outbound communications for an online endpoint:

| Configuration | Inbound </br> (Endpoint property) | Outbound </br> (Deployment property) | Supported? |
| -------- | -------------------------------- | --------------------------------- | --------- |
| secure inbound with secure outbound | `public_network_access` is disabled | `egress_public_network_access` is disabled   | Yes |
| secure inbound with public outbound | `public_network_access` is disabled</br>The workspace must also allow public access. | `egress_public_network_access` is enabled  | Yes |
| public inbound with secure outbound | `public_network_access` is enabled | `egress_public_network_access` is disabled    | Yes |
| public inbound with public outbound | `public_network_access` is enabled</br>The workspace must also allow public access. | `egress_public_network_access` is enabled  | Yes |

> [!IMPORTANT]
> Outbound communication from managed online endpoint deployment is to the _workspace API_. When the endpoint is configured to use __public outbound__, then the workspace must be able to accept that public communication (allow public access).

## End-to-end example

Use the information in this section to create an example configuration that uses private endpoints to secure online endpoints.

> [!TIP]
> In this example, and Azure Virtual Machine is created inside the VNet. You connect to the VM using SSH, and run the deployment from the VM. This configuration is used to simplify the steps in this example, and does not represent a typical secure configuration. For example, in a production environment you would most likely use a VPN client or Azure ExpressRoute to directly connect clients to the virtual network.

### Create workspace and secured resources

The steps in this section use an Azure Resource Manager template to create the following Azure resources:

* Azure Virtual Network
* Azure Machine Learning workspace
* Azure Container Registry
* Azure Key Vault
* Azure Storage account (blob & file storage)

Public access is disabled for all the services. While the Azure Machine Learning workspace is secured behind a vnet, it's configured to allow public network access. For more information, see [CLI 2.0 secure communications](how-to-configure-cli.md#secure-communications). A scoring subnet is created, along with outbound rules that allow communication with the following Azure services:

* Azure Active Directory
* Azure Resource Manager
* Azure Front Door
* Microsoft Container Registries

The following diagram shows the different components created in this architecture:

The following diagram shows the overall architecture of this example:

:::image type="content" source="./media/how-to-secure-online-endpoint/endpoint-network-isolation-diagram.png" alt-text="Diagram of the services created.":::

To create the resources, use the following Azure CLI commands. To create a resource group. Replace `<my-resource-group>` and `<my-location>` with the desierd values.  

```azurecli
# create resource group
az group create --name <my-resource-group> --location <my-location>
```

Clone the example files for the deployment, use the following command:

```azurecli
#Clone the example files
git clone https://github.com/Azure/azureml-examples
```

To create the resources, use the following Azure CLI commands. Replace `<UNIQUE_SUFFIX>` with a unique suffix for the resources that are created.

```azurecli
az deployment group create --template-file endpoints/online/managed/vnet/setup_ws/main.bicep --parameters suffix=$SUFFIX --resource-group <my-resource-group>
```
### Create the virtual machine jump box

To create an Azure Virtual Machine that can be used to connect to the VNet, use the following command. Replace `<your-new-password>` with the password you want to use when connecting to this VM:

```azurecli
# create vm
az vm create --name test-vm --vnet-name vnet-$SUFFIX --subnet snet-scoring --image UbuntuLTS --admin-username azureuser --admin-password <your-new-password> --resource-group <my-resource-group>
```

> [!IMPORTANT]
> The VM created by these commands has a public endpoint that you can connect to over the public network.

The response from this command is similar to the following JSON document:

```json
{
  "fqdns": "",
  "id": "/subscriptions/<GUID>/resourceGroups/<my-resource-group>/providers/Microsoft.Compute/virtualMachines/test-vm",
  "location": "westus",
  "macAddress": "00-0D-3A-ED-D8-E8",
  "powerState": "VM running",
  "privateIpAddress": "192.168.0.12",
  "publicIpAddress": "20.114.122.77",
  "resourceGroup": "<my-resource-group>",
  "zones": ""
}
```

Use the following command to connect to the VM using SSH. Replace `publicIpAddress` with the value of the public IP address in the response from the previous command:

```azurecli
ssh azureusere@publicIpAddress
```

When prompted, enter the password you used when creating the VM.

### Configure the VM

1. Use the following commands from the SSH session to install the CLI and Docker:

```azurecli
### Part of automated testing: only required when this script is called via vm run-command invoke inorder to gather the parameters ###
set -e
for args in "$@"
do
    keyname=$(echo $args | cut -d ':' -f 1)
    result=$(echo $args | cut -d ':' -f 2)
    export $keyname=$result
done

# $USER is no set when used from az vm run-command
export USER=$(whoami)

# <setup_docker_az_cli> 
# setup docker
sudo apt-get update -y && sudo apt install docker.io -y && sudo snap install docker && docker --version && sudo usermod -aG docker $USER
# setup az cli and ml extension
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash && az extension add --upgrade -n ml -y
# </setup_docker_az_cli> 

# login using az cli. 
### NOTE to user: use `az login` - and do NOT use the below command (it requires setting up of user assigned identity). ###
az login --identity -u /subscriptions/$SUBSCRIPTION/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.ManagedIdentity/userAssignedIdentities/$IDENTITY_NAME

# <configure_defaults> 
# configure cli defaults
az account set --subscription $SUBSCRIPTION
az configure --defaults group=$RESOURCE_GROUP workspace=$WORKSPACE location=$LOCATION
# </configure_defaults> 

# Clone the samples repo. This is needed to build the image and create the managed online deployment.
# Note: We will hardcode the below line in the docs (without GIT_BRANCH) so we don't need to explain the logic to the user.
sudo mkdir -p /home/samples; sudo git clone -b $GIT_BRANCH --depth 1 https://github.com/Azure/azureml-examples.git /home/samples/azureml-examples -q

```

1. To create the environment variables used by this example, run the following commands. Replace `<YOUR_SUBSCRIPTION_ID>` with your Azure subscription ID. Replace `<YOUR_RESOURCE_GROUP>` with the resource group that contains your workspace. Replace `<SUFFIX_USED_IN_SETUP>` with the suffix you provided earlier. Replace `<LOCATION>` with the location of your Azure workspace. Replace `<YOUR_ENDPOINT_NAME>` with the name to use for the endpoint.

    > [!TIP]
    > Use the tabs to select whether you want to perform a deployment using an MLflow model or generic ML model.

    # [Generic model](#tab/model)

```azurecli
#!/bin/bash

set -e

# This is the instructions for docs.User has to execute this from a test VM - that is why user cannot use defaults from their local setup


# <set_env_vars> 
export SUBSCRIPTION="<YOUR_SUBSCRIPTION_ID>"
export RESOURCE_GROUP="<YOUR_RESOURCE_GROUP>"
export LOCATION="<LOCATION>"

# SUFFIX that was used when creating the workspace resources. Alternatively the resource names can be looked up from the resource group after the vnet setup script has completed.
export SUFFIX="<SUFFIX_USED_IN_SETUP>"

# SUFFIX used during the initial setup. Alternatively the resource names can be looked up from the resource group after the  setup script has completed.
export WORKSPACE=mlw-$SUFFIX
export ACR_NAME=cr$SUFFIX

# provide a unique name for the endpoint
export ENDPOINT_NAME="<YOUR_ENDPOINT_NAME>"

# name of the image that will be built for this sample and pushed into acr - no need to change this
export IMAGE_NAME="img"

# Yaml files that will be used to create endpoint and deployment. These are relative to azureml-examples/cli/ directory. Do not change these
export ENDPOINT_FILE_PATH="endpoints/online/managed/vnet/sample/endpoint.yml"
export DEPLOYMENT_FILE_PATH="endpoints/online/managed/vnet/sample/blue-deployment-vnet.yml"
export SAMPLE_REQUEST_PATH="endpoints/online/managed/vnet/sample/sample-request.json"
export ENV_DIR_PATH="endpoints/online/managed/vnet/sample/environment"
# </set_env_vars>

export SUFFIX="mevnet" # used during setup of secure vnet workspace: setup/setup-repo/azure-github.sh
export SUBSCRIPTION=$(az account show --query "id" -o tsv)
export RESOURCE_GROUP=$(az configure -l --query "[?name=='group'].value" -o tsv)
export LOCATION=$(az configure -l --query "[?name=='location'].value" -o tsv)
# remove all whitespace from location
export LOCATION="$(echo -e "${LOCATION}" | tr -d '[:space:]')"
export IDENTITY_NAME=uai$SUFFIX
# export ACR_NAME=cr$SUFFIX
export WORKSPACE=mlw-$SUFFIX
export ENDPOINT_NAME=$ENDPOINT_NAME
# VM name used during creation: endpoints/online/managed/vnet/setup_vm/vm-main.bicep
export VM_NAME="moevnet-vm"
# VNET name and subnet name used during vnet worskapce setup: endpoints/online/managed/vnet/setup_ws/main.bicep
export VNET_NAME=vnet-$SUFFIX
export SUBNET_NAME="snet-scoring"
export ENDPOINT_NAME=endpt-vnet-`echo $RANDOM`

# Get the current branch name of the azureml-examples. Useful in PR scenario. Since the sample code is cloned and executed from a VM, we need to pass the branch name when running az vm run-command
# If running from local machine, change it to your branch name
export GIT_BRANCH=$GITHUB_HEAD_REF
# need to set branch name manually if executed from main
if [ "$GIT_BRANCH" == "" ];
then
   GIT_BRANCH="main"
fi

# We use a different workspace for managed vnet endpoints
az configure --defaults workspace=$WORKSPACE

export ACR_NAME=$(az ml workspace show -n $WORKSPACE --query container_registry -o tsv | cut -d'/' -f9-)
if [[ -z "$ACRNAME" ]]
then
    export ACR_NAME=$(az acr list --query '[].{Name:name}' --output tsv)
fi

### setup VM & deploy/test ###
# if vm exists, wait for 15 mins before trying to delete
export VM_EXISTS=$(az vm list -o tsv --query "[?name=='$VM_NAME'].name")
if [ "$VM_EXISTS" != "" ];
then
   echo "VM already exists from previous run. Waiting for 15 mins before deleting."
   sleep 15m
   az vm delete -n $VM_NAME -y
fi

# Create the VM. In the docs we will provide instructions to create a VM using az vm create -n $VM_NAME
az deployment group create --name $VM_NAME-$ENDPOINT_NAME --template-file endpoints/online/managed/vnet/setup_vm/vm-main.bicep --parameters vmName=$VM_NAME identityName=$IDENTITY_NAME vnetName=$VNET_NAME subnetName=$SUBNET_NAME

az vm run-command invoke -n $VM_NAME --command-id RunShellScript --scripts @endpoints/online/managed/vnet/setup_vm/scripts/vmsetup.sh --parameters "SUBSCRIPTION:$SUBSCRIPTION" "RESOURCE_GROUP:$RESOURCE_GROUP" "LOCATION:$LOCATION" "IDENTITY_NAME:$IDENTITY_NAME" "GIT_BRANCH:$GIT_BRANCH"

# build image
az vm run-command invoke -n $VM_NAME --command-id RunShellScript --scripts @endpoints/online/managed/vnet/setup_vm/scripts/build_image.sh --parameters "SUBSCRIPTION:$SUBSCRIPTION" "RESOURCE_GROUP:$RESOURCE_GROUP" "LOCATION:$LOCATION" "IDENTITY_NAME:$IDENTITY_NAME" "ACR_NAME=$ACR_NAME" "IMAGE_NAME:$IMAGE_NAME" "ENV_DIR_PATH:$ENV_DIR_PATH"

# create endpoint/deployment inside managed vnet
az vm run-command invoke -n $VM_NAME --command-id RunShellScript --scripts @endpoints/online/managed/vnet/setup_vm/scripts/create_moe.sh --parameters "SUBSCRIPTION:$SUBSCRIPTION" "RESOURCE_GROUP:$RESOURCE_GROUP" "LOCATION:$LOCATION" "IDENTITY_NAME:$IDENTITY_NAME" "WORKSPACE:$WORKSPACE" "ENDPOINT_NAME:$ENDPOINT_NAME" "ACR_NAME=$ACR_NAME" "IMAGE_NAME:$IMAGE_NAME" "ENDPOINT_FILE_PATH:$ENDPOINT_FILE_PATH" "DEPLOYMENT_FILE_PATH:$DEPLOYMENT_FILE_PATH" "SAMPLE_REQUEST_PATH:$SAMPLE_REQUEST_PATH"

# test the endpoint by scoring it
export CMD_OUTPUT=$(az vm run-command invoke -n $VM_NAME --command-id RunShellScript --scripts @endpoints/online/managed/vnet/setup_vm/scripts/score_endpoint.sh --parameters "SUBSCRIPTION:$SUBSCRIPTION" "RESOURCE_GROUP:$RESOURCE_GROUP" "LOCATION:$LOCATION" "IDENTITY_NAME:$IDENTITY_NAME" "WORKSPACE:$WORKSPACE" "ENDPOINT_NAME:$ENDPOINT_NAME" "SAMPLE_REQUEST_PATH:$SAMPLE_REQUEST_PATH")

# the scoring output for sample request should be [11055.977245525679, 4503.079536107787]. We are validating if part of the number is available in the output (not comparing all the decimals to accomodate rounding discrepencies)
if [[ $CMD_OUTPUT =~ "11055" ]]; then
   echo "Scoring works!"
else
   echo "Error in scoring"
   # delete the VM before exiting with error
   az vm delete -n $VM_NAME -y --no-wait
   # exit with error
   exit 1
fi


### Cleanup
# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes --no-wait
# </delete_endpoint>
# <delete_vm> 
az vm delete -n $VM_NAME -y --no-wait
# </delete_vm> 
```

    # [MLflow model](#tab/mlflow)

```azurecli
#!/bin/bash

set -e

# This is the instructions for docs.User has to execute this from a test VM - that is why user cannot use defaults from their local setup

# <set_env_vars> 
export SUBSCRIPTION="<YOUR_SUBSCRIPTION_ID>"
export RESOURCE_GROUP="<YOUR_RESOURCE_GROUP>"
export LOCATION="<LOCATION>"

# SUFFIX that was used when creating the workspace resources. Alternatively the resource names can be looked up from the resource group after the vnet setup script has completed.
export SUFFIX="<SUFFIX_USED_IN_SETUP>"

# SUFFIX used during the initial setup. Alternatively the resource names can be looked up from the resource group after the  setup script has completed.
export WORKSPACE=mlw-$SUFFIX
export ACR_NAME=cr$SUFFIX

# provide a unique name for the endpoint
export ENDPOINT_NAME="<YOUR_ENDPOINT_NAME>"

# name of the image that will be built for this sample and pushed into acr - no need to change this
export IMAGE_NAME="mlflow"

# Yaml files that will be used to create endpoint and deployment. These are relative to azureml-examples/cli/ directory. Do not change these
export ENDPOINT_FILE_PATH="endpoints/online/managed/vnet/mlflow/endpoint.yml"
export DEPLOYMENT_FILE_PATH="endpoints/online/managed/vnet/mlflow/blue-deployment-vnet.yml"
export SAMPLE_REQUEST_PATH="endpoints/online/managed/vnet/mlflow/sample-request.json"
export ENV_DIR_PATH="endpoints/online/managed/vnet/mlflow/environment"
# </set_env_vars>

export SUFFIX="mevnet" # used during setup of secure vnet workspace: setup/setup-repo/azure-github.sh
export SUBSCRIPTION=$(az account show --query "id" -o tsv)
export RESOURCE_GROUP=$(az configure -l --query "[?name=='group'].value" -o tsv)
export LOCATION=$(az configure -l --query "[?name=='location'].value" -o tsv)
# remove all whitespace from location
export LOCATION="$(echo -e "${LOCATION}" | tr -d '[:space:]')"
export IDENTITY_NAME=uai$SUFFIX
export WORKSPACE=mlw-$SUFFIX
export ENDPOINT_NAME=$ENDPOINT_NAME
# VM name used during creation: endpoints/online/managed/vnet/setup_vm/vm-main.bicep
export VM_NAME="moevnet-mlflow-vm"
# VNET name and subnet name used during vnet worskapce setup: endpoints/online/managed/vnet/setup_ws/main.bicep
export VNET_NAME=vnet-$SUFFIX
export SUBNET_NAME="snet-scoring"
export ENDPOINT_NAME=endpt-vnet-mlflow-`echo $RANDOM`

# Get the current branch name of the azureml-examples. Useful in PR scenario. Since the sample code is cloned and executed from a VM, we need to pass the branch name when running az vm run-command
# If running from local machine, change it to your branch name
export GIT_BRANCH=$GITHUB_HEAD_REF
# need to set branch name manually if executed from main
if [ "$GIT_BRANCH" == "" ];
then
   GIT_BRANCH="main"
fi

# We use a different workspace for managed vnet endpoints
az configure --defaults workspace=$WORKSPACE

export ACR_NAME=$(az ml workspace show -n $WORKSPACE --query container_registry -o tsv | cut -d'/' -f9-)
if [[ -z "$ACRNAME" ]]
then
    export ACR_NAME=$(az acr list --query '[].{Name:name}' --output tsv)
fi

### setup VM & deploy/test ###
# if vm exists, wait for 15 mins before trying to delete
export VM_EXISTS=$(az vm list -o tsv --query "[?name=='$VM_NAME'].name")
if [ "$VM_EXISTS" != "" ];
then
   echo "VM already exists from previous run. Waiting for 15 mins before deleting."
   sleep 15m
   az vm delete -n $VM_NAME -y
fi

# Create the VM. In the docs we will provide instructions to create a VM using az vm create -n $VM_NAME
az deployment group create  --name $VM_NAME-$ENDPOINT_NAME --template-file endpoints/online/managed/vnet/setup_vm/vm-main.bicep --parameters vmName=$VM_NAME identityName=$IDENTITY_NAME vnetName=$VNET_NAME subnetName=$SUBNET_NAME

# command in script: az deployment group create --template-file endpoints/online/managed/vnet/setup/vm_main.bicep #identity name is hardcoded uai-identity 
az vm run-command invoke -n $VM_NAME --command-id RunShellScript --scripts @endpoints/online/managed/vnet/setup_vm/scripts/vmsetup.sh --parameters "SUBSCRIPTION:$SUBSCRIPTION" "RESOURCE_GROUP:$RESOURCE_GROUP" "LOCATION:$LOCATION" "IDENTITY_NAME:$IDENTITY_NAME" "GIT_BRANCH:$GIT_BRANCH"

# build image
az vm run-command invoke -n $VM_NAME --command-id RunShellScript --scripts @endpoints/online/managed/vnet/setup_vm/scripts/build_image.sh --parameters "SUBSCRIPTION:$SUBSCRIPTION" "RESOURCE_GROUP:$RESOURCE_GROUP" "LOCATION:$LOCATION" "IDENTITY_NAME:$IDENTITY_NAME" "ACR_NAME=$ACR_NAME" "IMAGE_NAME:$IMAGE_NAME" "ENV_DIR_PATH:$ENV_DIR_PATH"

# create endpoint/deployment inside managed vnet
az vm run-command invoke -n $VM_NAME --command-id RunShellScript --scripts @endpoints/online/managed/vnet/setup_vm/scripts/create_moe.sh --parameters "SUBSCRIPTION:$SUBSCRIPTION" "RESOURCE_GROUP:$RESOURCE_GROUP" "LOCATION:$LOCATION" "IDENTITY_NAME:$IDENTITY_NAME" "WORKSPACE:$WORKSPACE" "ENDPOINT_NAME:$ENDPOINT_NAME" "ACR_NAME=$ACR_NAME" "IMAGE_NAME:$IMAGE_NAME" "ENDPOINT_FILE_PATH:$ENDPOINT_FILE_PATH" "DEPLOYMENT_FILE_PATH:$DEPLOYMENT_FILE_PATH" "SAMPLE_REQUEST_PATH:$SAMPLE_REQUEST_PATH"

# test the endpoint by scoring it
export CMD_OUTPUT=$(az vm run-command invoke -n $VM_NAME --command-id RunShellScript --scripts @endpoints/online/managed/vnet/setup_vm/scripts/score_endpoint.sh --parameters "SUBSCRIPTION:$SUBSCRIPTION" "RESOURCE_GROUP:$RESOURCE_GROUP" "LOCATION:$LOCATION" "IDENTITY_NAME:$IDENTITY_NAME" "WORKSPACE:$WORKSPACE" "ENDPOINT_NAME:$ENDPOINT_NAME" "SAMPLE_REQUEST_PATH:$SAMPLE_REQUEST_PATH")

# the scoring output for sample request should be [6141.267272547523, 6407.1333176127255]. We are validating if part of the number is available in the output (not comparing all the decimals to accomodate rounding discrepencies)
if [[ $CMD_OUTPUT =~ "6141" ]]; then
   echo "Scoring works!"
else
   echo "Error in scoring"
   # delete the VM before exiting with error
   az vm delete -n $VM_NAME -y --no-wait
   # exit with error
   exit 1
fi


### Cleanup
# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes --no-wait
# </delete_endpoint>
# <delete_vm> 
az vm delete -n $VM_NAME -y --no-wait
# </delete_vm> 
```


1. To sign in to the Azure CLI in the VM environment, use the following command:

```azurecli
### TESTED

# <az_extension_list>
az extension list 
# </az_extension_list>

# <az_ml_install>
az extension add -n ml
# </az_ml_install>

# <list_defaults>
az configure -l -o table
# </list_defaults>

apt-get install sudo
# <az_extension_install_linux>
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash 
az extension add -n ml -y 
# </az_extension_install_linux>

# <az_ml_update>
az extension update -n ml
# </az_ml_update>

# <az_ml_verify>
az ml -h
# </az_ml_verify>

# <az_extension_remove>
az extension remove -n azure-cli-ml
az extension remove -n ml
# </az_extension_remove>

# <git_clone>
git clone --depth 1 https://github.com/Azure/azureml-examples
cd azureml-examples/cli
# </git_clone>

# <az_version>
az version
# </az_version>

### UNTESTED
exit

# <az_account_set>
az account set -s "<YOUR_SUBSCRIPTION_NAME_OR_ID>"
# </az_account_set>

# <az_login>
az login
# </az_login>

```

1. To configure the defaults for the CLI, use the following commands:

```azurecli
### Part of automated testing: only required when this script is called via vm run-command invoke inorder to gather the parameters ###
set -e
for args in "$@"
do
    keyname=$(echo $args | cut -d ':' -f 1)
    result=$(echo $args | cut -d ':' -f 2)
    export $keyname=$result
done

# $USER is no set when used from az vm run-command
export USER=$(whoami)

# <setup_docker_az_cli> 
# setup docker
sudo apt-get update -y && sudo apt install docker.io -y && sudo snap install docker && docker --version && sudo usermod -aG docker $USER
# setup az cli and ml extension
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash && az extension add --upgrade -n ml -y
# </setup_docker_az_cli> 

# login using az cli. 
### NOTE to user: use `az login` - and do NOT use the below command (it requires setting up of user assigned identity). ###
az login --identity -u /subscriptions/$SUBSCRIPTION/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.ManagedIdentity/userAssignedIdentities/$IDENTITY_NAME

# <configure_defaults> 
# configure cli defaults
az account set --subscription $SUBSCRIPTION
az configure --defaults group=$RESOURCE_GROUP workspace=$WORKSPACE location=$LOCATION
# </configure_defaults> 

# Clone the samples repo. This is needed to build the image and create the managed online deployment.
# Note: We will hardcode the below line in the docs (without GIT_BRANCH) so we don't need to explain the logic to the user.
sudo mkdir -p /home/samples; sudo git clone -b $GIT_BRANCH --depth 1 https://github.com/Azure/azureml-examples.git /home/samples/azureml-examples -q

```

1. To clone the example files for the deployment, use the following command:

    ```azurecli
    sudo mkdir -p /home/samples; sudo git clone -b main --depth 1 https://github.com/Azure/azureml-examples.git /home/samples/azureml-examples
    ```

1. To build a custom docker image to use with the deployment, use the following commands:

```azurecli
set -e
### Part of automated testing: only required when this script is called via vm run-command invoke inorder to gather the parameters ###
for args in "$@"
do
    keyname=$(echo $args | cut -d ':' -f 1)
    result=$(echo $args | cut -d ':' -f 2)
    export $keyname=$result
done

# login using the user assigned identity. 
az login --identity -u /subscriptions/$SUBSCRIPTION/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.ManagedIdentity/userAssignedIdentities/$IDENTITY_NAME
az account set --subscription $SUBSCRIPTION
az configure --defaults group=$RESOURCE_GROUP workspace=$WORKSPACE location=$LOCATION

# <build_image> 
# Navigate to the samples
cd /home/samples/azureml-examples/cli/$ENV_DIR_PATH
# login to acr. Optionally, to avoid using sudo, complete the docker post install steps: https://docs.docker.com/engine/install/linux-postinstall/
sudo az acr login -n "$ACR_NAME"
# Build the docker image with the sample docker file
sudo docker build -t "$ACR_NAME.azurecr.io/repo/$IMAGE_NAME":v1 .
# push the image to the ACR
sudo docker push "$ACR_NAME.azurecr.io/repo/$IMAGE_NAME":v1
# check if the image exists in acr
az acr repository show -n "$ACR_NAME" --repository "repo/$IMAGE_NAME"
# </build_image> 
```

    > [!TIP]
    > In this example, we build the Docker image before pushing it to Azure Container Registry. Alternatively, you can build the image in your vnet by using an Azure Machine Learning compute cluster and environments. For more information, see [Secure Azure Machine Learning workspace](how-to-secure-workspace-vnet.md#enable-azure-container-registry-acr).

### Create a secured managed online endpoint

1. To create a managed online endpoint that is secured using a private endpoint for inbound and outbound communication, use the following commands:

    > [!TIP]
    > You can test or debug the Docker image locally by using the `--local` flag when creating the deployment. For more information, see the [Deploy and debug locally](how-to-deploy-online-endpoints.md#deploy-and-debug-locally-by-using-local-endpoints) article.

```azurecli
set -e
### Part of automated testing: only required when this script is called via vm run-command invoke inorder to gather the parameters ###
for args in "$@"
do
    keyname=$(echo $args | cut -d ':' -f 1)
    result=$(echo $args | cut -d ':' -f 2)
    export $keyname=$result
done

# login using the user assigned identity. 
az login --identity -u /subscriptions/$SUBSCRIPTION/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.ManagedIdentity/userAssignedIdentities/$IDENTITY_NAME
az account set --subscription $SUBSCRIPTION
az configure --defaults group=$RESOURCE_GROUP workspace=$WORKSPACE location=$LOCATION

# <create_local_deployment> 
# navigate to the cli directory in the azurem-examples repo
cd /home/samples/azureml-examples/cli/

# create endpoint
az ml online-endpoint create --local --name $ENDPOINT_NAME -f $ENDPOINT_FILE_PATH --set public_network_access="disabled"
# create deployment in managed vnet
az ml online-deployment create --local --name blue --endpoint $ENDPOINT_NAME -f $DEPLOYMENT_FILE_PATH --all-traffic --set environment.image="$ACR_NAME.azurecr.io/repo/$IMAGE_NAME:v1" egress_public_network_access="disabled"
# check if scoring works
az ml online-endpoint invoke --local --name $ENDPOINT_NAME --request-file $SAMPLE_REQUEST_PATH
# </create_local_deployment> 

# <create_vnet_deployment> 
# navigate to the cli directory in the azurem-examples repo
cd /home/samples/azureml-examples/cli/

# create endpoint
az ml online-endpoint create --name $ENDPOINT_NAME -f $ENDPOINT_FILE_PATH --set public_network_access="disabled"
# create deployment in managed vnet
az ml online-deployment create --name blue --endpoint $ENDPOINT_NAME -f $DEPLOYMENT_FILE_PATH --all-traffic --set environment.image="$ACR_NAME.azurecr.io/repo/$IMAGE_NAME:v1" egress_public_network_access="disabled"
# </create_vnet_deployment> 

# <get_logs> 
az ml online-deployment get-logs -n blue --endpoint $ENDPOINT_NAME
# </get_logs>

# check if scoring works
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file $SAMPLE_REQUEST_PATH


```


1. To make a scoring request with the endpoint, use the following commands:

```azurecli
set -e
### Part of automated testing: only required when this script is called via vm run-command invoke inorder to gather the parameters ###
for args in "$@"
do
    keyname=$(echo $args | cut -d ':' -f 1)
    result=$(echo $args | cut -d ':' -f 2)
    export $keyname=$result
done

# login using the user assigned identity. 
az login --identity -u /subscriptions/$SUBSCRIPTION/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.ManagedIdentity/userAssignedIdentities/$IDENTITY_NAME
az account set --subscription $SUBSCRIPTION
az configure --defaults group=$RESOURCE_GROUP workspace=$WORKSPACE location=$LOCATION


# navigate to the samples directory
cd /home/samples/azureml-examples/cli/

# <check_deployment> 
# Try scoring using the CLI
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file $SAMPLE_REQUEST_PATH

# Try scoring using curl
ENDPOINT_KEY=$(az ml online-endpoint get-credentials -n $ENDPOINT_NAME -o tsv --query primaryKey)
SCORING_URI=$(az ml online-endpoint show -n $ENDPOINT_NAME -o tsv --query scoring_uri)
curl --request POST "$SCORING_URI" --header "Authorization: Bearer $ENDPOINT_KEY" --header 'Content-Type: application/json' --data @$SAMPLE_REQUEST_PATH
# </check_deployment> 


```

### Cleanup

To delete the endpoint, use the following command:

```azurecli
#!/bin/bash

set -e

# This is the instructions for docs.User has to execute this from a test VM - that is why user cannot use defaults from their local setup


# <set_env_vars> 
export SUBSCRIPTION="<YOUR_SUBSCRIPTION_ID>"
export RESOURCE_GROUP="<YOUR_RESOURCE_GROUP>"
export LOCATION="<LOCATION>"

# SUFFIX that was used when creating the workspace resources. Alternatively the resource names can be looked up from the resource group after the vnet setup script has completed.
export SUFFIX="<SUFFIX_USED_IN_SETUP>"

# SUFFIX used during the initial setup. Alternatively the resource names can be looked up from the resource group after the  setup script has completed.
export WORKSPACE=mlw-$SUFFIX
export ACR_NAME=cr$SUFFIX

# provide a unique name for the endpoint
export ENDPOINT_NAME="<YOUR_ENDPOINT_NAME>"

# name of the image that will be built for this sample and pushed into acr - no need to change this
export IMAGE_NAME="img"

# Yaml files that will be used to create endpoint and deployment. These are relative to azureml-examples/cli/ directory. Do not change these
export ENDPOINT_FILE_PATH="endpoints/online/managed/vnet/sample/endpoint.yml"
export DEPLOYMENT_FILE_PATH="endpoints/online/managed/vnet/sample/blue-deployment-vnet.yml"
export SAMPLE_REQUEST_PATH="endpoints/online/managed/vnet/sample/sample-request.json"
export ENV_DIR_PATH="endpoints/online/managed/vnet/sample/environment"
# </set_env_vars>

export SUFFIX="mevnet" # used during setup of secure vnet workspace: setup/setup-repo/azure-github.sh
export SUBSCRIPTION=$(az account show --query "id" -o tsv)
export RESOURCE_GROUP=$(az configure -l --query "[?name=='group'].value" -o tsv)
export LOCATION=$(az configure -l --query "[?name=='location'].value" -o tsv)
# remove all whitespace from location
export LOCATION="$(echo -e "${LOCATION}" | tr -d '[:space:]')"
export IDENTITY_NAME=uai$SUFFIX
# export ACR_NAME=cr$SUFFIX
export WORKSPACE=mlw-$SUFFIX
export ENDPOINT_NAME=$ENDPOINT_NAME
# VM name used during creation: endpoints/online/managed/vnet/setup_vm/vm-main.bicep
export VM_NAME="moevnet-vm"
# VNET name and subnet name used during vnet worskapce setup: endpoints/online/managed/vnet/setup_ws/main.bicep
export VNET_NAME=vnet-$SUFFIX
export SUBNET_NAME="snet-scoring"
export ENDPOINT_NAME=endpt-vnet-`echo $RANDOM`

# Get the current branch name of the azureml-examples. Useful in PR scenario. Since the sample code is cloned and executed from a VM, we need to pass the branch name when running az vm run-command
# If running from local machine, change it to your branch name
export GIT_BRANCH=$GITHUB_HEAD_REF
# need to set branch name manually if executed from main
if [ "$GIT_BRANCH" == "" ];
then
   GIT_BRANCH="main"
fi

# We use a different workspace for managed vnet endpoints
az configure --defaults workspace=$WORKSPACE

export ACR_NAME=$(az ml workspace show -n $WORKSPACE --query container_registry -o tsv | cut -d'/' -f9-)
if [[ -z "$ACRNAME" ]]
then
    export ACR_NAME=$(az acr list --query '[].{Name:name}' --output tsv)
fi

### setup VM & deploy/test ###
# if vm exists, wait for 15 mins before trying to delete
export VM_EXISTS=$(az vm list -o tsv --query "[?name=='$VM_NAME'].name")
if [ "$VM_EXISTS" != "" ];
then
   echo "VM already exists from previous run. Waiting for 15 mins before deleting."
   sleep 15m
   az vm delete -n $VM_NAME -y
fi

# Create the VM. In the docs we will provide instructions to create a VM using az vm create -n $VM_NAME
az deployment group create --name $VM_NAME-$ENDPOINT_NAME --template-file endpoints/online/managed/vnet/setup_vm/vm-main.bicep --parameters vmName=$VM_NAME identityName=$IDENTITY_NAME vnetName=$VNET_NAME subnetName=$SUBNET_NAME

az vm run-command invoke -n $VM_NAME --command-id RunShellScript --scripts @endpoints/online/managed/vnet/setup_vm/scripts/vmsetup.sh --parameters "SUBSCRIPTION:$SUBSCRIPTION" "RESOURCE_GROUP:$RESOURCE_GROUP" "LOCATION:$LOCATION" "IDENTITY_NAME:$IDENTITY_NAME" "GIT_BRANCH:$GIT_BRANCH"

# build image
az vm run-command invoke -n $VM_NAME --command-id RunShellScript --scripts @endpoints/online/managed/vnet/setup_vm/scripts/build_image.sh --parameters "SUBSCRIPTION:$SUBSCRIPTION" "RESOURCE_GROUP:$RESOURCE_GROUP" "LOCATION:$LOCATION" "IDENTITY_NAME:$IDENTITY_NAME" "ACR_NAME=$ACR_NAME" "IMAGE_NAME:$IMAGE_NAME" "ENV_DIR_PATH:$ENV_DIR_PATH"

# create endpoint/deployment inside managed vnet
az vm run-command invoke -n $VM_NAME --command-id RunShellScript --scripts @endpoints/online/managed/vnet/setup_vm/scripts/create_moe.sh --parameters "SUBSCRIPTION:$SUBSCRIPTION" "RESOURCE_GROUP:$RESOURCE_GROUP" "LOCATION:$LOCATION" "IDENTITY_NAME:$IDENTITY_NAME" "WORKSPACE:$WORKSPACE" "ENDPOINT_NAME:$ENDPOINT_NAME" "ACR_NAME=$ACR_NAME" "IMAGE_NAME:$IMAGE_NAME" "ENDPOINT_FILE_PATH:$ENDPOINT_FILE_PATH" "DEPLOYMENT_FILE_PATH:$DEPLOYMENT_FILE_PATH" "SAMPLE_REQUEST_PATH:$SAMPLE_REQUEST_PATH"

# test the endpoint by scoring it
export CMD_OUTPUT=$(az vm run-command invoke -n $VM_NAME --command-id RunShellScript --scripts @endpoints/online/managed/vnet/setup_vm/scripts/score_endpoint.sh --parameters "SUBSCRIPTION:$SUBSCRIPTION" "RESOURCE_GROUP:$RESOURCE_GROUP" "LOCATION:$LOCATION" "IDENTITY_NAME:$IDENTITY_NAME" "WORKSPACE:$WORKSPACE" "ENDPOINT_NAME:$ENDPOINT_NAME" "SAMPLE_REQUEST_PATH:$SAMPLE_REQUEST_PATH")

# the scoring output for sample request should be [11055.977245525679, 4503.079536107787]. We are validating if part of the number is available in the output (not comparing all the decimals to accomodate rounding discrepencies)
if [[ $CMD_OUTPUT =~ "11055" ]]; then
   echo "Scoring works!"
else
   echo "Error in scoring"
   # delete the VM before exiting with error
   az vm delete -n $VM_NAME -y --no-wait
   # exit with error
   exit 1
fi


### Cleanup
# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes --no-wait
# </delete_endpoint>
# <delete_vm> 
az vm delete -n $VM_NAME -y --no-wait
# </delete_vm> 
```

To delete the VM, use the following command:

```azurecli
#!/bin/bash

set -e

# This is the instructions for docs.User has to execute this from a test VM - that is why user cannot use defaults from their local setup


# <set_env_vars> 
export SUBSCRIPTION="<YOUR_SUBSCRIPTION_ID>"
export RESOURCE_GROUP="<YOUR_RESOURCE_GROUP>"
export LOCATION="<LOCATION>"

# SUFFIX that was used when creating the workspace resources. Alternatively the resource names can be looked up from the resource group after the vnet setup script has completed.
export SUFFIX="<SUFFIX_USED_IN_SETUP>"

# SUFFIX used during the initial setup. Alternatively the resource names can be looked up from the resource group after the  setup script has completed.
export WORKSPACE=mlw-$SUFFIX
export ACR_NAME=cr$SUFFIX

# provide a unique name for the endpoint
export ENDPOINT_NAME="<YOUR_ENDPOINT_NAME>"

# name of the image that will be built for this sample and pushed into acr - no need to change this
export IMAGE_NAME="img"

# Yaml files that will be used to create endpoint and deployment. These are relative to azureml-examples/cli/ directory. Do not change these
export ENDPOINT_FILE_PATH="endpoints/online/managed/vnet/sample/endpoint.yml"
export DEPLOYMENT_FILE_PATH="endpoints/online/managed/vnet/sample/blue-deployment-vnet.yml"
export SAMPLE_REQUEST_PATH="endpoints/online/managed/vnet/sample/sample-request.json"
export ENV_DIR_PATH="endpoints/online/managed/vnet/sample/environment"
# </set_env_vars>

export SUFFIX="mevnet" # used during setup of secure vnet workspace: setup/setup-repo/azure-github.sh
export SUBSCRIPTION=$(az account show --query "id" -o tsv)
export RESOURCE_GROUP=$(az configure -l --query "[?name=='group'].value" -o tsv)
export LOCATION=$(az configure -l --query "[?name=='location'].value" -o tsv)
# remove all whitespace from location
export LOCATION="$(echo -e "${LOCATION}" | tr -d '[:space:]')"
export IDENTITY_NAME=uai$SUFFIX
# export ACR_NAME=cr$SUFFIX
export WORKSPACE=mlw-$SUFFIX
export ENDPOINT_NAME=$ENDPOINT_NAME
# VM name used during creation: endpoints/online/managed/vnet/setup_vm/vm-main.bicep
export VM_NAME="moevnet-vm"
# VNET name and subnet name used during vnet worskapce setup: endpoints/online/managed/vnet/setup_ws/main.bicep
export VNET_NAME=vnet-$SUFFIX
export SUBNET_NAME="snet-scoring"
export ENDPOINT_NAME=endpt-vnet-`echo $RANDOM`

# Get the current branch name of the azureml-examples. Useful in PR scenario. Since the sample code is cloned and executed from a VM, we need to pass the branch name when running az vm run-command
# If running from local machine, change it to your branch name
export GIT_BRANCH=$GITHUB_HEAD_REF
# need to set branch name manually if executed from main
if [ "$GIT_BRANCH" == "" ];
then
   GIT_BRANCH="main"
fi

# We use a different workspace for managed vnet endpoints
az configure --defaults workspace=$WORKSPACE

export ACR_NAME=$(az ml workspace show -n $WORKSPACE --query container_registry -o tsv | cut -d'/' -f9-)
if [[ -z "$ACRNAME" ]]
then
    export ACR_NAME=$(az acr list --query '[].{Name:name}' --output tsv)
fi

### setup VM & deploy/test ###
# if vm exists, wait for 15 mins before trying to delete
export VM_EXISTS=$(az vm list -o tsv --query "[?name=='$VM_NAME'].name")
if [ "$VM_EXISTS" != "" ];
then
   echo "VM already exists from previous run. Waiting for 15 mins before deleting."
   sleep 15m
   az vm delete -n $VM_NAME -y
fi

# Create the VM. In the docs we will provide instructions to create a VM using az vm create -n $VM_NAME
az deployment group create --name $VM_NAME-$ENDPOINT_NAME --template-file endpoints/online/managed/vnet/setup_vm/vm-main.bicep --parameters vmName=$VM_NAME identityName=$IDENTITY_NAME vnetName=$VNET_NAME subnetName=$SUBNET_NAME

az vm run-command invoke -n $VM_NAME --command-id RunShellScript --scripts @endpoints/online/managed/vnet/setup_vm/scripts/vmsetup.sh --parameters "SUBSCRIPTION:$SUBSCRIPTION" "RESOURCE_GROUP:$RESOURCE_GROUP" "LOCATION:$LOCATION" "IDENTITY_NAME:$IDENTITY_NAME" "GIT_BRANCH:$GIT_BRANCH"

# build image
az vm run-command invoke -n $VM_NAME --command-id RunShellScript --scripts @endpoints/online/managed/vnet/setup_vm/scripts/build_image.sh --parameters "SUBSCRIPTION:$SUBSCRIPTION" "RESOURCE_GROUP:$RESOURCE_GROUP" "LOCATION:$LOCATION" "IDENTITY_NAME:$IDENTITY_NAME" "ACR_NAME=$ACR_NAME" "IMAGE_NAME:$IMAGE_NAME" "ENV_DIR_PATH:$ENV_DIR_PATH"

# create endpoint/deployment inside managed vnet
az vm run-command invoke -n $VM_NAME --command-id RunShellScript --scripts @endpoints/online/managed/vnet/setup_vm/scripts/create_moe.sh --parameters "SUBSCRIPTION:$SUBSCRIPTION" "RESOURCE_GROUP:$RESOURCE_GROUP" "LOCATION:$LOCATION" "IDENTITY_NAME:$IDENTITY_NAME" "WORKSPACE:$WORKSPACE" "ENDPOINT_NAME:$ENDPOINT_NAME" "ACR_NAME=$ACR_NAME" "IMAGE_NAME:$IMAGE_NAME" "ENDPOINT_FILE_PATH:$ENDPOINT_FILE_PATH" "DEPLOYMENT_FILE_PATH:$DEPLOYMENT_FILE_PATH" "SAMPLE_REQUEST_PATH:$SAMPLE_REQUEST_PATH"

# test the endpoint by scoring it
export CMD_OUTPUT=$(az vm run-command invoke -n $VM_NAME --command-id RunShellScript --scripts @endpoints/online/managed/vnet/setup_vm/scripts/score_endpoint.sh --parameters "SUBSCRIPTION:$SUBSCRIPTION" "RESOURCE_GROUP:$RESOURCE_GROUP" "LOCATION:$LOCATION" "IDENTITY_NAME:$IDENTITY_NAME" "WORKSPACE:$WORKSPACE" "ENDPOINT_NAME:$ENDPOINT_NAME" "SAMPLE_REQUEST_PATH:$SAMPLE_REQUEST_PATH")

# the scoring output for sample request should be [11055.977245525679, 4503.079536107787]. We are validating if part of the number is available in the output (not comparing all the decimals to accomodate rounding discrepencies)
if [[ $CMD_OUTPUT =~ "11055" ]]; then
   echo "Scoring works!"
else
   echo "Error in scoring"
   # delete the VM before exiting with error
   az vm delete -n $VM_NAME -y --no-wait
   # exit with error
   exit 1
fi


### Cleanup
# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes --no-wait
# </delete_endpoint>
# <delete_vm> 
az vm delete -n $VM_NAME -y --no-wait
# </delete_vm> 
```

To delete all the resources created in this article, use the following command. Replace `<resource-group-name>` with the name of the resource group used in this example:

```azurecli
az group delete --resource-group <resource-group-name>
```

## Troubleshooting

[!INCLUDE [network isolation issues](../../includes/machine-learning-online-endpoint-troubleshooting.md)]

## Next steps

- [Safe rollout for online endpoints](how-to-safely-rollout-online-endpoints.md)
- [How to autoscale managed online endpoints](how-to-autoscale-endpoints.md)
- [View costs for an Azure Machine Learning managed online endpoint](how-to-view-online-endpoints-costs.md)
- [Access Azure resources with a online endpoint and managed identity](how-to-access-resources-from-endpoints-managed-identities.md)
- [Troubleshoot online endpoints deployment](how-to-troubleshoot-online-endpoints.md)