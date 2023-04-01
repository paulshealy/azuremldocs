
# Install and set up the CLI (v2)

[!INCLUDE [cli v2](../../includes/machine-learning-cli-v2.md)]


> [!div class="op_single_selector" title1="Select the version of Azure Machine Learning CLI extension you are using:"]
> * [v1](v1/reference-azure-machine-learning-cli.md)
> * [v2 (current version)](how-to-configure-cli.md)

The `ml` extension to the [Azure CLI](/cli/azure/) is the enhanced interface for Azure Machine Learning. It enables you to train and deploy models from the command line, with features that accelerate scaling data science up and out while tracking the model lifecycle.

## Prerequisites

- To use the CLI, you must have an Azure subscription. If you don't have an Azure subscription, create a free account before you begin. Try the [free or paid version of Azure Machine Learning](https://azure.microsoft.com/free/) today.
- To use the CLI commands in this document from your **local environment**, you need the [Azure CLI](/cli/azure/install-azure-cli).

## Installation

The new Machine Learning extension **requires Azure CLI version `>=2.15.0`**. Ensure this requirement is met:

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

If it isn't, [upgrade your Azure CLI](/cli/azure/update-azure-cli).

Check the Azure CLI extensions you've installed:

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

Remove any existing installation of the `ml` extension and also the CLI v1 `azure-cli-ml` extension:

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

Now, install the `ml` extension:

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

Run the help command to verify your installation and see available subcommands:

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

You can upgrade the extension to the latest version:

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

### Installation on Linux

If you're using Linux, the fastest way to install the necessary CLI version and the Machine Learning extension is:

```bash
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

For more, see [Install the Azure CLI for Linux](/cli/azure/install-azure-cli-linux).

## Set up

Login:

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

If you have access to multiple Azure subscriptions, you can set your active subscription:

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

Optionally, setup common variables in your shell for usage in subsequent commands:

```azurecli
echo "Setting variables..."

# <set_variables>

GROUP="azureml-examples"

LOCATION="eastus"

WORKSPACE="main"

# </set_variables>



# additional variables

SUBSCRIPTION=$(az account show --query id -o tsv)

SECRET_NAME="AZ_CREDS"



echo "Installing Azure CLI extension for Azure Machine Learning..."

# <az_ml_install>

az extension add -n ml -y

# </az_ml_install>



echo "Creating resource group..."

# <az_group_create>

az group create -n $GROUP -l $LOCATION

# </az_group_create>



echo "Creating service principal and setting repository secret..."

# <set_repo_secret>

az ad sp create-for-rbac --name $GROUP --role owner --scopes /subscriptions/$SUBSCRIPTION/resourceGroups/$GROUP --sdk-auth | gh secret set $SECRET_NAME

# </set_repo_secret>



echo "Creating Azure Machine Learning workspace..."

# <az_ml_workspace_create>

az ml workspace create -n $WORKSPACE -g $GROUP -l $LOCATION

# </az_ml_workspace_create>



echo "Configuring Azure CLI defaults..."

# <az_configure_defaults>

az configure --defaults group=$GROUP workspace=$WORKSPACE location=$LOCATION

# </az_configure_defaults>



echo "Setting up workspace..."

bash -x setup-workspace.sh

echo "Setting up managed vnet workspace"
# Managed online endpoint vnet setup: Create via bicep: vnet, workspace, storage, acr, kv, nsg, PEs + UAI
# <managed_vnet_workspace_suffix>
# SUFFIX will be used as resource name suffix in created workspace and related resources
export SUFFIX="<UNIQUE_SUFFIX>"
# </managed_vnet_workspace_suffix>
export SUFFIX="mevnet"
# <managed_vnet_workspace_create>
az deployment group create --template-file endpoints/online/managed/vnet/setup_ws/main.bicep --parameters suffix=$SUFFIX
# Note: if you get an error that appinsights is not available in your current location, use optional parameter to the above script: appinsightsLocation=<location> (e.g. westus2)
# </managed_vnet_workspace_create>

# create the user assigned identity used in the vnet egress testing
az deployment group create --template-file endpoints/online/managed/vnet/setup_ws/uai.bicep --parameters suffix=$SUFFIX

echo "Setting up internal workspaces..."

bash -x create-workspace-internal.sh



echo "Setting up extra workspaces..."

bash -x create-workspace-extras.sh


```

> [!WARNING]
> This uses Bash syntax for setting variables -- adjust as needed for your shell. You can also replace the values in commands below inline rather than using variables.

If it doesn't already exist, you can create the Azure resource group:

```azurecli
echo "Setting variables..."

# <set_variables>

GROUP="azureml-examples"

LOCATION="eastus"

WORKSPACE="main"

# </set_variables>



# additional variables

SUBSCRIPTION=$(az account show --query id -o tsv)

SECRET_NAME="AZ_CREDS"



