
# Access Azure resources from an online endpoint with a managed identity 

[!INCLUDE [dev v2](../../includes/machine-learning-dev-v2.md)]

Learn how to access Azure resources from your scoring script with an online endpoint and either a system-assigned managed identity or a user-assigned managed identity. 

Managed endpoints allow Azure Machine Learning to manage the burden of provisioning your compute resource and deploying your machine learning model. Typically your model needs to access Azure resources such as the Azure Container Registry or your blob storage for inferencing; with a managed identity you can access these resources without needing to manage credentials in your code. [Learn more about managed identities](../active-directory/managed-identities-azure-resources/overview.md).

This guide assumes you don't have a managed identity, a storage account or an online endpoint. If you already have these components, skip to the [give access permission to the managed identity](#give-access-permission-to-the-managed-identity) section. 

## Prerequisites

# [System-assigned (CLI)](#tab/system-identity-cli)

* To use Azure Machine Learning, you must have an Azure subscription. If you don't have an Azure subscription, create a free account before you begin. Try the [free or paid version of Azure Machine Learning](https://azure.microsoft.com/free/) today.

* Install and configure the Azure CLI and ML (v2) extension. For more information, see [Install, set up, and use the 2.0 CLI](how-to-configure-cli.md).

* An Azure Resource group, in which you (or the service principal you use) need to have `User Access Administrator` and  `Contributor` access. You'll have such a resource group if you configured your ML extension per the above article.

* An Azure Machine Learning workspace. You'll have a workspace if you configured your ML extension per the above article.

* A trained machine learning model ready for scoring and deployment. If you are following along with the sample, a model is provided.

* If you haven't already set the defaults for the Azure CLI, save your default settings. To avoid passing in the values for your subscription, workspace, and resource group multiple times, run this code:

   ```azurecli
   az account set --subscription <subscription ID>
   az configure --defaults gitworkspace=<Azure Machine Learning workspace name> group=<resource group>
   ```

* To follow along with the sample, clone the samples repository

    ```azurecli
    git clone https://github.com/Azure/azureml-examples --depth 1
    cd azureml-examples/cli
    ```
    
# [User-assigned (CLI)](#tab/user-identity-cli)

* To use Azure Machine Learning, you must have an Azure subscription. If you don't have an Azure subscription, create a free account before you begin. Try the [free or paid version of Azure Machine Learning](https://azure.microsoft.com/free/) today.

* Install and configure the Azure CLI and ML (v2) extension. For more information, see [Install, set up, and use the 2.0 CLI](how-to-configure-cli.md).

* An Azure Resource group, in which you (or the service principal you use) need to have `User Access Administrator` and  `Contributor` access. You'll have such a resource group if you configured your ML extension per the above article.

* An Azure Machine Learning workspace. You'll have a workspace if you configured your ML extension per the above article.

* A trained machine learning model ready for scoring and deployment. If you are following along with the sample, a model is provided.

* If you haven't already set the defaults for the Azure CLI, save your default settings. To avoid passing in the values for your subscription, workspace, and resource group multiple times, run this code:

   ```azurecli
   az account set --subscription <subscription ID>
   az configure --defaults gitworkspace=<Azure Machine Learning workspace name> group=<resource group>
   ```

* To follow along with the sample, clone the samples repository

    ```azurecli
    git clone https://github.com/Azure/azureml-examples --depth 1
    cd azureml-examples/cli
    ```

# [System-assigned (Python)](#tab/system-identity-python)

* To use Azure Machine Learning, you must have an Azure subscription. If you don't have an Azure subscription, create a free account before you begin. Try the [free or paid version of Azure Machine Learning](https://azure.microsoft.com/free/) today.