echo "Installing Azure CLI extension for Azure Machine Learning..."

# <az_ml_install>

az extension add -n ml -y

# </az_ml_install>



echo "Creating resource group..."

# <az_group_create>

az group create -n $GROUP -l $LOCATION

# </az_group_create>



echo "Creating service principal and setting repository secret..."

# <set_repo_secret>

az ad sp create-for-rbac --name $GROUP --role owner --scopes /subscriptions/$SUBSCRIPTION/resourceGroups/$GROUP --sdk-auth | gh secret set $SECRET_NAME

# </set_repo_secret>



echo "Creating Azure Machine Learning workspace..."

# <az_ml_workspace_create>

az ml workspace create -n $WORKSPACE -g $GROUP -l $LOCATION

# </az_ml_workspace_create>



echo "Configuring Azure CLI defaults..."

# <az_configure_defaults>

az configure --defaults group=$GROUP workspace=$WORKSPACE location=$LOCATION

# </az_configure_defaults>



echo "Setting up workspace..."

bash -x setup-workspace.sh

echo "Setting up managed vnet workspace"
# Managed online endpoint vnet setup: Create via bicep: vnet, workspace, storage, acr, kv, nsg, PEs + UAI
# <managed_vnet_workspace_suffix>
# SUFFIX will be used as resource name suffix in created workspace and related resources
export SUFFIX="<UNIQUE_SUFFIX>"
# </managed_vnet_workspace_suffix>
export SUFFIX="mevnet"
# <managed_vnet_workspace_create>
az deployment group create --template-file endpoints/online/managed/vnet/setup_ws/main.bicep --parameters suffix=$SUFFIX
# Note: if you get an error that appinsights is not available in your current location, use optional parameter to the above script: appinsightsLocation=<location> (e.g. westus2)
# </managed_vnet_workspace_create>

# create the user assigned identity used in the vnet egress testing
az deployment group create --template-file endpoints/online/managed/vnet/setup_ws/uai.bicep --parameters suffix=$SUFFIX

echo "Setting up internal workspaces..."

bash -x create-workspace-internal.sh



echo "Setting up extra workspaces..."

bash -x create-workspace-extras.sh


```

And create a machine learning workspace:

```azurecli
echo "Setting variables..."

# <set_variables>

GROUP="azureml-examples"

LOCATION="eastus"

WORKSPACE="main"

# </set_variables>



# additional variables

SUBSCRIPTION=$(az account show --query id -o tsv)

SECRET_NAME="AZ_CREDS"



echo "Installing Azure CLI extension for Azure Machine Learning..."

# <az_ml_install>

az extension add -n ml -y

# </az_ml_install>



echo "Creating resource group..."

# <az_group_create>

az group create -n $GROUP -l $LOCATION

# </az_group_create>



echo "Creating service principal and setting repository secret..."

# <set_repo_secret>

az ad sp create-for-rbac --name $GROUP --role owner --scopes /subscriptions/$SUBSCRIPTION/resourceGroups/$GROUP --sdk-auth | gh secret set $SECRET_NAME

# </set_repo_secret>



echo "Creating Azure Machine Learning workspace..."

# <az_ml_workspace_create>

az ml workspace create -n $WORKSPACE -g $GROUP -l $LOCATION

# </az_ml_workspace_create>



echo "Configuring Azure CLI defaults..."

# <az_configure_defaults>

az configure --defaults group=$GROUP workspace=$WORKSPACE location=$LOCATION

# </az_configure_defaults>



echo "Setting up workspace..."

bash -x setup-workspace.sh

echo "Setting up managed vnet workspace"
# Managed online endpoint vnet setup: Create via bicep: vnet, workspace, storage, acr, kv, nsg, PEs + UAI
# <managed_vnet_workspace_suffix>
# SUFFIX will be used as resource name suffix in created workspace and related resources
export SUFFIX="<UNIQUE_SUFFIX>"
# </managed_vnet_workspace_suffix>
export SUFFIX="mevnet"
# <managed_vnet_workspace_create>
az deployment group create --template-file endpoints/online/managed/vnet/setup_ws/main.bicep --parameters suffix=$SUFFIX
# Note: if you get an error that appinsights is not available in your current location, use optional parameter to the above script: appinsightsLocation=<location> (e.g. westus2)
# </managed_vnet_workspace_create>

# create the user assigned identity used in the vnet egress testing
az deployment group create --template-file endpoints/online/managed/vnet/setup_ws/uai.bicep --parameters suffix=$SUFFIX

echo "Setting up internal workspaces..."

bash -x create-workspace-internal.sh



echo "Setting up extra workspaces..."

bash -x create-workspace-extras.sh


```

Machine learning subcommands require the `--workspace/-w` and `--resource-group/-g` parameters. To avoid typing these repeatedly, configure defaults:

```azurecli
#!/bin/bash
# rc install - uncomment and adjust below to run all tests on a CLI release candidate
# az extension remove -n ml

# <az_ml_install>
az extension add -n ml -y
# </az_ml_install>

# Use a daily build
# az extension add --source https://azuremlsdktestpypi.blob.core.windows.net/wheels/sdk-cli-v2-public/ml-2.9.0-py3-none-any.whl --yes
# remove ml extension if it is installed
# if az extension show -n ml &>/dev/null; then
#     echo -n 'Removing ml extension...'
#     if ! az extension remove -n ml -o none --only-show-errors &>/dev/null; then
#         echo 'Error failed to remove ml extension' >&2
#     fi
#     echo -n 'Re-installing ml...'
# fi

# if ! az extension add --yes --source "https://azuremlsdktestpypi.blob.core.windows.net/wheels/sdk-cli-v2-public/ml-2.10.0-py3-none-any.whl" -o none --only-show-errors &>/dev/null; then
#     echo 'Error failed to install ml azure-cli extension' >&2
#     exit 1
# fi

# az version

## For backward compatibility - running on old subscription
# <set_variables>
GROUP="azureml-examples"
LOCATION="eastus"
WORKSPACE="main"
# </set_variables>

# If RESOURCE_GROUP_NAME is empty, the az configure is pending.
RESOURCE_GROUP_NAME=${RESOURCE_GROUP_NAME:-}
if [[ -z "$RESOURCE_GROUP_NAME" ]]
then
    echo "No resource group name [RESOURCE_GROUP_NAME] specified, defaulting to ${GROUP}."
    # Installing extension temporarily assuming the run is on old subscription
    # without bootstrap script.

    # <az_configure_defaults>
    az configure --defaults group=$GROUP workspace=$WORKSPACE location=$LOCATION
    # </az_configure_defaults>
    echo "Default resource group set to $GROUP"
else
    echo "Workflows are using the new subscription."
fi
```

> [!TIP]
> Most code examples assume you have set a default workspace and resource group. You can override these on the command line.

You can show your current defaults using `--list-defaults/-l`:

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

> [!TIP]
> Combining with `--output/-o` allows for more readable output formats.

## Secure communications

The `ml` CLI extension (sometimes called 'CLI v2') for Azure Machine Learning sends operational data (YAML parameters and metadata) over the public internet. All the `ml` CLI extension commands communicate with the Azure Resource Manager. This communication is secured using HTTPS/TLS 1.2.

Data in a data store that is secured in a virtual network is _not_ sent over the public internet. For example, if your training data is located in the default storage account for the workspace, and the storage account is in a virtual network.

> [!NOTE]
> With the previous extension (`azure-cli-ml`, sometimes called 'CLI v1'), only some of the commands communicate with the Azure Resource Manager. Specifically, commands that create, update, delete, list, or show Azure resources. Operations such as submitting a training job communicate directly with the Azure Machine Learning workspace. If your workspace is [secured with a private endpoint](how-to-configure-private-link.md), that is enough to secure commands provided by the `azure-cli-ml` extension.

# [Public workspace](#tab/public)

If your Azure Machine Learning workspace is public (that is, not behind a virtual network), then there is no additional configuration required. Communications are secured using HTTPS/TLS 1.2

# [Private workspace](#tab/private)

If your Azure Machine Learning workspace uses a private endpoint and virtual network, choose one of the following configurations to use:

* If you are __OK__ with the CLI v2 communication over the public internet, use the following `--public-network-access` parameter for the `az ml workspace update` command to enable public network access. For example, the following command updates a workspace for public network access:

    ```azurecli
    az ml workspace update --name myworkspace --public-network-access enabled
    ```

* If you are __not OK__ with the CLI v2 communication over the public internet, you can use an Azure Private Link to increase security of the communication. Use the following links to secure communications with Azure Resource Manager by using Azure Private Link.

    1. [Secure your Azure Machine Learning workspace inside a virtual network using a private endpoint](how-to-configure-private-link.md).
    2. [Create a Private Link for managing Azure resources](../azure-resource-manager/management/create-private-link-access-portal.md). 
    3. [Create a private endpoint](../azure-resource-manager/management/create-private-link-access-portal.md#create-private-endpoint) for the Private Link created in the previous step.

    > [!IMPORTANT]
    > To configure the private link for Azure Resource Manager, you must be the _subscription owner_ for the Azure subscription, and an _owner_ or _contributor_ of the root management group. For more information, see [Create a private link for managing Azure resources](../azure-resource-manager/management/create-private-link-access-portal.md).


## Next steps

- [Train models using CLI (v2)](how-to-train-model.md)
- [Set up the Visual Studio Code Azure Machine Learning extension](how-to-setup-vs-code.md)
- [Train an image classification TensorFlow model using the Azure Machine Learning Visual Studio Code extension](tutorial-train-deploy-image-classification-model-vscode.md)
- [Explore Azure Machine Learning with examples](samples-notebooks.md)