* Install and configure the Azure ML Python SDK (v2). For more information, see [Install and set up SDK (v2)](https://aka.ms/sdk-v2-install).

* An Azure Resource group, in which you (or the service principal you use) need to have `User Access Administrator` and  `Contributor` access. You'll have such a resource group if you configured your ML extension per the above article.

* An Azure Machine Learning workspace. You'll have a workspace if you configured your ML extension per the above article.

* A trained machine learning model ready for scoring and deployment. If you are following along with the sample, a model is provided.

* Clone the samples repository. 

    ```azurecli
    git clone https://github.com/Azure/azureml-examples --depth 1
    cd azureml-examples/sdk/endpoints/online/managed/managed-identities
    ```
* To follow along with this notebook, access the companion [example notebook](https://github.com/Azure/azureml-examples/blob/main/sdk/python/endpoints/online/managed/managed-identities/online-endpoints-managed-identity-uai.ipynb) within in the  `sdk/endpoints/online/managed/managed-identities` directory. 

* Additional Python packages are required for this example: 

    * Microsoft Azure Storage Management Client 

    * Microsoft Azure Authorization Management Client

    Install them with the following code: 
    
```python
%pip install --pre azure-mgmt-storage
%pip install --pre azure-mgmt-authorization
```

# [User-assigned (Python)](#tab/user-identity-python)

* To use Azure Machine Learning, you must have an Azure subscription. If you don't have an Azure subscription, create a free account before you begin. Try the [free or paid version of Azure Machine Learning](https://azure.microsoft.com/free/) today.

* Role creation permissions for your subscription or the Azure resources accessed by the User-assigned identity. 

* Install and configure the Azure ML Python SDK (v2). For more information, see [Install and set up SDK (v2)](https://aka.ms/sdk-v2-install).

* An Azure Resource group, in which you (or the service principal you use) need to have `User Access Administrator` and  `Contributor` access. You'll have such a resource group if you configured your ML extension per the above article.

* An Azure Machine Learning workspace. You'll have a workspace if you configured your ML extension per the above article.

* A trained machine learning model ready for scoring and deployment. If you are following along with the sample, a model is provided.

* Clone the samples repository. 

    ```azurecli
    git clone https://github.com/Azure/azureml-examples --depth 1
    cd azureml-examples/sdk/endpoints/online/managed/managed-identities
    ```
* To follow along with this notebook, access the companion [example notebook](https://github.com/Azure/azureml-examples/blob/main/sdk/python/endpoints/online/managed/managed-identities/online-endpoints-managed-identity-uai.ipynb) within in the  `sdk/endpoints/online/managed/managed-identities` directory. 

* Additional Python packages are required for this example: 

    * Microsoft Azure Msi Management Client 

    * Microsoft Azure Storage Client

    * Microsoft Azure Authorization Management Client

    Install them with the following code: 
    
```python
%pip install --pre azure-mgmt-msi
%pip install --pre azure-mgmt-storage
%pip install --pre azure-mgmt-authorization
```
    

## Limitations

* The identity for an endpoint is immutable. During endpoint creation, you can associate it with a system-assigned identity (default) or a user-assigned identity. You can't change the identity after the endpoint has been created.

## Configure variables for deployment

Configure the variable names for the workspace, workspace location, and the endpoint you want to create for use with your deployment.

# [System-assigned (CLI)](#tab/system-identity-cli)

The following code exports these values as environment variables in your endpoint:

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-sai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-sai-create-endpoint.yml
# </create_endpoint>
endpoint_status=`az ml online-endpoint show --name $ENDPOINT_NAME --query "provisioning_state" -o tsv`

echo $endpoint_status

if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else 
  echo "Endpoint creation failed"
  exit 1
fi

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# role assignment fails without sleep statement
sleep 60

# <get_system_identity>
system_identity=`az ml online-endpoint show --name $ENDPOINT_NAME --query "identity.principal_id" -o tsv`
# </get_system_identity>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $system_identity --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue --file endpoints/online/managed/managed-identities/2-sai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue
# </check_deploy_Status>

deploy_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $deploy_status
if [[ $deploy_status == "Succeeded" ]]
then
  echo "Deployment completed successfully"
else
  echo "Deployment failed"
  exit 1
fi

# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

```

Next, specify what you want to name your blob storage account, blob container, and file. These variable names are defined here, and are referred to in `az storage account create` and `az storage container create` commands in the next section.

The following code exports those values as environment variables:

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-sai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-sai-create-endpoint.yml
# </create_endpoint>
endpoint_status=`az ml online-endpoint show --name $ENDPOINT_NAME --query "provisioning_state" -o tsv`

echo $endpoint_status

if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else 
  echo "Endpoint creation failed"
  exit 1
fi

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# role assignment fails without sleep statement
sleep 60

# <get_system_identity>
system_identity=`az ml online-endpoint show --name $ENDPOINT_NAME --query "identity.principal_id" -o tsv`
# </get_system_identity>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $system_identity --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue --file endpoints/online/managed/managed-identities/2-sai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue
# </check_deploy_Status>

deploy_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $deploy_status
if [[ $deploy_status == "Succeeded" ]]
then
  echo "Deployment completed successfully"
else
  echo "Deployment failed"
  exit 1
fi

# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

```

After these variables are exported, create a text file locally. When the endpoint is deployed, the scoring script will access this text file using the system-assigned managed identity that's generated upon endpoint creation.

# [User-assigned (CLI)](#tab/user-identity-cli)

Decide on the name of your endpoint, workspace, workspace location and export that value as an environment variable:

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-uai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <set_user_identity_name>
export UAI_NAME="<USER_ASSIGNED_IDENTITY_NAME>"
# </set_user_identity_name>

export UAI_NAME=oep-user-identity-$TEST_ID

# <create_user_identity>
az identity create --name $UAI_NAME
# </create_user_identity>

# role assignment fails without sleep statement
sleep 60

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <get_user_identity_client_id>
uai_clientid=`az identity list --query "[?name=='$UAI_NAME'].clientId" -o tsv`
uai_principalid=`az identity list --query "[?name=='$UAI_NAME'].principalId" -o tsv`
# </get_user_identity_client_id>

# <get_user_identity_id>
uai_id=`az identity list --query "[?name=='$UAI_NAME'].id" -o tsv`
# </get_user_identity_id>

# <get_container_registry_id>
container_registry=`az ml workspace show --name $WORKSPACE --query container_registry -o tsv`
# </get_container_registry_id>

# <get_workspace_storage_id>
storage_account=`az ml workspace show --name $WORKSPACE --query storage_account -o tsv`
# </get_workspace_storage_id>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <give_permission_to_container_registry>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "AcrPull" --scope $container_registry
# </give_permission_to_container_registry>

# <give_permission_to_workspace_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "Storage Blob Data Reader" --scope $storage_account
# </give_permission_to_workspace_storage_account>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-uai-create-endpoint.yml --set identity.user_assigned_identities[0].resource_id=$uai_id
# </create_endpoint>

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue -f endpoints/online/managed/managed-identities/2-uai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME environment_variables.UAI_CLIENT_ID=$uai_clientid
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME -n blue
# </check_deploy_Status>

endpoint_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $endpoint_status
if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else
  echo "Endpoint creation failed"
  exit 1
fi


# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

# <delete_user_identity>
az identity delete --name $UAI_NAME
# </delete_user_identity>

```

Next, specify what you want to name your blob storage account, blob container, and file. These variable names are defined here, and are referred to in `az storage account create` and `az storage container create` commands in the next section.

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-uai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <set_user_identity_name>
export UAI_NAME="<USER_ASSIGNED_IDENTITY_NAME>"
# </set_user_identity_name>

export UAI_NAME=oep-user-identity-$TEST_ID

# <create_user_identity>
az identity create --name $UAI_NAME
# </create_user_identity>

# role assignment fails without sleep statement
sleep 60

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <get_user_identity_client_id>
uai_clientid=`az identity list --query "[?name=='$UAI_NAME'].clientId" -o tsv`
uai_principalid=`az identity list --query "[?name=='$UAI_NAME'].principalId" -o tsv`
# </get_user_identity_client_id>

# <get_user_identity_id>
uai_id=`az identity list --query "[?name=='$UAI_NAME'].id" -o tsv`
# </get_user_identity_id>

# <get_container_registry_id>
container_registry=`az ml workspace show --name $WORKSPACE --query container_registry -o tsv`
# </get_container_registry_id>

# <get_workspace_storage_id>
storage_account=`az ml workspace show --name $WORKSPACE --query storage_account -o tsv`
# </get_workspace_storage_id>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <give_permission_to_container_registry>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "AcrPull" --scope $container_registry
# </give_permission_to_container_registry>

# <give_permission_to_workspace_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "Storage Blob Data Reader" --scope $storage_account
# </give_permission_to_workspace_storage_account>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-uai-create-endpoint.yml --set identity.user_assigned_identities[0].resource_id=$uai_id
# </create_endpoint>

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue -f endpoints/online/managed/managed-identities/2-uai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME environment_variables.UAI_CLIENT_ID=$uai_clientid
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME -n blue
# </check_deploy_Status>

endpoint_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $endpoint_status
if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else
  echo "Endpoint creation failed"
  exit 1
fi


# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

# <delete_user_identity>
az identity delete --name $UAI_NAME
# </delete_user_identity>

```

After these variables are exported, create a text file locally. When the endpoint is deployed, the scoring script will access this text file using the user-assigned managed identity used in the endpoint. 

Decide on the name of your user identity name, and export that value as an environment variable:

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-uai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <set_user_identity_name>
export UAI_NAME="<USER_ASSIGNED_IDENTITY_NAME>"
# </set_user_identity_name>

export UAI_NAME=oep-user-identity-$TEST_ID

# <create_user_identity>
az identity create --name $UAI_NAME
# </create_user_identity>

# role assignment fails without sleep statement
sleep 60

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <get_user_identity_client_id>
uai_clientid=`az identity list --query "[?name=='$UAI_NAME'].clientId" -o tsv`
uai_principalid=`az identity list --query "[?name=='$UAI_NAME'].principalId" -o tsv`
# </get_user_identity_client_id>

# <get_user_identity_id>
uai_id=`az identity list --query "[?name=='$UAI_NAME'].id" -o tsv`
# </get_user_identity_id>

# <get_container_registry_id>
container_registry=`az ml workspace show --name $WORKSPACE --query container_registry -o tsv`
# </get_container_registry_id>

# <get_workspace_storage_id>
storage_account=`az ml workspace show --name $WORKSPACE --query storage_account -o tsv`
# </get_workspace_storage_id>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <give_permission_to_container_registry>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "AcrPull" --scope $container_registry
# </give_permission_to_container_registry>

# <give_permission_to_workspace_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "Storage Blob Data Reader" --scope $storage_account
# </give_permission_to_workspace_storage_account>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-uai-create-endpoint.yml --set identity.user_assigned_identities[0].resource_id=$uai_id
# </create_endpoint>

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue -f endpoints/online/managed/managed-identities/2-uai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME environment_variables.UAI_CLIENT_ID=$uai_clientid
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME -n blue
# </check_deploy_Status>

endpoint_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $endpoint_status
if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else
  echo "Endpoint creation failed"
  exit 1
fi


# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

# <delete_user_identity>
az identity delete --name $UAI_NAME
# </delete_user_identity>

```

# [System-assigned (Python)](#tab/system-identity-python)

Assign values for the workspace and deployment-related variables:

```python
subscription_id = "<SUBSCRIPTION_ID>"
resource_group = "<RESOURCE_GROUP>"
workspace_name = "<AML_WORKSPACE_NAME>"
endpoint_name = "<ENDPOINT_NAME>"
```

Next, specify what you want to name your blob storage account, blob container, and file. These variable names are defined here, and are referred to in the storage account and container creation code by the `StorageManagementClient` and `ContainerClient`. 

```python
storage_account_name = "<STORAGE_ACCOUNT_NAME>"
storage_container_name = "<CONTAINER_TO_ACCESS>"
file_name = "<FILE_TO_ACCESS>"
```

After these variables are assigned, create a text file locally. When the endpoint is deployed, the scoring script will access this text file using the system-assigned managed identity that's generated upon endpoint creation.

Now, get a handle to the workspace and retrieve its location:

```python
from azure.ai.ml import MLClient
from azure.identity import AzureCliCredential
from azure.ai.ml.entities import (
    ManagedOnlineDeployment,
    ManagedOnlineEndpoint,
    Model,
    CodeConfiguration,
    Environment,
)

credential = AzureCliCredential()
ml_client = MLClient(credential, subscription_id, resource_group, workspace_name)

workspace_location = ml_client.workspaces.get(workspace_name).location
```

We will use this value to create a storage account. 


# [User-assigned (Python)](#tab/user-identity-python)


Assign values for the workspace and deployment-related variables:

```python
subscription_id = "<SUBSCRIPTION_ID>"
resource_group = "<RESOURCE_GROUP>"
workspace_name = "<AML_WORKSPACE_NAME>"
endpoint_name = "<ENDPOINT_NAME>"
```

Next, specify what you want to name your blob storage account, blob container, and file. These variable names are defined here, and are referred to in the storage account and container creation code by the `StorageManagementClient` and `ContainerClient`. 

```python
storage_account_name = "<STORAGE_ACCOUNT_NAME>"
storage_container_name = "<CONTAINER_TO_ACCESS>"
file_name = "<FILE_TO_ACCESS>"
```

After these variables are assigned, create a text file locally. When the endpoint is deployed, the scoring script will access this text file using the user-assigned managed identity that's generated upon endpoint creation.

Decide on the name of your user identity name:
```python
uai_name = "<USER_ASSIGNED_IDENTITY_NAME>"
```

Now, get a handle to the workspace and retrieve its location:

```python
from azure.ai.ml import MLClient
from azure.identity import AzureCliCredential
from azure.ai.ml.entities import (
    ManagedOnlineDeployment,
    ManagedOnlineEndpoint,
    Model,
    CodeConfiguration,
    Environment,
)

credential = AzureCliCredential()
ml_client = MLClient(credential, subscription_id, resource_group, workspace_name)

workspace_location = ml_client.workspaces.get(workspace_name).location
```

We will use this value to create a storage account. 


## Define the deployment configuration


# [System-assigned (CLI)](#tab/system-identity-cli)

To deploy an online endpoint with the CLI, you need to define the configuration in a YAML file. For more information on the YAML schema, see [online endpoint YAML reference](reference-yaml-endpoint-online.md) document.

The YAML files in the following examples are used to create online endpoints. 

The following YAML example is located at `endpoints/online/managed/managed-identities/1-sai-create-endpoint`. The file, 

* Defines the name by which you want to refer to the endpoint, `my-sai-endpoint`.
* Specifies the type of authorization to use to access the endpoint, `auth-mode: key`.

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/managedOnlineEndpoint.schema.json
name: my-sai-endpoint
auth_mode: key

```

This YAML example, `2-sai-deployment.yml`,

* Specifies that the type of endpoint you want to create is an `online` endpoint.
* Indicates that the endpoint has an associated deployment called `blue`.
* Configures the details of the deployment such as, which model to deploy and which environment and scoring script to use.

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/managedOnlineDeployment.schema.json
name: blue
model:
  path: ../../model-1/model/
code_configuration:
  code: ../../model-1/onlinescoring/
  scoring_script: score_managedidentity.py
environment:
  conda_file: ../../model-1/environment/conda-managedidentity.yml
  image: mcr.microsoft.com/azureml/openmpi4.1.0-ubuntu20.04:latest
instance_type: Standard_DS3_v2
instance_count: 1
environment_variables:
  STORAGE_ACCOUNT_NAME: "storage_place_holder"
  STORAGE_CONTAINER_NAME: "container_place_holder"
  FILE_NAME: "file_place_holder"

```

# [User-assigned (CLI)](#tab/user-identity-cli)

To deploy an online endpoint with the CLI, you need to define the configuration in a YAML file. For more information on the YAML schema, see [online endpoint YAML reference](reference-yaml-endpoint-online.md) document.

The YAML files in the following examples are used to create online endpoints. 

The following YAML example is located at `endpoints/online/managed/managed-identities/1-uai-create-endpoint`. The file, 

* Defines the name by which you want to refer to the endpoint, `my-uai-endpoint`.
* Specifies the type of authorization to use to access the endpoint, `auth-mode: key`.
* Indicates the identity type to use, `type: user_assigned`

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/managedOnlineEndpoint.schema.json
name: my-uai-endpoint
auth_mode: key
identity:
  type: user_assigned
  user_assigned_identities:
    - resource_id: user_identity_ARM_id_place_holder

```

This YAML example, `2-sai-deployment.yml`,

* Specifies that the type of endpoint you want to create is an `online` endpoint.
* Indicates that the endpoint has an associated deployment called `blue`.
* Configures the details of the deployment such as, which model to deploy and which environment and scoring script to use.

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/managedOnlineDeployment.schema.json
model:
  path: ../../model-1/model/
code_configuration:
  code: ../../model-1/onlinescoring/
  scoring_script: score_managedidentity.py
environment: 
  conda_file: ../../model-1/environment/conda-managedidentity.yml
  image: mcr.microsoft.com/azureml/openmpi4.1.0-ubuntu20.04:latest
instance_type: Standard_DS3_v2
instance_count: 1
environment_variables:
  STORAGE_ACCOUNT_NAME: "storage_place_holder"
  STORAGE_CONTAINER_NAME: "container_place_holder"
  FILE_NAME: "file_place_holder"
  UAI_CLIENT_ID: "uai_client_id_place_holder"
```

# [System-assigned (Python)](#tab/system-identity-python)

To deploy an online endpoint with the Python SDK (v2), objects may be used to define the configuration as below. Alternatively, YAML files may be loaded using the `.load` method. 

The following Python endpoint object: 

* Assigns the name by which you want to refer to the endpoint to the variable `endpoint_name. 
* Specifies the type of authorization to use to access the endpoint `auth-mode="key"`.

```python
endpoint = ManagedOnlineEndpoint(name=endpoint_name, auth_mode="key")
```

This deployment object: 

* Specifies that the type of deployment you want to create is a `ManagedOnlineDeployment` via the class. 
* Indicates that the endpoint has an associated deployment called `blue`.
* Configures the details of the deployment such as the `name` and `instance_count` 
* Defines additional objects inline and associates them with the deployment for `Model`,`CodeConfiguration`, and `Environment`. 
* Includes environment variables needed for the system-assigned managed identity to access storage.  


```python
deployment = ManagedOnlineDeployment(
    name="blue",
    endpoint_name=endpoint_name,
    model=Model(path="../../model-1/model/"),
    code_configuration=CodeConfiguration(
        code="../../model-1/onlinescoring/", scoring_script="score_managedidentity.py"
    ),
    environment=Environment(
        conda_file="../../model-1/environment/conda-managedidentity.yml",
        image="mcr.microsoft.com/azureml/openmpi4.1.0-ubuntu20.04:latest",
    ),
    instance_type="Standard_DS3_v2",
    instance_count=1,
    environment_variables={
        "STORAGE_ACCOUNT_NAME": storage_account_name,
        "STORAGE_CONTAINER_NAME": storage_container_name,
        "FILE_NAME": file_name,
    },
)
```

# [User-assigned (Python)](#tab/user-identity-python)

To deploy an online endpoint with the Python SDK (v2), objects may be used to define the configuration as below. Alternatively, YAML files may be loaded using the `.load` method. 

For a user-assigned identity, we will define the endpoint configuration below once the User-Assigned Managed Identity has been created. 

This deployment object: 

* Specifies that the type of deployment you want to create is a `ManagedOnlineDeployment` via the class. 
* Indicates that the endpoint has an associated deployment called `blue`.
* Configures the details of the deployment such as the `name` and `instance_count` 
* Defines additional objects inline and associates them with the deployment for `Model`,`CodeConfiguration`, and `Environment`. 
* Includes environment variables needed for the user-assigned managed identity to access storage. 
* Adds a placeholder environment variable for `UAI_CLIENT_ID`, which will be added after creating one and before actually deploying this configuration. 


```python
deployment = ManagedOnlineDeployment(
    name="blue",
    endpoint_name=endpoint_name,
    model=Model(path="../../model-1/model/"),
    code_configuration=CodeConfiguration(
        code="../../model-1/onlinescoring/", scoring_script="score_managedidentity.py"
    ),
    environment=Environment(
        conda_file="../../model-1/environment/conda-managedidentity.yml",
        image="mcr.microsoft.com/azureml/openmpi4.1.0-ubuntu20.04:latest",
    ),
    instance_type="Standard_DS3_v2",
    instance_count=1,
    environment_variables={
        "STORAGE_ACCOUNT_NAME": storage_account_name,
        "STORAGE_CONTAINER_NAME": storage_container_name,
        "FILE_NAME": file_name,
        # We will update this after creating an identity
        "UAI_CLIENT_ID": "uai_client_id_place_holder",
    },
)
```



## Create the managed identity 
To access Azure resources, create a system-assigned or user-assigned managed identity for your online endpoint. 

# [System-assigned (CLI)](#tab/system-identity-cli)

When you [create an online endpoint](#create-an-online-endpoint), a system-assigned managed identity is automatically generated for you, so no need to create a separate one. 

# [User-assigned (CLI)](#tab/user-identity-cli)

To create a user-assigned managed identity, use the following:

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-uai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <set_user_identity_name>
export UAI_NAME="<USER_ASSIGNED_IDENTITY_NAME>"
# </set_user_identity_name>

export UAI_NAME=oep-user-identity-$TEST_ID

# <create_user_identity>
az identity create --name $UAI_NAME
# </create_user_identity>

# role assignment fails without sleep statement
sleep 60

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <get_user_identity_client_id>
uai_clientid=`az identity list --query "[?name=='$UAI_NAME'].clientId" -o tsv`
uai_principalid=`az identity list --query "[?name=='$UAI_NAME'].principalId" -o tsv`
# </get_user_identity_client_id>

# <get_user_identity_id>
uai_id=`az identity list --query "[?name=='$UAI_NAME'].id" -o tsv`
# </get_user_identity_id>

# <get_container_registry_id>
container_registry=`az ml workspace show --name $WORKSPACE --query container_registry -o tsv`
# </get_container_registry_id>

# <get_workspace_storage_id>
storage_account=`az ml workspace show --name $WORKSPACE --query storage_account -o tsv`
# </get_workspace_storage_id>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <give_permission_to_container_registry>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "AcrPull" --scope $container_registry
# </give_permission_to_container_registry>

# <give_permission_to_workspace_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "Storage Blob Data Reader" --scope $storage_account
# </give_permission_to_workspace_storage_account>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-uai-create-endpoint.yml --set identity.user_assigned_identities[0].resource_id=$uai_id
# </create_endpoint>

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue -f endpoints/online/managed/managed-identities/2-uai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME environment_variables.UAI_CLIENT_ID=$uai_clientid
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME -n blue
# </check_deploy_Status>

endpoint_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $endpoint_status
if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else
  echo "Endpoint creation failed"
  exit 1
fi


# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

# <delete_user_identity>
az identity delete --name $UAI_NAME
# </delete_user_identity>

```

# [System-assigned (Python)](#tab/system-identity-python)

When you [create an online endpoint](#create-an-online-endpoint), a system-assigned managed identity is automatically generated for you, so no need to create a separate one. 

# [User-assigned (Python)](#tab/user-identity-python)

To create a user-assigned managed identity, first get a handle to the `ManagedServiceIdentityClient`: 

```python
from azure.mgmt.msi import ManagedServiceIdentityClient
from azure.mgmt.msi.models import Identity

credential = AzureCliCredential()
msi_client = ManagedServiceIdentityClient(
    subscription_id=subscription_id,
    credential=credential,
)
```

Then, create the identity:

```python
msi_client.user_assigned_identities.create_or_update(
    resource_group_name=resource_group,
    resource_name=uai_name,
    parameters=Identity(location=workspace_location),
)
```

Now, retrieve the identity object, which contains details we will use below: 

```python
uai_identity = msi_client.user_assigned_identities.get(
    resource_group_name=resource_group,
    resource_name=uai_name,
)
uai_identity.as_dict()
```


## Create storage account and container

For this example, create a blob storage account and blob container, and then upload the previously created text file to the blob container. 
This is the storage account and blob container that you'll give the online endpoint and managed identity access to. 

# [System-assigned (CLI)](#tab/system-identity-cli)

First, create a storage account.

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-sai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-sai-create-endpoint.yml
# </create_endpoint>
endpoint_status=`az ml online-endpoint show --name $ENDPOINT_NAME --query "provisioning_state" -o tsv`

echo $endpoint_status

if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else 
  echo "Endpoint creation failed"
  exit 1
fi

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# role assignment fails without sleep statement
sleep 60

# <get_system_identity>
system_identity=`az ml online-endpoint show --name $ENDPOINT_NAME --query "identity.principal_id" -o tsv`
# </get_system_identity>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $system_identity --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue --file endpoints/online/managed/managed-identities/2-sai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue
# </check_deploy_Status>

deploy_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $deploy_status
if [[ $deploy_status == "Succeeded" ]]
then
  echo "Deployment completed successfully"
else
  echo "Deployment failed"
  exit 1
fi

# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

```

Next, create the blob container in the storage account.

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-sai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-sai-create-endpoint.yml
# </create_endpoint>
endpoint_status=`az ml online-endpoint show --name $ENDPOINT_NAME --query "provisioning_state" -o tsv`

echo $endpoint_status

if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else 
  echo "Endpoint creation failed"
  exit 1
fi

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# role assignment fails without sleep statement
sleep 60

# <get_system_identity>
system_identity=`az ml online-endpoint show --name $ENDPOINT_NAME --query "identity.principal_id" -o tsv`
# </get_system_identity>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $system_identity --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue --file endpoints/online/managed/managed-identities/2-sai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue
# </check_deploy_Status>

deploy_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $deploy_status
if [[ $deploy_status == "Succeeded" ]]
then
  echo "Deployment completed successfully"
else
  echo "Deployment failed"
  exit 1
fi

# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

```

Then, upload your text file to the blob container.

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-sai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-sai-create-endpoint.yml
# </create_endpoint>
endpoint_status=`az ml online-endpoint show --name $ENDPOINT_NAME --query "provisioning_state" -o tsv`

echo $endpoint_status

if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else 
  echo "Endpoint creation failed"
  exit 1
fi

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# role assignment fails without sleep statement
sleep 60

# <get_system_identity>
system_identity=`az ml online-endpoint show --name $ENDPOINT_NAME --query "identity.principal_id" -o tsv`
# </get_system_identity>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $system_identity --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue --file endpoints/online/managed/managed-identities/2-sai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue
# </check_deploy_Status>

deploy_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $deploy_status
if [[ $deploy_status == "Succeeded" ]]
then
  echo "Deployment completed successfully"
else
  echo "Deployment failed"
  exit 1
fi

# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

```

# [User-assigned (CLI)](#tab/user-identity-cli)

First, create a storage account.  

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-uai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <set_user_identity_name>
export UAI_NAME="<USER_ASSIGNED_IDENTITY_NAME>"
# </set_user_identity_name>

export UAI_NAME=oep-user-identity-$TEST_ID

# <create_user_identity>
az identity create --name $UAI_NAME
# </create_user_identity>

# role assignment fails without sleep statement
sleep 60

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <get_user_identity_client_id>
uai_clientid=`az identity list --query "[?name=='$UAI_NAME'].clientId" -o tsv`
uai_principalid=`az identity list --query "[?name=='$UAI_NAME'].principalId" -o tsv`
# </get_user_identity_client_id>

# <get_user_identity_id>
uai_id=`az identity list --query "[?name=='$UAI_NAME'].id" -o tsv`
# </get_user_identity_id>

# <get_container_registry_id>
container_registry=`az ml workspace show --name $WORKSPACE --query container_registry -o tsv`
# </get_container_registry_id>

# <get_workspace_storage_id>
storage_account=`az ml workspace show --name $WORKSPACE --query storage_account -o tsv`
# </get_workspace_storage_id>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <give_permission_to_container_registry>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "AcrPull" --scope $container_registry
# </give_permission_to_container_registry>

# <give_permission_to_workspace_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "Storage Blob Data Reader" --scope $storage_account
# </give_permission_to_workspace_storage_account>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-uai-create-endpoint.yml --set identity.user_assigned_identities[0].resource_id=$uai_id
# </create_endpoint>

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue -f endpoints/online/managed/managed-identities/2-uai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME environment_variables.UAI_CLIENT_ID=$uai_clientid
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME -n blue
# </check_deploy_Status>

endpoint_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $endpoint_status
if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else
  echo "Endpoint creation failed"
  exit 1
fi


# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

# <delete_user_identity>
az identity delete --name $UAI_NAME
# </delete_user_identity>

```

You can also retrieve an existing storage account ID with the following. 

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-uai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <set_user_identity_name>
export UAI_NAME="<USER_ASSIGNED_IDENTITY_NAME>"
# </set_user_identity_name>

export UAI_NAME=oep-user-identity-$TEST_ID

# <create_user_identity>
az identity create --name $UAI_NAME
# </create_user_identity>

# role assignment fails without sleep statement
sleep 60

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <get_user_identity_client_id>
uai_clientid=`az identity list --query "[?name=='$UAI_NAME'].clientId" -o tsv`
uai_principalid=`az identity list --query "[?name=='$UAI_NAME'].principalId" -o tsv`
# </get_user_identity_client_id>

# <get_user_identity_id>
uai_id=`az identity list --query "[?name=='$UAI_NAME'].id" -o tsv`
# </get_user_identity_id>

# <get_container_registry_id>
container_registry=`az ml workspace show --name $WORKSPACE --query container_registry -o tsv`
# </get_container_registry_id>

# <get_workspace_storage_id>
storage_account=`az ml workspace show --name $WORKSPACE --query storage_account -o tsv`
# </get_workspace_storage_id>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <give_permission_to_container_registry>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "AcrPull" --scope $container_registry
# </give_permission_to_container_registry>

# <give_permission_to_workspace_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "Storage Blob Data Reader" --scope $storage_account
# </give_permission_to_workspace_storage_account>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-uai-create-endpoint.yml --set identity.user_assigned_identities[0].resource_id=$uai_id
# </create_endpoint>

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue -f endpoints/online/managed/managed-identities/2-uai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME environment_variables.UAI_CLIENT_ID=$uai_clientid
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME -n blue
# </check_deploy_Status>

endpoint_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $endpoint_status
if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else
  echo "Endpoint creation failed"
  exit 1
fi


# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

# <delete_user_identity>
az identity delete --name $UAI_NAME
# </delete_user_identity>

```

Next, create the blob container in the storage account. 

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-uai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <set_user_identity_name>
export UAI_NAME="<USER_ASSIGNED_IDENTITY_NAME>"
# </set_user_identity_name>

export UAI_NAME=oep-user-identity-$TEST_ID

# <create_user_identity>
az identity create --name $UAI_NAME
# </create_user_identity>

# role assignment fails without sleep statement
sleep 60

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <get_user_identity_client_id>
uai_clientid=`az identity list --query "[?name=='$UAI_NAME'].clientId" -o tsv`
uai_principalid=`az identity list --query "[?name=='$UAI_NAME'].principalId" -o tsv`
# </get_user_identity_client_id>

# <get_user_identity_id>
uai_id=`az identity list --query "[?name=='$UAI_NAME'].id" -o tsv`
# </get_user_identity_id>

# <get_container_registry_id>
container_registry=`az ml workspace show --name $WORKSPACE --query container_registry -o tsv`
# </get_container_registry_id>

# <get_workspace_storage_id>
storage_account=`az ml workspace show --name $WORKSPACE --query storage_account -o tsv`
# </get_workspace_storage_id>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <give_permission_to_container_registry>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "AcrPull" --scope $container_registry
# </give_permission_to_container_registry>

# <give_permission_to_workspace_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "Storage Blob Data Reader" --scope $storage_account
# </give_permission_to_workspace_storage_account>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-uai-create-endpoint.yml --set identity.user_assigned_identities[0].resource_id=$uai_id
# </create_endpoint>

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue -f endpoints/online/managed/managed-identities/2-uai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME environment_variables.UAI_CLIENT_ID=$uai_clientid
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME -n blue
# </check_deploy_Status>

endpoint_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $endpoint_status
if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else
  echo "Endpoint creation failed"
  exit 1
fi


# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

# <delete_user_identity>
az identity delete --name $UAI_NAME
# </delete_user_identity>

```

Then, upload file in container. 

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-uai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <set_user_identity_name>
export UAI_NAME="<USER_ASSIGNED_IDENTITY_NAME>"
# </set_user_identity_name>

export UAI_NAME=oep-user-identity-$TEST_ID

# <create_user_identity>
az identity create --name $UAI_NAME
# </create_user_identity>

# role assignment fails without sleep statement
sleep 60

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <get_user_identity_client_id>
uai_clientid=`az identity list --query "[?name=='$UAI_NAME'].clientId" -o tsv`
uai_principalid=`az identity list --query "[?name=='$UAI_NAME'].principalId" -o tsv`
# </get_user_identity_client_id>

# <get_user_identity_id>
uai_id=`az identity list --query "[?name=='$UAI_NAME'].id" -o tsv`
# </get_user_identity_id>

# <get_container_registry_id>
container_registry=`az ml workspace show --name $WORKSPACE --query container_registry -o tsv`
# </get_container_registry_id>

# <get_workspace_storage_id>
storage_account=`az ml workspace show --name $WORKSPACE --query storage_account -o tsv`
# </get_workspace_storage_id>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <give_permission_to_container_registry>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "AcrPull" --scope $container_registry
# </give_permission_to_container_registry>

# <give_permission_to_workspace_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "Storage Blob Data Reader" --scope $storage_account
# </give_permission_to_workspace_storage_account>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-uai-create-endpoint.yml --set identity.user_assigned_identities[0].resource_id=$uai_id
# </create_endpoint>

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue -f endpoints/online/managed/managed-identities/2-uai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME environment_variables.UAI_CLIENT_ID=$uai_clientid
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME -n blue
# </check_deploy_Status>

endpoint_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $endpoint_status
if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else
  echo "Endpoint creation failed"
  exit 1
fi


# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

# <delete_user_identity>
az identity delete --name $UAI_NAME
# </delete_user_identity>

```

# [System-assigned (Python)](#tab/system-identity-python)

First, get a handle to the `StorageManagementclient`:

```python
from azure.mgmt.storage import StorageManagementClient
from azure.storage.blob import ContainerClient
from azure.mgmt.storage.models import Sku, StorageAccountCreateParameters, BlobContainer

credential = AzureCliCredential()
storage_client = StorageManagementClient(
    credential=credential, subscription_id=subscription_id
)
```


Then, create a storage account: 

```python
storage_account_parameters = StorageAccountCreateParameters(
    sku=Sku(name="Standard_LRS"), kind="Storage", location=workspace_location
)

storage_account = storage_client.storage_accounts.begin_create(
    resource_group_name=resource_group,
    account_name=storage_account_name,
    parameters=storage_account_parameters,
).result()
```

Next, create the blob container in the storage account:

```python
blob_container = storage_client.blob_containers.create(
    resource_group_name=resource_group,
    account_name=storage_account_name,
    container_name=storage_container_name,
    blob_container=BlobContainer(),
)
```

Retrieve the storage account key and create a handle to the container with `ContainerClient`: 

```python
res = storage_client.storage_accounts.list_keys(
    resource_group_name=resource_group,
    account_name=storage_account_name,
)
key = res.keys[0].value

container_client = ContainerClient(
    account_url=storage_account.primary_endpoints.blob,
    container_name=storage_container_name,
    credential=key,
)
```

Then, upload a blob to the container with the `ContainerClient`:

```python
with open(file_name, "rb") as f:
    container_client.upload_blob(name=file_name, data=f.read())
```

# [User-assigned (Python)](#tab/user-identity-python)

First, get a handle to the `StorageManagementclient`:

```python
from azure.mgmt.storage import StorageManagementClient
from azure.storage.blob import ContainerClient
from azure.mgmt.storage.models import Sku, StorageAccountCreateParameters, BlobContainer

credential = AzureCliCredential()
storage_client = StorageManagementClient(
    credential=credential, subscription_id=subscription_id
)
```

Then, create a storage account: 

```python
storage_account_parameters = StorageAccountCreateParameters(
    sku=Sku(name="Standard_LRS"), kind="Storage", location=workspace_location
)

storage_account = storage_client.storage_accounts.begin_create(
    resource_group_name=resource_group,
    account_name=storage_account_name,
    parameters=storage_account_parameters,
).result()
```

Next, create the blob container in the storage account:

```python
blob_container = storage_client.blob_containers.create(
    resource_group_name=resource_group,
    account_name=storage_account_name,
    container_name=storage_container_name,
    blob_container=BlobContainer(),
)
```

Retrieve the storage account key and create a handle to the container with `ContainerClient`: 

```python
res = storage_client.storage_accounts.list_keys(
    resource_group_name=resource_group,
    account_name=storage_account_name,
)
key = res.keys[0].value

container_client = ContainerClient(
    account_url=storage_account.primary_endpoints.blob,
    container_name=storage_container_name,
    credential=key,
)
```

Then, upload a blob to the container with the `ContainerClient`:

```python
with open(file_name, "rb") as f:
    container_client.upload_blob(name=file_name, data=f.read())
```


## Create an online endpoint

The following code creates an online endpoint without specifying a deployment. 

> [!WARNING]
> The identity for an endpoint is immutable. During endpoint creation, you can associate it with a system-assigned identity (default) or a user-assigned identity. You can't change the identity after the endpoint has been created.

# [System-assigned (CLI)](#tab/system-identity-cli)
When you create an online endpoint, a system-assigned managed identity is created for the endpoint by default.

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-sai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-sai-create-endpoint.yml
# </create_endpoint>
endpoint_status=`az ml online-endpoint show --name $ENDPOINT_NAME --query "provisioning_state" -o tsv`

echo $endpoint_status

if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else 
  echo "Endpoint creation failed"
  exit 1
fi

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# role assignment fails without sleep statement
sleep 60

# <get_system_identity>
system_identity=`az ml online-endpoint show --name $ENDPOINT_NAME --query "identity.principal_id" -o tsv`
# </get_system_identity>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $system_identity --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue --file endpoints/online/managed/managed-identities/2-sai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue
# </check_deploy_Status>

deploy_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $deploy_status
if [[ $deploy_status == "Succeeded" ]]
then
  echo "Deployment completed successfully"
else
  echo "Deployment failed"
  exit 1
fi

# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

```

Check the status of the endpoint with the following.

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-sai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-sai-create-endpoint.yml
# </create_endpoint>
endpoint_status=`az ml online-endpoint show --name $ENDPOINT_NAME --query "provisioning_state" -o tsv`

echo $endpoint_status

if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else 
  echo "Endpoint creation failed"
  exit 1
fi

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# role assignment fails without sleep statement
sleep 60

# <get_system_identity>
system_identity=`az ml online-endpoint show --name $ENDPOINT_NAME --query "identity.principal_id" -o tsv`
# </get_system_identity>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $system_identity --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue --file endpoints/online/managed/managed-identities/2-sai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue
# </check_deploy_Status>

deploy_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $deploy_status
if [[ $deploy_status == "Succeeded" ]]
then
  echo "Deployment completed successfully"
else
  echo "Deployment failed"
  exit 1
fi

# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

```

If you encounter any issues, see [Troubleshooting online endpoints deployment and scoring](how-to-troubleshoot-managed-online-endpoints.md).

# [User-assigned (CLI)](#tab/user-identity-cli)

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-uai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <set_user_identity_name>
export UAI_NAME="<USER_ASSIGNED_IDENTITY_NAME>"
# </set_user_identity_name>

export UAI_NAME=oep-user-identity-$TEST_ID

# <create_user_identity>
az identity create --name $UAI_NAME
# </create_user_identity>

# role assignment fails without sleep statement
sleep 60

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <get_user_identity_client_id>
uai_clientid=`az identity list --query "[?name=='$UAI_NAME'].clientId" -o tsv`
uai_principalid=`az identity list --query "[?name=='$UAI_NAME'].principalId" -o tsv`
# </get_user_identity_client_id>

# <get_user_identity_id>
uai_id=`az identity list --query "[?name=='$UAI_NAME'].id" -o tsv`
# </get_user_identity_id>

# <get_container_registry_id>
container_registry=`az ml workspace show --name $WORKSPACE --query container_registry -o tsv`
# </get_container_registry_id>

# <get_workspace_storage_id>
storage_account=`az ml workspace show --name $WORKSPACE --query storage_account -o tsv`
# </get_workspace_storage_id>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <give_permission_to_container_registry>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "AcrPull" --scope $container_registry
# </give_permission_to_container_registry>

# <give_permission_to_workspace_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "Storage Blob Data Reader" --scope $storage_account
# </give_permission_to_workspace_storage_account>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-uai-create-endpoint.yml --set identity.user_assigned_identities[0].resource_id=$uai_id
# </create_endpoint>

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue -f endpoints/online/managed/managed-identities/2-uai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME environment_variables.UAI_CLIENT_ID=$uai_clientid
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME -n blue
# </check_deploy_Status>

endpoint_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $endpoint_status
if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else
  echo "Endpoint creation failed"
  exit 1
fi


# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

# <delete_user_identity>
az identity delete --name $UAI_NAME
# </delete_user_identity>

```

Check the status of the endpoint with the following.

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-uai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <set_user_identity_name>
export UAI_NAME="<USER_ASSIGNED_IDENTITY_NAME>"
# </set_user_identity_name>

export UAI_NAME=oep-user-identity-$TEST_ID

# <create_user_identity>
az identity create --name $UAI_NAME
# </create_user_identity>

# role assignment fails without sleep statement
sleep 60

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <get_user_identity_client_id>
uai_clientid=`az identity list --query "[?name=='$UAI_NAME'].clientId" -o tsv`
uai_principalid=`az identity list --query "[?name=='$UAI_NAME'].principalId" -o tsv`
# </get_user_identity_client_id>

# <get_user_identity_id>
uai_id=`az identity list --query "[?name=='$UAI_NAME'].id" -o tsv`
# </get_user_identity_id>

# <get_container_registry_id>
container_registry=`az ml workspace show --name $WORKSPACE --query container_registry -o tsv`
# </get_container_registry_id>

# <get_workspace_storage_id>
storage_account=`az ml workspace show --name $WORKSPACE --query storage_account -o tsv`
# </get_workspace_storage_id>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <give_permission_to_container_registry>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "AcrPull" --scope $container_registry
# </give_permission_to_container_registry>

# <give_permission_to_workspace_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "Storage Blob Data Reader" --scope $storage_account
# </give_permission_to_workspace_storage_account>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-uai-create-endpoint.yml --set identity.user_assigned_identities[0].resource_id=$uai_id
# </create_endpoint>

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue -f endpoints/online/managed/managed-identities/2-uai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME environment_variables.UAI_CLIENT_ID=$uai_clientid
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME -n blue
# </check_deploy_Status>

endpoint_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $endpoint_status
if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else
  echo "Endpoint creation failed"
  exit 1
fi


# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

# <delete_user_identity>
az identity delete --name $UAI_NAME
# </delete_user_identity>

```

If you encounter any issues, see [Troubleshooting online endpoints deployment and scoring](how-to-troubleshoot-managed-online-endpoints.md).

# [System-assigned (Python)](#tab/system-identity-python)

When you create an online endpoint, a system-assigned managed identity is created for the endpoint by default.

```python
ml_client.online_endpoints.begin_create_or_update(endpoint).result()
```

Check the status of the endpoint via the details of the deployed endpoint object with the following code:  

```python
endpoint = ml_client.online_endpoints.get(endpoint_name)
print(endpoint.identity.type)
print(endpoint.identity.principal_id)
```

If you encounter any issues, see [Troubleshooting online endpoints deployment and scoring](how-to-troubleshoot-managed-online-endpoints.md).


# [User-assigned (Python)](#tab/user-identity-python)

The following Python endpoint object: 

* Assigns the name by which you want to refer to the endpoint to the variable `endpoint_name. 
* Specifies the type of authorization to use to access the endpoint `auth-mode="key"`.
* Defines its identity as a ManagedServiceIdentity and specifies the Managed Identity created above as user-assigned. 

Define and deploy the endpoint: 

```python
from azure.ai.ml.entities import ManagedIdentityConfiguration, IdentityConfiguration

endpoint = ManagedOnlineEndpoint(
    name=endpoint_name,
    auth_mode="key",
    identity=IdentityConfiguration(
        type="user_assigned",
        user_assigned_identities=[
            ManagedIdentityConfiguration(resource_id=uai_identity.id)
        ],
    ),
)

ml_client.online_endpoints.begin_create_or_update(endpoint).result()
```


Check the status of the endpoint via the details of the deployed endpoint object with the following code:  

```python
endpoint = ml_client.online_endpoints.get(endpoint_name)
print(endpoint.identity.type)
print(endpoint.identity.user_assigned_identities)
```

If you encounter any issues, see [Troubleshooting online endpoints deployment and scoring](how-to-troubleshoot-managed-online-endpoints.md).


## Give access permission to the managed identity

>[!IMPORTANT] 
> Online endpoints require Azure Container Registry pull permission, AcrPull permission, to the container registry and Storage Blob Data Reader permission to the default datastore of the workspace.

You can allow the online endpoint permission to access your storage via its system-assigned managed identity or give permission to the user-assigned managed identity to access the storage account created in the previous section.

# [System-assigned (CLI)](#tab/system-identity-cli)

Retrieve the system-assigned managed identity that was created for your endpoint.

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-sai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-sai-create-endpoint.yml
# </create_endpoint>
endpoint_status=`az ml online-endpoint show --name $ENDPOINT_NAME --query "provisioning_state" -o tsv`

echo $endpoint_status

if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else 
  echo "Endpoint creation failed"
  exit 1
fi

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# role assignment fails without sleep statement
sleep 60

# <get_system_identity>
system_identity=`az ml online-endpoint show --name $ENDPOINT_NAME --query "identity.principal_id" -o tsv`
# </get_system_identity>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $system_identity --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue --file endpoints/online/managed/managed-identities/2-sai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue
# </check_deploy_Status>

deploy_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $deploy_status
if [[ $deploy_status == "Succeeded" ]]
then
  echo "Deployment completed successfully"
else
  echo "Deployment failed"
  exit 1
fi

# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

```

From here, you can give the system-assigned managed identity permission to access your storage.

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-sai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-sai-create-endpoint.yml
# </create_endpoint>
endpoint_status=`az ml online-endpoint show --name $ENDPOINT_NAME --query "provisioning_state" -o tsv`

echo $endpoint_status

if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else 
  echo "Endpoint creation failed"
  exit 1
fi

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# role assignment fails without sleep statement
sleep 60

# <get_system_identity>
system_identity=`az ml online-endpoint show --name $ENDPOINT_NAME --query "identity.principal_id" -o tsv`
# </get_system_identity>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $system_identity --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue --file endpoints/online/managed/managed-identities/2-sai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue
# </check_deploy_Status>

deploy_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $deploy_status
if [[ $deploy_status == "Succeeded" ]]
then
  echo "Deployment completed successfully"
else
  echo "Deployment failed"
  exit 1
fi

# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

```

# [User-assigned (CLI)](#tab/user-identity-cli)

Retrieve user-assigned managed identity client ID.

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-uai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <set_user_identity_name>
export UAI_NAME="<USER_ASSIGNED_IDENTITY_NAME>"
# </set_user_identity_name>

export UAI_NAME=oep-user-identity-$TEST_ID

# <create_user_identity>
az identity create --name $UAI_NAME
# </create_user_identity>

# role assignment fails without sleep statement
sleep 60

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <get_user_identity_client_id>
uai_clientid=`az identity list --query "[?name=='$UAI_NAME'].clientId" -o tsv`
uai_principalid=`az identity list --query "[?name=='$UAI_NAME'].principalId" -o tsv`
# </get_user_identity_client_id>

# <get_user_identity_id>
uai_id=`az identity list --query "[?name=='$UAI_NAME'].id" -o tsv`
# </get_user_identity_id>

# <get_container_registry_id>
container_registry=`az ml workspace show --name $WORKSPACE --query container_registry -o tsv`
# </get_container_registry_id>

# <get_workspace_storage_id>
storage_account=`az ml workspace show --name $WORKSPACE --query storage_account -o tsv`
# </get_workspace_storage_id>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <give_permission_to_container_registry>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "AcrPull" --scope $container_registry
# </give_permission_to_container_registry>

# <give_permission_to_workspace_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "Storage Blob Data Reader" --scope $storage_account
# </give_permission_to_workspace_storage_account>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-uai-create-endpoint.yml --set identity.user_assigned_identities[0].resource_id=$uai_id
# </create_endpoint>

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue -f endpoints/online/managed/managed-identities/2-uai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME environment_variables.UAI_CLIENT_ID=$uai_clientid
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME -n blue
# </check_deploy_Status>

endpoint_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $endpoint_status
if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else
  echo "Endpoint creation failed"
  exit 1
fi


# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

# <delete_user_identity>
az identity delete --name $UAI_NAME
# </delete_user_identity>

```

Retrieve the user-assigned managed identity ID.

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-uai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <set_user_identity_name>
export UAI_NAME="<USER_ASSIGNED_IDENTITY_NAME>"
# </set_user_identity_name>

export UAI_NAME=oep-user-identity-$TEST_ID

# <create_user_identity>
az identity create --name $UAI_NAME
# </create_user_identity>

# role assignment fails without sleep statement
sleep 60

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <get_user_identity_client_id>
uai_clientid=`az identity list --query "[?name=='$UAI_NAME'].clientId" -o tsv`
uai_principalid=`az identity list --query "[?name=='$UAI_NAME'].principalId" -o tsv`
# </get_user_identity_client_id>

# <get_user_identity_id>
uai_id=`az identity list --query "[?name=='$UAI_NAME'].id" -o tsv`
# </get_user_identity_id>

# <get_container_registry_id>
container_registry=`az ml workspace show --name $WORKSPACE --query container_registry -o tsv`
# </get_container_registry_id>

# <get_workspace_storage_id>
storage_account=`az ml workspace show --name $WORKSPACE --query storage_account -o tsv`
# </get_workspace_storage_id>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <give_permission_to_container_registry>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "AcrPull" --scope $container_registry
# </give_permission_to_container_registry>

# <give_permission_to_workspace_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "Storage Blob Data Reader" --scope $storage_account
# </give_permission_to_workspace_storage_account>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-uai-create-endpoint.yml --set identity.user_assigned_identities[0].resource_id=$uai_id
# </create_endpoint>

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue -f endpoints/online/managed/managed-identities/2-uai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME environment_variables.UAI_CLIENT_ID=$uai_clientid
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME -n blue
# </check_deploy_Status>

endpoint_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $endpoint_status
if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else
  echo "Endpoint creation failed"
  exit 1
fi


# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

# <delete_user_identity>
az identity delete --name $UAI_NAME
# </delete_user_identity>

```

Get the container registry associated with workspace.

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-uai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <set_user_identity_name>
export UAI_NAME="<USER_ASSIGNED_IDENTITY_NAME>"
# </set_user_identity_name>

export UAI_NAME=oep-user-identity-$TEST_ID

# <create_user_identity>
az identity create --name $UAI_NAME
# </create_user_identity>

# role assignment fails without sleep statement
sleep 60

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <get_user_identity_client_id>
uai_clientid=`az identity list --query "[?name=='$UAI_NAME'].clientId" -o tsv`
uai_principalid=`az identity list --query "[?name=='$UAI_NAME'].principalId" -o tsv`
# </get_user_identity_client_id>

# <get_user_identity_id>
uai_id=`az identity list --query "[?name=='$UAI_NAME'].id" -o tsv`
# </get_user_identity_id>

# <get_container_registry_id>
container_registry=`az ml workspace show --name $WORKSPACE --query container_registry -o tsv`
# </get_container_registry_id>

# <get_workspace_storage_id>
storage_account=`az ml workspace show --name $WORKSPACE --query storage_account -o tsv`
# </get_workspace_storage_id>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <give_permission_to_container_registry>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "AcrPull" --scope $container_registry
# </give_permission_to_container_registry>

# <give_permission_to_workspace_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "Storage Blob Data Reader" --scope $storage_account
# </give_permission_to_workspace_storage_account>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-uai-create-endpoint.yml --set identity.user_assigned_identities[0].resource_id=$uai_id
# </create_endpoint>

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue -f endpoints/online/managed/managed-identities/2-uai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME environment_variables.UAI_CLIENT_ID=$uai_clientid
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME -n blue
# </check_deploy_Status>

endpoint_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $endpoint_status
if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else
  echo "Endpoint creation failed"
  exit 1
fi


# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

# <delete_user_identity>
az identity delete --name $UAI_NAME
# </delete_user_identity>

```

Retrieve the default storage of the workspace.

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-uai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <set_user_identity_name>
export UAI_NAME="<USER_ASSIGNED_IDENTITY_NAME>"
# </set_user_identity_name>

export UAI_NAME=oep-user-identity-$TEST_ID

# <create_user_identity>
az identity create --name $UAI_NAME
# </create_user_identity>

# role assignment fails without sleep statement
sleep 60

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <get_user_identity_client_id>
uai_clientid=`az identity list --query "[?name=='$UAI_NAME'].clientId" -o tsv`
uai_principalid=`az identity list --query "[?name=='$UAI_NAME'].principalId" -o tsv`
# </get_user_identity_client_id>

# <get_user_identity_id>
uai_id=`az identity list --query "[?name=='$UAI_NAME'].id" -o tsv`
# </get_user_identity_id>

# <get_container_registry_id>
container_registry=`az ml workspace show --name $WORKSPACE --query container_registry -o tsv`
# </get_container_registry_id>

# <get_workspace_storage_id>
storage_account=`az ml workspace show --name $WORKSPACE --query storage_account -o tsv`
# </get_workspace_storage_id>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <give_permission_to_container_registry>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "AcrPull" --scope $container_registry
# </give_permission_to_container_registry>

# <give_permission_to_workspace_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "Storage Blob Data Reader" --scope $storage_account
# </give_permission_to_workspace_storage_account>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-uai-create-endpoint.yml --set identity.user_assigned_identities[0].resource_id=$uai_id
# </create_endpoint>

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue -f endpoints/online/managed/managed-identities/2-uai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME environment_variables.UAI_CLIENT_ID=$uai_clientid
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME -n blue
# </check_deploy_Status>

endpoint_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $endpoint_status
if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else
  echo "Endpoint creation failed"
  exit 1
fi


# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

# <delete_user_identity>
az identity delete --name $UAI_NAME
# </delete_user_identity>

```

Give permission of storage account to the user-assigned managed identity.  

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-uai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <set_user_identity_name>
export UAI_NAME="<USER_ASSIGNED_IDENTITY_NAME>"
# </set_user_identity_name>

export UAI_NAME=oep-user-identity-$TEST_ID

# <create_user_identity>
az identity create --name $UAI_NAME
# </create_user_identity>

# role assignment fails without sleep statement
sleep 60

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <get_user_identity_client_id>
uai_clientid=`az identity list --query "[?name=='$UAI_NAME'].clientId" -o tsv`
uai_principalid=`az identity list --query "[?name=='$UAI_NAME'].principalId" -o tsv`
# </get_user_identity_client_id>

# <get_user_identity_id>
uai_id=`az identity list --query "[?name=='$UAI_NAME'].id" -o tsv`
# </get_user_identity_id>

# <get_container_registry_id>
container_registry=`az ml workspace show --name $WORKSPACE --query container_registry -o tsv`
# </get_container_registry_id>

# <get_workspace_storage_id>
storage_account=`az ml workspace show --name $WORKSPACE --query storage_account -o tsv`
# </get_workspace_storage_id>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <give_permission_to_container_registry>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "AcrPull" --scope $container_registry
# </give_permission_to_container_registry>

# <give_permission_to_workspace_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "Storage Blob Data Reader" --scope $storage_account
# </give_permission_to_workspace_storage_account>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-uai-create-endpoint.yml --set identity.user_assigned_identities[0].resource_id=$uai_id
# </create_endpoint>

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue -f endpoints/online/managed/managed-identities/2-uai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME environment_variables.UAI_CLIENT_ID=$uai_clientid
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME -n blue
# </check_deploy_Status>

endpoint_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $endpoint_status
if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else
  echo "Endpoint creation failed"
  exit 1
fi


# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

# <delete_user_identity>
az identity delete --name $UAI_NAME
# </delete_user_identity>

```

Give permission of container registry to user assigned managed identity.

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-uai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <set_user_identity_name>
export UAI_NAME="<USER_ASSIGNED_IDENTITY_NAME>"
# </set_user_identity_name>

export UAI_NAME=oep-user-identity-$TEST_ID

# <create_user_identity>
az identity create --name $UAI_NAME
# </create_user_identity>

# role assignment fails without sleep statement
sleep 60

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <get_user_identity_client_id>
uai_clientid=`az identity list --query "[?name=='$UAI_NAME'].clientId" -o tsv`
uai_principalid=`az identity list --query "[?name=='$UAI_NAME'].principalId" -o tsv`
# </get_user_identity_client_id>

# <get_user_identity_id>
uai_id=`az identity list --query "[?name=='$UAI_NAME'].id" -o tsv`
# </get_user_identity_id>

# <get_container_registry_id>
container_registry=`az ml workspace show --name $WORKSPACE --query container_registry -o tsv`
# </get_container_registry_id>

# <get_workspace_storage_id>
storage_account=`az ml workspace show --name $WORKSPACE --query storage_account -o tsv`
# </get_workspace_storage_id>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <give_permission_to_container_registry>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "AcrPull" --scope $container_registry
# </give_permission_to_container_registry>

# <give_permission_to_workspace_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "Storage Blob Data Reader" --scope $storage_account
# </give_permission_to_workspace_storage_account>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-uai-create-endpoint.yml --set identity.user_assigned_identities[0].resource_id=$uai_id
# </create_endpoint>

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue -f endpoints/online/managed/managed-identities/2-uai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME environment_variables.UAI_CLIENT_ID=$uai_clientid
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME -n blue
# </check_deploy_Status>

endpoint_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $endpoint_status
if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else
  echo "Endpoint creation failed"
  exit 1
fi


# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

# <delete_user_identity>
az identity delete --name $UAI_NAME
# </delete_user_identity>

```

Give permission of default workspace storage to user-assigned managed identity.

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-uai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <set_user_identity_name>
export UAI_NAME="<USER_ASSIGNED_IDENTITY_NAME>"
# </set_user_identity_name>

export UAI_NAME=oep-user-identity-$TEST_ID

# <create_user_identity>
az identity create --name $UAI_NAME
# </create_user_identity>

# role assignment fails without sleep statement
sleep 60

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <get_user_identity_client_id>
uai_clientid=`az identity list --query "[?name=='$UAI_NAME'].clientId" -o tsv`
uai_principalid=`az identity list --query "[?name=='$UAI_NAME'].principalId" -o tsv`
# </get_user_identity_client_id>

# <get_user_identity_id>
uai_id=`az identity list --query "[?name=='$UAI_NAME'].id" -o tsv`
# </get_user_identity_id>

# <get_container_registry_id>
container_registry=`az ml workspace show --name $WORKSPACE --query container_registry -o tsv`
# </get_container_registry_id>

# <get_workspace_storage_id>
storage_account=`az ml workspace show --name $WORKSPACE --query storage_account -o tsv`
# </get_workspace_storage_id>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <give_permission_to_container_registry>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "AcrPull" --scope $container_registry
# </give_permission_to_container_registry>

# <give_permission_to_workspace_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "Storage Blob Data Reader" --scope $storage_account
# </give_permission_to_workspace_storage_account>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-uai-create-endpoint.yml --set identity.user_assigned_identities[0].resource_id=$uai_id
# </create_endpoint>

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue -f endpoints/online/managed/managed-identities/2-uai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME environment_variables.UAI_CLIENT_ID=$uai_clientid
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME -n blue
# </check_deploy_Status>

endpoint_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $endpoint_status
if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else
  echo "Endpoint creation failed"
  exit 1
fi


# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

# <delete_user_identity>
az identity delete --name $UAI_NAME
# </delete_user_identity>

```

# [System-assigned (Python)](#tab/system-identity-python)

First, make an `AuthorizationManagementClient` to list Role Definitions: 

```python
from azure.mgmt.authorization import AuthorizationManagementClient
from azure.mgmt.authorization.v2018_01_01_preview.models import RoleDefinition
import uuid

role_definition_client = AuthorizationManagementClient(
    credential=credential,
    subscription_id=subscription_id,
    api_version="2018-01-01-preview",
)
```

Now, initialize one to make Role Assignments: 

```python
from azure.mgmt.authorization.v2020_10_01_preview.models import (
    RoleAssignment,
    RoleAssignmentCreateParameters,
)

role_assignment_client = AuthorizationManagementClient(
    credential=credential,
    subscription_id=subscription_id,
    api_version="2020-10-01-preview",
)
```


Then, get the Principal ID of the System-assigned managed identity: 

```python
endpoint = ml_client.online_endpoints.get(endpoint_name)
system_principal_id = endpoint.identity.principal_id
```

Next, give assign the `Storage Blob Data Reader` role to the endpoint. The Role Definition is retrieved by name and passed along with the Principal ID of the endpoint. The role is applied at the scope of the storage account created above and allows the endpoint to read the file. 

```python
role_name = "Storage Blob Data Reader"
scope = storage_account.id

role_defs = role_definition_client.role_definitions.list(scope=scope)
role_def = next((r for r in role_defs if r.role_name == role_name))

role_assignment_client.role_assignments.create(
    scope=scope,
    role_assignment_name=str(uuid.uuid4()),
    parameters=RoleAssignmentCreateParameters(
        role_definition_id=role_def.id, principal_id=system_principal_id
    ),
)
```


# [User-assigned (Python)](#tab/user-identity-python)

First, make an `AuthorizationManagementClient` to list Role Definitions: 

```python
from azure.mgmt.authorization import AuthorizationManagementClient
from azure.mgmt.authorization.v2018_01_01_preview.models import RoleDefinition
import uuid

role_definition_client = AuthorizationManagementClient(
    credential=credential,
    subscription_id=subscription_id,
    api_version="2018-01-01-preview",
)
```

Now, initialize one to make Role Assignments: 

```python
from azure.mgmt.authorization.v2020_10_01_preview.models import (
    RoleAssignment,
    RoleAssignmentCreateParameters,
)

role_assignment_client = AuthorizationManagementClient(
    credential=credential,
    subscription_id=subscription_id,
    api_version="2020-10-01-preview",
)
```

Then, get the Principal ID and Client ID of the User-assigned managed identity. To assign roles, we only need the Principal ID. However, we will use the Client ID to fill the `UAI_CLIENT_ID` placeholder environment variable before creating the deployment.

```python
uai_identity = msi_client.user_assigned_identities.get(
    resource_group_name=resource_group, resource_name=uai_name
)
uai_principal_id = uai_identity.principal_id
uai_client_id = uai_identity.client_id
```

Next, assign the `Storage Blob Data Reader` role to the endpoint. The Role Definition is retrieved by name and passed along with the Principal ID of the endpoint. The role is applied at the scope of the storage account created above to allow the endpoint to read the file. 

```python
role_name = "Storage Blob Data Reader"
scope = storage_account.id

role_defs = role_definition_client.role_definitions.list(scope=scope)
role_def = next((r for r in role_defs if r.role_name == role_name))

role_assignment_client.role_assignments.create(
    scope=scope,
    role_assignment_name=str(uuid.uuid4()),
    parameters=RoleAssignmentCreateParameters(
        role_definition_id=role_def.id,
        principal_id=uai_principal_id,
        principal_type="ServicePrincipal",
    ),
)
```

For the next two permissions, we'll need the workspace and container registry objects: 

```python
workspace = ml_client.workspaces.get(workspace_name)
container_registry = workspace.container_registry
```

Next, assign the `AcrPull` role to the User-assigned identity. This role allows images to be pulled from an Azure Container Registry. The scope is applied at the level of the container registry associated with the workspace.

```python
role_name = "AcrPull"
scope = container_registry

role_defs = role_definition_client.role_definitions.list(scope=scope)
role_def = next((r for r in role_defs if r.role_name == role_name))

role_assignment_client.role_assignments.create(
    scope=scope,
    role_assignment_name=str(uuid.uuid4()),
    parameters=RoleAssignmentCreateParameters(
        role_definition_id=role_def.id,
        principal_id=uai_principal_id,
        principal_type="ServicePrincipal",
    ),
)
```

Finally, assign the `Storage Blob Data Reader` role to the endpoint at the workspace storage account scope. This role assignment will allow the endpoint to read blobs in the workspace storage account as well as the newly created storage account.

The role has the same name and capabilities as the first role assigned above, however it is applied at a different scope and has a different ID. 

```python
role_name = "Storage Blob Data Reader"
scope = workspace.storage_account

role_defs = role_definition_client.role_definitions.list(scope=scope)
role_def = next((r for r in role_defs if r.role_name == role_name))

role_assignment_client.role_assignments.create(
    scope=scope,
    role_assignment_name=str(uuid.uuid4()),
    parameters=RoleAssignmentCreateParameters(
        role_definition_id=role_def.id,
        principal_id=uai_principal_id,
        principal_type="ServicePrincipal",
    ),
)
```


## Scoring script to access Azure resource

Refer to the following script to understand how to use your identity token to access Azure resources, in this scenario, the storage account created in previous sections. 

```python
import os
import logging
import json
import numpy
import joblib
import requests
from azure.identity import ManagedIdentityCredential
from azure.storage.blob import BlobClient


def access_blob_storage_sdk():
    credential = ManagedIdentityCredential(client_id=os.getenv("UAI_CLIENT_ID"))
    storage_account = os.getenv("STORAGE_ACCOUNT_NAME")
    storage_container = os.getenv("STORAGE_CONTAINER_NAME")
    file_name = os.getenv("FILE_NAME")

    blob_client = BlobClient(
        account_url=f"https://{storage_account}.blob.core.windows.net/",
        container_name=storage_container,
        blob_name=file_name,
        credential=credential,
    )
    blob_contents = blob_client.download_blob().content_as_text()
    logging.info(f"Blob contains: {blob_contents}")


def get_token_rest():
    """
    Retrieve an access token via REST.
    """

    access_token = None
    msi_endpoint = os.environ.get("MSI_ENDPOINT", None)
    msi_secret = os.environ.get("MSI_SECRET", None)

    # If UAI_CLIENT_ID is provided then assume that endpoint was created with user assigned identity,
    # # otherwise system assigned identity deployment.
    client_id = os.environ.get("UAI_CLIENT_ID", None)
    if client_id is not None:
        token_url = (
            msi_endpoint + f"?clientid={client_id}&resource=https://storage.azure.com/"
        )
    else:
        token_url = msi_endpoint + f"?resource=https://storage.azure.com/"

    logging.info("Trying to get identity token...")
    headers = {"secret": msi_secret, "Metadata": "true"}
    resp = requests.get(token_url, headers=headers)
    resp.raise_for_status()
    access_token = resp.json()["access_token"]
    logging.info("Retrieved token successfully.")
    return access_token


def access_blob_storage_rest():
    """
    Access a blob via REST.
    """

    logging.info("Trying to access blob storage...")
    storage_account = os.environ.get("STORAGE_ACCOUNT_NAME")
    storage_container = os.environ.get("STORAGE_CONTAINER_NAME")
    file_name = os.environ.get("FILE_NAME")
    logging.info(
        f"storage_account: {storage_account}, container: {storage_container}, filename: {file_name}"
    )
    token = get_token_rest()

    blob_url = f"https://{storage_account}.blob.core.windows.net/{storage_container}/{file_name}?api-version=2019-04-01"
    auth_headers = {
        "Authorization": f"Bearer {token}",
        "x-ms-blob-type": "BlockBlob",
        "x-ms-version": "2019-02-02",
    }
    resp = requests.get(blob_url, headers=auth_headers)
    resp.raise_for_status()
    logging.info(f"Blob contains: {resp.text}")


def init():
    global model
    # AZUREML_MODEL_DIR is an environment variable created during deployment.
    # It is the path to the model folder (./azureml-models/$MODEL_NAME/$VERSION)
    # For multiple models, it points to the folder containing all deployed models (./azureml-models)
    # Please provide your model's folder name if there is one
    model_path = os.path.join(
        os.getenv("AZUREML_MODEL_DIR"), "model/sklearn_regression_model.pkl"
    )
    # deserialize the model file back into a sklearn model
    model = joblib.load(model_path)
    logging.info("Model loaded")

    # Access Azure resource (Blob storage) using system assigned identity token
    access_blob_storage_rest()
    access_blob_storage_sdk()

    logging.info("Init complete")


# note you can pass in multiple rows for scoring
def run(raw_data):
    logging.info("Request received")
    data = json.loads(raw_data)["data"]
    data = numpy.array(data)
    result = model.predict(data)
    logging.info("Request processed")
    return result.tolist()

```

## Create a deployment with your configuration

Create a deployment that's associated with the online endpoint. [Learn more about deploying to online endpoints](how-to-deploy-online-endpoints.md).

>[!WARNING]
> This deployment can take approximately 8-14 minutes depending on whether the underlying environment/image is being built for the first time. Subsequent deployments using the same environment will go quicker.

# [System-assigned (CLI)](#tab/system-identity-cli)

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-sai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-sai-create-endpoint.yml
# </create_endpoint>
endpoint_status=`az ml online-endpoint show --name $ENDPOINT_NAME --query "provisioning_state" -o tsv`

echo $endpoint_status

if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else 
  echo "Endpoint creation failed"
  exit 1
fi

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# role assignment fails without sleep statement
sleep 60

# <get_system_identity>
system_identity=`az ml online-endpoint show --name $ENDPOINT_NAME --query "identity.principal_id" -o tsv`
# </get_system_identity>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $system_identity --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue --file endpoints/online/managed/managed-identities/2-sai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue
# </check_deploy_Status>

deploy_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $deploy_status
if [[ $deploy_status == "Succeeded" ]]
then
  echo "Deployment completed successfully"
else
  echo "Deployment failed"
  exit 1
fi

# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

```

>[!NOTE]
> The value of the `--name` argument may override the `name` key inside the YAML file.

Check the status of the deployment.

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-sai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-sai-create-endpoint.yml
# </create_endpoint>
endpoint_status=`az ml online-endpoint show --name $ENDPOINT_NAME --query "provisioning_state" -o tsv`

echo $endpoint_status

if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else 
  echo "Endpoint creation failed"
  exit 1
fi

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# role assignment fails without sleep statement
sleep 60

# <get_system_identity>
system_identity=`az ml online-endpoint show --name $ENDPOINT_NAME --query "identity.principal_id" -o tsv`
# </get_system_identity>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $system_identity --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue --file endpoints/online/managed/managed-identities/2-sai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue
# </check_deploy_Status>

deploy_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $deploy_status
if [[ $deploy_status == "Succeeded" ]]
then
  echo "Deployment completed successfully"
else
  echo "Deployment failed"
  exit 1
fi

# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

```

To refine the above query to only return specific data, see [Query Azure CLI command output](/cli/azure/query-azure-cli).

> [!NOTE]
> The init method in the scoring script reads the file from your storage account using the system-assigned managed identity token.

To check the init method output, see the deployment log with the following code. 

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-sai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-sai-create-endpoint.yml
# </create_endpoint>
endpoint_status=`az ml online-endpoint show --name $ENDPOINT_NAME --query "provisioning_state" -o tsv`

echo $endpoint_status

if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else 
  echo "Endpoint creation failed"
  exit 1
fi

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# role assignment fails without sleep statement
sleep 60

# <get_system_identity>
system_identity=`az ml online-endpoint show --name $ENDPOINT_NAME --query "identity.principal_id" -o tsv`
# </get_system_identity>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $system_identity --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue --file endpoints/online/managed/managed-identities/2-sai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue
# </check_deploy_Status>

deploy_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $deploy_status
if [[ $deploy_status == "Succeeded" ]]
then
  echo "Deployment completed successfully"
else
  echo "Deployment failed"
  exit 1
fi

# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

```


# [User-assigned (CLI)](#tab/user-identity-cli)

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-uai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <set_user_identity_name>
export UAI_NAME="<USER_ASSIGNED_IDENTITY_NAME>"
# </set_user_identity_name>

export UAI_NAME=oep-user-identity-$TEST_ID

# <create_user_identity>
az identity create --name $UAI_NAME
# </create_user_identity>

# role assignment fails without sleep statement
sleep 60

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <get_user_identity_client_id>
uai_clientid=`az identity list --query "[?name=='$UAI_NAME'].clientId" -o tsv`
uai_principalid=`az identity list --query "[?name=='$UAI_NAME'].principalId" -o tsv`
# </get_user_identity_client_id>

# <get_user_identity_id>
uai_id=`az identity list --query "[?name=='$UAI_NAME'].id" -o tsv`
# </get_user_identity_id>

# <get_container_registry_id>
container_registry=`az ml workspace show --name $WORKSPACE --query container_registry -o tsv`
# </get_container_registry_id>

# <get_workspace_storage_id>
storage_account=`az ml workspace show --name $WORKSPACE --query storage_account -o tsv`
# </get_workspace_storage_id>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <give_permission_to_container_registry>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "AcrPull" --scope $container_registry
# </give_permission_to_container_registry>

# <give_permission_to_workspace_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "Storage Blob Data Reader" --scope $storage_account
# </give_permission_to_workspace_storage_account>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-uai-create-endpoint.yml --set identity.user_assigned_identities[0].resource_id=$uai_id
# </create_endpoint>

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue -f endpoints/online/managed/managed-identities/2-uai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME environment_variables.UAI_CLIENT_ID=$uai_clientid
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME -n blue
# </check_deploy_Status>

endpoint_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $endpoint_status
if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else
  echo "Endpoint creation failed"
  exit 1
fi


# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

# <delete_user_identity>
az identity delete --name $UAI_NAME
# </delete_user_identity>

```

>[!Note]
> The value of the `--name` argument may override the `name` key inside the YAML file.

Once the command executes, you can check the status of the deployment.

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-uai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <set_user_identity_name>
export UAI_NAME="<USER_ASSIGNED_IDENTITY_NAME>"
# </set_user_identity_name>

export UAI_NAME=oep-user-identity-$TEST_ID

# <create_user_identity>
az identity create --name $UAI_NAME
# </create_user_identity>

# role assignment fails without sleep statement
sleep 60

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <get_user_identity_client_id>
uai_clientid=`az identity list --query "[?name=='$UAI_NAME'].clientId" -o tsv`
uai_principalid=`az identity list --query "[?name=='$UAI_NAME'].principalId" -o tsv`
# </get_user_identity_client_id>

# <get_user_identity_id>
uai_id=`az identity list --query "[?name=='$UAI_NAME'].id" -o tsv`
# </get_user_identity_id>

# <get_container_registry_id>
container_registry=`az ml workspace show --name $WORKSPACE --query container_registry -o tsv`
# </get_container_registry_id>

# <get_workspace_storage_id>
storage_account=`az ml workspace show --name $WORKSPACE --query storage_account -o tsv`
# </get_workspace_storage_id>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <give_permission_to_container_registry>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "AcrPull" --scope $container_registry
# </give_permission_to_container_registry>

# <give_permission_to_workspace_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "Storage Blob Data Reader" --scope $storage_account
# </give_permission_to_workspace_storage_account>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-uai-create-endpoint.yml --set identity.user_assigned_identities[0].resource_id=$uai_id
# </create_endpoint>

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue -f endpoints/online/managed/managed-identities/2-uai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME environment_variables.UAI_CLIENT_ID=$uai_clientid
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME -n blue
# </check_deploy_Status>

endpoint_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $endpoint_status
if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else
  echo "Endpoint creation failed"
  exit 1
fi


# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

# <delete_user_identity>
az identity delete --name $UAI_NAME
# </delete_user_identity>

```

To refine the above query to only return specific data, see [Query Azure CLI command output](/cli/azure/query-azure-cli).

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-uai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <set_user_identity_name>
export UAI_NAME="<USER_ASSIGNED_IDENTITY_NAME>"
# </set_user_identity_name>

export UAI_NAME=oep-user-identity-$TEST_ID

# <create_user_identity>
az identity create --name $UAI_NAME
# </create_user_identity>

# role assignment fails without sleep statement
sleep 60

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <get_user_identity_client_id>
uai_clientid=`az identity list --query "[?name=='$UAI_NAME'].clientId" -o tsv`
uai_principalid=`az identity list --query "[?name=='$UAI_NAME'].principalId" -o tsv`
# </get_user_identity_client_id>

# <get_user_identity_id>
uai_id=`az identity list --query "[?name=='$UAI_NAME'].id" -o tsv`
# </get_user_identity_id>

# <get_container_registry_id>
container_registry=`az ml workspace show --name $WORKSPACE --query container_registry -o tsv`
# </get_container_registry_id>

# <get_workspace_storage_id>
storage_account=`az ml workspace show --name $WORKSPACE --query storage_account -o tsv`
# </get_workspace_storage_id>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <give_permission_to_container_registry>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "AcrPull" --scope $container_registry
# </give_permission_to_container_registry>

# <give_permission_to_workspace_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "Storage Blob Data Reader" --scope $storage_account
# </give_permission_to_workspace_storage_account>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-uai-create-endpoint.yml --set identity.user_assigned_identities[0].resource_id=$uai_id
# </create_endpoint>

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue -f endpoints/online/managed/managed-identities/2-uai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME environment_variables.UAI_CLIENT_ID=$uai_clientid
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME -n blue
# </check_deploy_Status>

endpoint_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $endpoint_status
if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else
  echo "Endpoint creation failed"
  exit 1
fi


# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

# <delete_user_identity>
az identity delete --name $UAI_NAME
# </delete_user_identity>

```

> [!NOTE]
> The init method in the scoring script reads the file from your storage account using the user-assigned managed identity token.

To check the init method output, see the deployment log with the following code. 

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-uai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <set_user_identity_name>
export UAI_NAME="<USER_ASSIGNED_IDENTITY_NAME>"
# </set_user_identity_name>

export UAI_NAME=oep-user-identity-$TEST_ID

# <create_user_identity>
az identity create --name $UAI_NAME
# </create_user_identity>

# role assignment fails without sleep statement
sleep 60

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <get_user_identity_client_id>
uai_clientid=`az identity list --query "[?name=='$UAI_NAME'].clientId" -o tsv`
uai_principalid=`az identity list --query "[?name=='$UAI_NAME'].principalId" -o tsv`
# </get_user_identity_client_id>

# <get_user_identity_id>
uai_id=`az identity list --query "[?name=='$UAI_NAME'].id" -o tsv`
# </get_user_identity_id>

# <get_container_registry_id>
container_registry=`az ml workspace show --name $WORKSPACE --query container_registry -o tsv`
# </get_container_registry_id>

# <get_workspace_storage_id>
storage_account=`az ml workspace show --name $WORKSPACE --query storage_account -o tsv`
# </get_workspace_storage_id>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <give_permission_to_container_registry>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "AcrPull" --scope $container_registry
# </give_permission_to_container_registry>

# <give_permission_to_workspace_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "Storage Blob Data Reader" --scope $storage_account
# </give_permission_to_workspace_storage_account>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-uai-create-endpoint.yml --set identity.user_assigned_identities[0].resource_id=$uai_id
# </create_endpoint>

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue -f endpoints/online/managed/managed-identities/2-uai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME environment_variables.UAI_CLIENT_ID=$uai_clientid
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME -n blue
# </check_deploy_Status>

endpoint_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $endpoint_status
if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else
  echo "Endpoint creation failed"
  exit 1
fi


# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

# <delete_user_identity>
az identity delete --name $UAI_NAME
# </delete_user_identity>

```

# [System-assigned (Python)](#tab/system-identity-python)

First, create the deployment:  

```python
ml_client.online_deployments.begin_create_or_update(deployment).result()
```

Once deployment completes, check its status and confirm its identity details: 

```python
deployment = ml_client.online_deployments.get(
    endpoint_name=endpoint_name, name=deployment.name
)
print(deployment)
```

> [!NOTE]
> The init method in the scoring script reads the file from your storage account using the system-assigned managed identity token.

To check the init method output, see the deployment log with the following code. 

```python
ml_client.online_deployments.get_logs(deployment.name, deployment.endpoint_name, 1000)
```

Now that the deployment is confirmed, set the traffic to 100%: 

```python
endpoint.traffic = {str(deployment.name): 100}
ml_client.begin_create_or_update(endpoint).result()
```

# [User-assigned (Python)](#tab/user-identity-python)

Before we deploy, update the `UAI_CLIENT_ID` environment variable placeholder. 

```python
deployment.environment_variables["UAI_CLIENT_ID"] = uai_client_id
```

Now, create the deployment: 

```python
ml_client.online_deployments.begin_create_or_update(deployment).result()
```

Once deployment completes, check its status and confirm its identity details: 

```python
deployment = ml_client.online_deployments.get(
    endpoint_name=endpoint_name, name=deployment.name
)
print(deployment)
```

> [!NOTE]
> The init method in the scoring script reads the file from your storage account using the user-assigned managed identity token.

To check the init method output, see the deployment log with the following code. 

```python
ml_client.online_deployments.get_logs(deployment.name, deployment.endpoint_name, 1000)
```

Now that the deployment is confirmed, set the traffic to 100%: 

```python
endpoint.traffic = {str(deployment.name): 100}
ml_client.begin_create_or_update(endpoint).result()
```


When your deployment completes,  the model, the environment, and the endpoint are registered to your Azure Machine Learning workspace.

## Test the endpoint

Once your online endpoint is deployed, test and confirm its operation with a request. Details of inferencing vary from model to model. For this guide, the JSON query parameters look like: 

```json
{"data": [
    [1,2,3,4,5,6,7,8,9,10], 
    [10,9,8,7,6,5,4,3,2,1]
]}
```

To call your endpoint, run:

# [System-assigned (CLI)](#tab/system-identity-cli)

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-sai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-sai-create-endpoint.yml
# </create_endpoint>
endpoint_status=`az ml online-endpoint show --name $ENDPOINT_NAME --query "provisioning_state" -o tsv`

echo $endpoint_status

if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else 
  echo "Endpoint creation failed"
  exit 1
fi

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# role assignment fails without sleep statement
sleep 60

# <get_system_identity>
system_identity=`az ml online-endpoint show --name $ENDPOINT_NAME --query "identity.principal_id" -o tsv`
# </get_system_identity>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $system_identity --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue --file endpoints/online/managed/managed-identities/2-sai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue
# </check_deploy_Status>

deploy_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $deploy_status
if [[ $deploy_status == "Succeeded" ]]
then
  echo "Deployment completed successfully"
else
  echo "Deployment failed"
  exit 1
fi

# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

```

# [User-assigned (CLI)](#tab/user-identity-cli)

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-uai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <set_user_identity_name>
export UAI_NAME="<USER_ASSIGNED_IDENTITY_NAME>"
# </set_user_identity_name>

export UAI_NAME=oep-user-identity-$TEST_ID

# <create_user_identity>
az identity create --name $UAI_NAME
# </create_user_identity>

# role assignment fails without sleep statement
sleep 60

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <get_user_identity_client_id>
uai_clientid=`az identity list --query "[?name=='$UAI_NAME'].clientId" -o tsv`
uai_principalid=`az identity list --query "[?name=='$UAI_NAME'].principalId" -o tsv`
# </get_user_identity_client_id>

# <get_user_identity_id>
uai_id=`az identity list --query "[?name=='$UAI_NAME'].id" -o tsv`
# </get_user_identity_id>

# <get_container_registry_id>
container_registry=`az ml workspace show --name $WORKSPACE --query container_registry -o tsv`
# </get_container_registry_id>

# <get_workspace_storage_id>
storage_account=`az ml workspace show --name $WORKSPACE --query storage_account -o tsv`
# </get_workspace_storage_id>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <give_permission_to_container_registry>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "AcrPull" --scope $container_registry
# </give_permission_to_container_registry>

# <give_permission_to_workspace_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "Storage Blob Data Reader" --scope $storage_account
# </give_permission_to_workspace_storage_account>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-uai-create-endpoint.yml --set identity.user_assigned_identities[0].resource_id=$uai_id
# </create_endpoint>

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue -f endpoints/online/managed/managed-identities/2-uai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME environment_variables.UAI_CLIENT_ID=$uai_clientid
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME -n blue
# </check_deploy_Status>

endpoint_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $endpoint_status
if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else
  echo "Endpoint creation failed"
  exit 1
fi


# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

# <delete_user_identity>
az identity delete --name $UAI_NAME
# </delete_user_identity>

```

# [System-assigned (Python)](#tab/system-identity-python)

```python
sample_data = "../../model-1/sample-request.json"
ml_client.online_endpoints.invoke(endpoint_name=endpoint_name, request_file=sample_data)
```

# [User-assigned (Python)](#tab/user-identity-python)


```python
sample_data = "../../model-1/sample-request.json"
ml_client.online_endpoints.invoke(endpoint_name=endpoint_name, request_file=sample_data)
```


## Delete the endpoint and storage account

If you don't plan to continue using the deployed online endpoint and storage, delete them to reduce costs. When you delete the endpoint, all of its associated deployments are deleted as well.

# [System-assigned (CLI)](#tab/system-identity-cli)
 
```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-sai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-sai-create-endpoint.yml
# </create_endpoint>
endpoint_status=`az ml online-endpoint show --name $ENDPOINT_NAME --query "provisioning_state" -o tsv`

echo $endpoint_status

if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else 
  echo "Endpoint creation failed"
  exit 1
fi

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# role assignment fails without sleep statement
sleep 60

# <get_system_identity>
system_identity=`az ml online-endpoint show --name $ENDPOINT_NAME --query "identity.principal_id" -o tsv`
# </get_system_identity>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $system_identity --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue --file endpoints/online/managed/managed-identities/2-sai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue
# </check_deploy_Status>

deploy_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $deploy_status
if [[ $deploy_status == "Succeeded" ]]
then
  echo "Deployment completed successfully"
else
  echo "Deployment failed"
  exit 1
fi

# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

```
```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-sai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-sai-create-endpoint.yml
# </create_endpoint>
endpoint_status=`az ml online-endpoint show --name $ENDPOINT_NAME --query "provisioning_state" -o tsv`

echo $endpoint_status

if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else 
  echo "Endpoint creation failed"
  exit 1
fi

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# role assignment fails without sleep statement
sleep 60

# <get_system_identity>
system_identity=`az ml online-endpoint show --name $ENDPOINT_NAME --query "identity.principal_id" -o tsv`
# </get_system_identity>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $system_identity --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue --file endpoints/online/managed/managed-identities/2-sai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue
# </check_deploy_Status>

deploy_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $deploy_status
if [[ $deploy_status == "Succeeded" ]]
then
  echo "Deployment completed successfully"
else
  echo "Deployment failed"
  exit 1
fi

# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

```

# [User-assigned (CLI)](#tab/user-identity-cli)

```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-uai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <set_user_identity_name>
export UAI_NAME="<USER_ASSIGNED_IDENTITY_NAME>"
# </set_user_identity_name>

export UAI_NAME=oep-user-identity-$TEST_ID

# <create_user_identity>
az identity create --name $UAI_NAME
# </create_user_identity>

# role assignment fails without sleep statement
sleep 60

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <get_user_identity_client_id>
uai_clientid=`az identity list --query "[?name=='$UAI_NAME'].clientId" -o tsv`
uai_principalid=`az identity list --query "[?name=='$UAI_NAME'].principalId" -o tsv`
# </get_user_identity_client_id>

# <get_user_identity_id>
uai_id=`az identity list --query "[?name=='$UAI_NAME'].id" -o tsv`
# </get_user_identity_id>

# <get_container_registry_id>
container_registry=`az ml workspace show --name $WORKSPACE --query container_registry -o tsv`
# </get_container_registry_id>

# <get_workspace_storage_id>
storage_account=`az ml workspace show --name $WORKSPACE --query storage_account -o tsv`
# </get_workspace_storage_id>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <give_permission_to_container_registry>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "AcrPull" --scope $container_registry
# </give_permission_to_container_registry>

# <give_permission_to_workspace_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "Storage Blob Data Reader" --scope $storage_account
# </give_permission_to_workspace_storage_account>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-uai-create-endpoint.yml --set identity.user_assigned_identities[0].resource_id=$uai_id
# </create_endpoint>

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue -f endpoints/online/managed/managed-identities/2-uai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME environment_variables.UAI_CLIENT_ID=$uai_clientid
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME -n blue
# </check_deploy_Status>

endpoint_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $endpoint_status
if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else
  echo "Endpoint creation failed"
  exit 1
fi


# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

# <delete_user_identity>
az identity delete --name $UAI_NAME
# </delete_user_identity>

```
```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-uai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <set_user_identity_name>
export UAI_NAME="<USER_ASSIGNED_IDENTITY_NAME>"
# </set_user_identity_name>

export UAI_NAME=oep-user-identity-$TEST_ID

# <create_user_identity>
az identity create --name $UAI_NAME
# </create_user_identity>

# role assignment fails without sleep statement
sleep 60

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <get_user_identity_client_id>
uai_clientid=`az identity list --query "[?name=='$UAI_NAME'].clientId" -o tsv`
uai_principalid=`az identity list --query "[?name=='$UAI_NAME'].principalId" -o tsv`
# </get_user_identity_client_id>

# <get_user_identity_id>
uai_id=`az identity list --query "[?name=='$UAI_NAME'].id" -o tsv`
# </get_user_identity_id>

# <get_container_registry_id>
container_registry=`az ml workspace show --name $WORKSPACE --query container_registry -o tsv`
# </get_container_registry_id>

# <get_workspace_storage_id>
storage_account=`az ml workspace show --name $WORKSPACE --query storage_account -o tsv`
# </get_workspace_storage_id>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <give_permission_to_container_registry>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "AcrPull" --scope $container_registry
# </give_permission_to_container_registry>

# <give_permission_to_workspace_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "Storage Blob Data Reader" --scope $storage_account
# </give_permission_to_workspace_storage_account>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-uai-create-endpoint.yml --set identity.user_assigned_identities[0].resource_id=$uai_id
# </create_endpoint>

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue -f endpoints/online/managed/managed-identities/2-uai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME environment_variables.UAI_CLIENT_ID=$uai_clientid
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME -n blue
# </check_deploy_Status>

endpoint_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $endpoint_status
if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else
  echo "Endpoint creation failed"
  exit 1
fi


# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

# <delete_user_identity>
az identity delete --name $UAI_NAME
# </delete_user_identity>

```
```azurecli
set -e


# <set_variables>
export WORKSPACE="<WORKSPACE_NAME>"
export LOCATION="<WORKSPACE_LOCATION>"
export ENDPOINT_NAME="<ENDPOINT_NAME>"
# </set_variables>

export WORKSPACE=$(az config get --query "defaults[?name == 'workspace'].value" -o tsv)
export LOCATION=$(az group show --query location -o tsv)
export TEST_ID=`echo $RANDOM`
export ENDPOINT_NAME=endpt-uai-$TEST_ID

# <configure_storage_names>
export STORAGE_ACCOUNT_NAME="<BLOB_STORAGE_TO_ACCESS>"
export STORAGE_CONTAINER_NAME="<CONTAINER_TO_ACCESS>"
export FILE_NAME="<FILE_TO_ACCESS>"
# </configure_storage_names>

export STORAGE_ACCOUNT_NAME=oepstorage$TEST_ID
export STORAGE_CONTAINER_NAME="hellocontainer"
export FILE_NAME="hello.txt"

# <set_user_identity_name>
export UAI_NAME="<USER_ASSIGNED_IDENTITY_NAME>"
# </set_user_identity_name>

export UAI_NAME=oep-user-identity-$TEST_ID

# <create_user_identity>
az identity create --name $UAI_NAME
# </create_user_identity>

# role assignment fails without sleep statement
sleep 60

# <create_storage_account>
az storage account create --name $STORAGE_ACCOUNT_NAME --location $LOCATION
# </create_storage_account>

# <get_storage_account_id>
storage_id=`az storage account show --name $STORAGE_ACCOUNT_NAME --query "id" -o tsv`
# </get_storage_account_id>

# <create_storage_container>
az storage container create --account-name $STORAGE_ACCOUNT_NAME --name $STORAGE_CONTAINER_NAME
# </create_storage_container>

# <upload_file_to_storage>
az storage blob upload --account-name $STORAGE_ACCOUNT_NAME --container-name $STORAGE_CONTAINER_NAME --name $FILE_NAME --file endpoints/online/managed/managed-identities/hello.txt
# </upload_file_to_storage>

# <get_user_identity_client_id>
uai_clientid=`az identity list --query "[?name=='$UAI_NAME'].clientId" -o tsv`
uai_principalid=`az identity list --query "[?name=='$UAI_NAME'].principalId" -o tsv`
# </get_user_identity_client_id>

# <get_user_identity_id>
uai_id=`az identity list --query "[?name=='$UAI_NAME'].id" -o tsv`
# </get_user_identity_id>

# <get_container_registry_id>
container_registry=`az ml workspace show --name $WORKSPACE --query container_registry -o tsv`
# </get_container_registry_id>

# <get_workspace_storage_id>
storage_account=`az ml workspace show --name $WORKSPACE --query storage_account -o tsv`
# </get_workspace_storage_id>

# <give_permission_to_user_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal --role "Storage Blob Data Reader" --scope $storage_id
# </give_permission_to_user_storage_account>

# <give_permission_to_container_registry>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "AcrPull" --scope $container_registry
# </give_permission_to_container_registry>

# <give_permission_to_workspace_storage_account>
az role assignment create --assignee-object-id $uai_principalid --assignee-principal-type ServicePrincipal  --role "Storage Blob Data Reader" --scope $storage_account
# </give_permission_to_workspace_storage_account>

# <create_endpoint>
az ml online-endpoint create --name $ENDPOINT_NAME -f endpoints/online/managed/managed-identities/1-uai-create-endpoint.yml --set identity.user_assigned_identities[0].resource_id=$uai_id
# </create_endpoint>

# <check_endpoint_Status>
az ml online-endpoint show --name $ENDPOINT_NAME
# </check_endpoint_Status>

# <deploy>
az ml online-deployment create --endpoint-name $ENDPOINT_NAME --all-traffic --name blue -f endpoints/online/managed/managed-identities/2-uai-deployment.yml --set environment_variables.STORAGE_ACCOUNT_NAME=$STORAGE_ACCOUNT_NAME environment_variables.STORAGE_CONTAINER_NAME=$STORAGE_CONTAINER_NAME environment_variables.FILE_NAME=$FILE_NAME environment_variables.UAI_CLIENT_ID=$uai_clientid
# </deploy>

# <check_deploy_Status>
az ml online-deployment show --endpoint-name $ENDPOINT_NAME -n blue
# </check_deploy_Status>

endpoint_status=`az ml online-deployment show --endpoint-name $ENDPOINT_NAME --name blue --query "provisioning_state" -o tsv`
echo $endpoint_status
if [[ $endpoint_status == "Succeeded" ]]
then
  echo "Endpoint created successfully"
else
  echo "Endpoint creation failed"
  exit 1
fi


# <check_deployment_log>
# Check deployment logs to confirm blob storage file contents read operation success.
az ml online-deployment get-logs --endpoint-name $ENDPOINT_NAME --name blue
# </check_deployment_log>

# <test_endpoint>
az ml online-endpoint invoke --name $ENDPOINT_NAME --request-file endpoints/online/model-1/sample-request.json
# </test_endpoint>

# <delete_endpoint>
az ml online-endpoint delete --name $ENDPOINT_NAME --yes
# </delete_endpoint>

# <delete_storage_account>
az storage account delete --name $STORAGE_ACCOUNT_NAME --yes
# </delete_storage_account>

# <delete_user_identity>
az identity delete --name $UAI_NAME
# </delete_user_identity>

```

# [System-assigned (Python)](#tab/system-identity-python)

Delete the endpoint: 

```python
ml_client.online_endpoints.begin_delete(endpoint_name)
```

Delete the storage account: 

```python
storage_client.storage_accounts.delete(
    resource_group_name=resource_group, account_name=storage_account_name
)
```

# [User-assigned (Python)](#tab/user-identity-python)

Delete the endpoint: 

```python
ml_client.online_endpoints.begin_delete(endpoint_name)
```


Delete the storage account: 

```python
storage_client.storage_accounts.delete(
    resource_group_name=resource_group, account_name=storage_account_name
)
```

Delete the User-assigned managed identity: 

```python
msi_client.user_assigned_identities.delete(
    resource_group_name=resource_group, resource_name=uai_name
)
```


## Next steps

* [Deploy and score a machine learning model by using an online endpoint](how-to-deploy-online-endpoints.md).
* For more on deployment, see [Safe rollout for online endpoints](how-to-safely-rollout-online-endpoints.md).
* For more information on using the CLI, see [Use the CLI extension for Azure Machine Learning](reference-azure-machine-learning-cli.md).
* To see which compute resources you can use, see [Managed online endpoints SKU list](reference-managed-online-endpoints-vm-sku-list.md).
* For more on costs, see [View costs for an Azure Machine Learning managed online endpoint](how-to-view-online-endpoints-costs.md).
* For information on monitoring endpoints, see [Monitor managed online endpoints](how-to-monitor-online-endpoints.md).
* For limitations for managed endpoints, see [Manage and increase quotas for resources with Azure Machine Learning](how-to-manage-quotas.md#azure-machine-learning-managed-online-endpoints).
