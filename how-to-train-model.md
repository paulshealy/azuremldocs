
# Train models with Azure Machine Learning CLI, SDK, and REST API

[!INCLUDE [dev v2](../../includes/machine-learning-dev-v2.md)]

Azure Machine Learning provides multiple ways to submit ML training jobs. In this article, you'll learn how to submit jobs using the following methods:

* Azure CLI extension for machine learning: The `ml` extension, also referred to as CLI v2.
* Python SDK v2 for Azure Machine Learning.
* REST API: The API that the CLI and SDK are built on.

## Prerequisites

* An Azure subscription. If you don't have an Azure subscription, create a free account before you begin. Try the [free or paid version of Azure Machine Learning](https://azure.microsoft.com/free/).
* An Azure Machine Learning workspace. If you don't have one, you can use the steps in the [Quickstart: Create Azure ML resources](quickstart-create-resources.md) article.

# [Python SDK](#tab/python)

To use the __SDK__ information, install the Azure Machine Learning [SDK v2 for Python](https://aka.ms/sdk-v2-install).

# [Azure CLI](#tab/azurecli)

To use the __CLI__ information, install the [Azure CLI and extension for machine learning](how-to-configure-cli.md).

# [REST API](#tab/restapi)

To use the __REST API__ information, you need the following items:

- A __service principal__ in your workspace. Administrative REST requests use [service principal authentication](how-to-setup-authentication.md#use-service-principal-authentication).
- A service principal __authentication token__. Follow the steps in [Retrieve a service principal authentication token](./how-to-manage-rest.md#retrieve-a-service-principal-authentication-token) to retrieve this token. 
- The __curl__ utility. The curl program is available in the [Windows Subsystem for Linux](/windows/wsl/install-win10) or any UNIX distribution. 

    > [!TIP]
    > In PowerShell, `curl` is an alias for `Invoke-WebRequest` and `curl -d "key=val" -X POST uri` becomes `Invoke-WebRequest -Body "key=val" -Method POST -Uri uri`.
    >
    > While it is possible to call the REST API from PowerShell, the examples in this article assume you are using Bash.

- The [jq](https://stedolan.github.io/jq/) utility for processing JSON. This utility is used to extract values from the JSON documents that are returned from REST API calls.


### Clone the examples repository

The code snippets in this article are based on examples in the [Azure ML examples GitHub repo](https://github.com/azure/azureml-examples). To clone the repository to your development environment, use the following command:

```bash
git clone --depth 1 https://github.com/Azure/azureml-examples
```

> [!TIP]
> Use `--depth 1` to clone only the latest commit to the repository, which reduces time to complete the operation.

## Example job

The examples in this article use the iris flower dataset to train an MLFlow model.

## Train in the cloud

When training in the cloud, you must connect to your Azure Machine Learning workspace and select a compute resource that will be used to run the training job.

### 1. Connect to the workspace

> [!TIP]
> Use the tabs below to select the method you want to use to train a model. Selecting a tab will automatically switch all the tabs in this article to the same tab. You can select another tab at any time.

# [Python SDK](#tab/python)

To connect to the workspace, you need identifier parameters - a subscription, resource group, and workspace name. You'll use these details in the `MLClient` from the `azure.ai.ml` namespace to get a handle to the required Azure Machine Learning workspace. To authenticate, you use the [default Azure authentication](/python/api/azure-identity/azure.identity.defaultazurecredential?view=azure-python&preserve-view=true). Check this [example](https://github.com/Azure/azureml-examples/blob/main/sdk/python/jobs/configuration.ipynb) for more details on how to configure credentials and connect to a workspace.

```python
#import required libraries
from azure.ai.ml import MLClient
from azure.identity import DefaultAzureCredential

#Enter details of your AzureML workspace
subscription_id = '<SUBSCRIPTION_ID>'
resource_group = '<RESOURCE_GROUP>'
workspace = '<AZUREML_WORKSPACE_NAME>'

#connect to the workspace
ml_client = MLClient(DefaultAzureCredential(), subscription_id, resource_group, workspace)
```

# [Azure CLI](#tab/azurecli)

When using the Azure CLI, you need identifier parameters - a subscription, resource group, and workspace name. While you can specify these parameters for each command, you can also set defaults that will be used for all the commands. Use the following commands to set default values. Replace `<subscription ID>`, `<AzureML workspace name>`, and `<resource group>` with the values for your configuration:

```azurecli
az account set --subscription <subscription ID>
az configure --defaults workspace=<AzureML workspace name> group=<resource group>
```

# [REST API](#tab/restapi)

The REST API examples in this article use `$SUBSCRIPTION_ID`, `$RESOURCE_GROUP`, `$LOCATION`, and `$WORKSPACE` placeholders. Replace the placeholders with your own values as follows:

* `$SUBSCRIPTION_ID`: Your Azure subscription ID.
* `$RESOURCE_GROUP`: The Azure resource group that contains your workspace.
* `$LOCATION`: The Azure region where your workspace is located.
* `$WORKSPACE`: The name of your Azure Machine Learning workspace.
* `$COMPUTE_NAME`: The name of your Azure Machine Learning compute cluster.

Administrative REST requests a [service principal authentication token](how-to-manage-rest.md#retrieve-a-service-principal-authentication-token). You can retrieve a token with the following command. The token is stored in the `$TOKEN` environment variable:

```azurecli
set -x

#<get_access_token>
TOKEN=$(az account get-access-token --query accessToken -o tsv)
#</get_access_token>

# <create_variables>
SUBSCRIPTION_ID=$(az account show --query id | tr -d '\r"')
LOCATION=$(az ml workspace show --query location | tr -d '\r"')
RESOURCE_GROUP=$(az group show --query name | tr -d '\r"')
WORKSPACE=$(az configure -l | jq -r '.[] | select(.name=="workspace") | .value')
#</create_variables>

# <set_endpoint_name>
export ENDPOINT_NAME=endpoint-`echo $RANDOM`
# </set_endpoint_name>

#<api_version>
API_VERSION="2022-05-01"
#</api_version>

echo -e "Using:\nSUBSCRIPTION_ID=$SUBSCRIPTION_ID\nLOCATION=$LOCATION\nRESOURCE_GROUP=$RESOURCE_GROUP\nWORKSPACE=$WORKSPACE"

# define how to wait  
wait_for_completion () {
    operation_id=$1
    status="unknown"

    if [[ $operation_id == "" || -z $operation_id  || $operation_id == "null" ]]; then
        echo "operation id cannot be empty"
        exit 1
    fi

    while [[ $status != "Succeeded" && $status != "Failed" ]]
    do
        echo "Getting operation status from: $operation_id"
        operation_result=$(curl --location --request GET $operation_id --header "Authorization: Bearer $TOKEN")
        # TODO error handling here
        status=$(echo $operation_result | jq -r '.status')
        echo "Current operation status: $status"
        sleep 5
    done

    if [[ $status == "Failed" ]]
    then
        error=$(echo $operation_result | jq -r '.error')
        echo "Error: $error"
    fi
}

# <get_storage_details>
# Get values for storage account
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores?api-version=$API_VERSION&isDefault=true" \
--header "Authorization: Bearer $TOKEN")
AZUREML_DEFAULT_DATASTORE=$(echo $response | jq -r '.value[0].name')
AZUREML_DEFAULT_CONTAINER=$(echo $response | jq -r '.value[0].properties.containerName')
export AZURE_STORAGE_ACCOUNT=$(echo $response | jq -r '.value[0].properties.accountName')
# </get_storage_details>

# <upload_code>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/score -s cli/endpoints/online/model-1/onlinescoring --account-name $AZURE_STORAGE_ACCOUNT
# </upload_code>

# <create_code>
az deployment group create -g $RESOURCE_GROUP \
--template-file arm-templates/code-version.json \
--parameters \
workspaceName=$WORKSPACE \
codeAssetName="score-sklearn" \
codeUri="https://$AZURE_STORAGE_ACCOUNT.blob.core.windows.net/$AZUREML_DEFAULT_CONTAINER/score"
# </create_code>

# <upload_model>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/model -s cli/endpoints/online/model-1/model --account-name $AZURE_STORAGE_ACCOUNT
# </upload_model>

# <create_model>
az deployment group create -g $RESOURCE_GROUP \
--template-file arm-templates/model-version.json \
--parameters \
workspaceName=$WORKSPACE \
modelAssetName="sklearn" \
modelUri="azureml://subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/workspaces/$WORKSPACE/datastores/$AZUREML_DEFAULT_DATASTORE/paths/model/sklearn_regression_model.pkl"
# </create_model>

# <read_condafile>
CONDA_FILE=$(cat cli/endpoints/online/model-1/environment/conda.yml)
# </read_condafile>

# <create_environment>
ENV_VERSION=$RANDOM
az deployment group create -g $RESOURCE_GROUP \
--template-file arm-templates/environment-version.json \
--parameters \
workspaceName=$WORKSPACE \
environmentAssetName=sklearn-env \
environmentAssetVersion=$ENV_VERSION \
dockerImage=mcr.microsoft.com/azureml/openmpi3.1.2-ubuntu18.04:20210727.v1 \
condaFile="$CONDA_FILE"
# </create_environment>

# <create_endpoint>
az deployment group create -g $RESOURCE_GROUP \
--template-file arm-templates/online-endpoint.json \
--parameters \
workspaceName=$WORKSPACE \
onlineEndpointName=$ENDPOINT_NAME \
identityType=SystemAssigned \
authMode=AMLToken \
location=$LOCATION
# </create_endpoint>

# <get_endpoint>
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/onlineEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN")

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id
# </get_endpoint>

# <create_deployment>
resourceScope="/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices"
az deployment group create -g $RESOURCE_GROUP \
 --template-file arm-templates/online-endpoint-deployment.json \
 --parameters \
 workspaceName=$WORKSPACE \
 location=$LOCATION \
 onlineEndpointName=$ENDPOINT_NAME \
 onlineDeploymentName=blue \
 codeId="$resourceScope/workspaces/$WORKSPACE/codes/score-sklearn/versions/1" \
 scoringScript=score.py \
 environmentId="$resourceScope/workspaces/$WORKSPACE/environments/sklearn-env/versions/$ENV_VERSION" \
 model="$resourceScope/workspaces/$WORKSPACE/models/score-sklearn/versions/1" \
 endpointComputeType=Managed \
 skuName=Standard_F2s_v2 \
 skuCapacity=1
 # </create_deployment>

# <get_deployment>
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/onlineEndpoints/$ENDPOINT_NAME/deployments/blue?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN")

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id

scoringUri=$(echo $response | jq -r '.properties' | jq -r '.scoringUri')
# </get_endpoint>

# <get_endpoint_access_token>
response=$(curl -H "Content-Length: 0" --location --request POST "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/onlineEndpoints/$ENDPOINT_NAME/token?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN")
accessToken=$(echo $response | jq -r '.accessToken')
# </get_endpoint_access_token>

# <score_endpoint>
curl --location --request POST $scoringUri \
--header "Authorization: Bearer $accessToken" \
--header "Content-Type: application/json" \
--data-raw @cli/endpoints/online/model-1/sample-request.json
# </score_endpoint>

# <get_deployment_logs>
curl --location --request POST "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/onlineEndpoints/$ENDPOINT_NAME/deployments/blue/getLogs?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{ \"tail\": 100 }"
# </get_deployment_logs>

# <delete_endpoint>
curl --location --request DELETE "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/onlineEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" || true
# </delete_endpoint>

```

The service provider uses the `api-version` argument to ensure compatibility. The `api-version` argument varies from service to service. Set the API version as a variable to accommodate future versions:

```rest-api
set -x

# <create_variables>
SUBSCRIPTION_ID=$(az account show --query id | tr -d '\r"')
LOCATION=$(az ml workspace show --query location | tr -d '\r"')
RESOURCE_GROUP=$(az group show --query name | tr -d '\r"')
WORKSPACE=$(az configure -l | jq -r '.[] | select(.name=="workspace") | .value')

#</create_variables>

#<get_access_token>
TOKEN=$(az account get-access-token --query accessToken -o tsv)
#</get_access_token>

# <set_endpoint_name>
export ENDPOINT_NAME=endpt-rest-`echo $RANDOM`
# </set_endpoint_name>

#<api_version>
API_VERSION="2022-05-01"
#</api_version>

echo "Using:\nSUBSCRIPTION_ID: $SUBSCRIPTION_ID\nLOCATION: $LOCATION\nRESOURCE_GROUP: $RESOURCE_GROUP\nWORKSPACE: $WORKSPACE\nENDPOINT_NAME: $ENDPOINT_NAME"

# define how to wait  
wait_for_completion () {
    operation_id=$1
    status="unknown"

    if [[ $operation_id == "" || -z $operation_id  || $operation_id == "null" ]]; then
        echo "operation id cannot be empty"
        exit 1
    fi

    while [[ $status != "Succeeded" && $status != "Failed" ]]
    do
        echo "Getting operation status from: $operation_id"
        operation_result=$(curl --location --request GET $operation_id --header "Authorization: Bearer $TOKEN")
        # TODO error handling here
        status=$(echo $operation_result | jq -r '.status')
        echo "Current operation status: $status"
        sleep 5
    done

    if [[ $status == "Failed" ]]
    then
        error=$(echo $operation_result | jq -r '.error')
        echo "Error: $error"
    fi
}

# <get_storage_details>
# Get values for storage account
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores?api-version=$API_VERSION&isDefault=true" \
--header "Authorization: Bearer $TOKEN")
AZUREML_DEFAULT_DATASTORE=$(echo $response | jq -r '.value[0].name')
AZUREML_DEFAULT_CONTAINER=$(echo $response | jq -r '.value[0].properties.containerName')
export AZURE_STORAGE_ACCOUNT=$(echo $response | jq -r '.value[0].properties.accountName')
# </get_storage_details>

# TODO: we can get the default container from listing datastores
# TODO using the latter two as env vars shouldn't be necessary

# <upload_code>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/score -s endpoints/online/model-1/onlinescoring
# </upload_code>

# <create_code>
curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-sklearn/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
  \"properties\": {
    \"codeUri\": \"https://$AZURE_STORAGE_ACCOUNT.blob.core.windows.net/$AZUREML_DEFAULT_CONTAINER/score\"
  }
}"
# </create_code>

# <upload_model>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/model -s endpoints/online/model-1/model
# </upload_model>

# <create_model>
curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/sklearn/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"modelUri\":\"azureml://subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/workspaces/$WORKSPACE/datastores/$AZUREML_DEFAULT_DATASTORE/paths/model\"
    }
}"
# </create_model>

# <read_condafile>
CONDA_FILE=$(cat endpoints/online/model-1/environment/conda.yml)
# <read_condafile>

# <create_environment>
ENV_VERSION=$RANDOM
curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/sklearn-env/versions/$ENV_VERSION?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"condaFile\": \"$CONDA_FILE\",
        \"image\": \"mcr.microsoft.com/azureml/openmpi3.1.2-ubuntu18.04:20210727.v1\"
    }
}"
# </create_environment>

#<create_endpoint>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/onlineEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"identity\": {
       \"type\": \"systemAssigned\"
    },
    \"properties\": {
        \"authMode\": \"AMLToken\"
    },
    \"location\": \"$LOCATION\"
}")
#</create_endpoint>

echo "Endpoint response: $response"
operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id

# <create_deployment>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/onlineEndpoints/$ENDPOINT_NAME/deployments/blue?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"location\": \"$LOCATION\",
    \"sku\": {
        \"capacity\": 1,
        \"name\": \"Standard_DS2_v2\"
    },
    \"properties\": {
        \"endpointComputeType\": \"Managed\",
        \"scaleSettings\": {
            \"scaleType\": \"Default\"
        },
        \"model\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/sklearn/versions/1\",
        \"codeConfiguration\": {
            \"codeId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-sklearn/versions/1\",
            \"scoringScript\": \"score.py\"
        },
        \"environmentId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/sklearn-env/versions/$ENV_VERSION\"
    }
}")
#</create_deployment>

echo "Endpoint response: $response"
operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id

# <get_endpoint>
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/onlineEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN")

scoringUri=$(echo $response | jq -r '.properties' | jq -r '.scoringUri')
# </get_endpoint>

# <get_access_token>
response=$(curl -H "Content-Length: 0" --location --request POST "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/onlineEndpoints/$ENDPOINT_NAME/token?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN")
accessToken=$(echo $response | jq -r '.accessToken')
# </get_access_token>

# <score_endpoint>
curl --location --request POST $scoringUri \
--header "Authorization: Bearer $accessToken" \
--header "Content-Type: application/json" \
--data-raw @endpoints/online/model-1/sample-request.json
# </score_endpoint>

# <get_deployment_logs>
curl --location --request POST "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/onlineEndpoints/$ENDPOINT_NAME/deployments/blue/getLogs?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{ \"tail\": 100 }"
#</get_deployment_logs>

# delete endpoint
# <delete_endpoint>
curl --location --request DELETE "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/onlineEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" || true
# </delete_endpoint>

```

When you train using the REST API, data and training scripts must be uploaded to a storage account that the workspace can access. The following example gets the storage information for your workspace and saves it into variables so we can use it later:

```azurecli
set -x

#<get_access_token>
TOKEN=$(az account get-access-token --query accessToken -o tsv)
#</get_access_token>

# <create_variables>
SUBSCRIPTION_ID=$(az account show --query id | tr -d '\r"')
LOCATION=$(az ml workspace show --query location | tr -d '\r"')
RESOURCE_GROUP=$(az group show --query name | tr -d '\r"')
WORKSPACE=$(az configure -l | jq -r '.[] | select(.name=="workspace") | .value')
#</create_variables>

# <set_endpoint_name>
export ENDPOINT_NAME=endpoint-`echo $RANDOM`
# </set_endpoint_name>

#<api_version>
API_VERSION="2022-05-01"
#</api_version>

echo -e "Using:\nSUBSCRIPTION_ID=$SUBSCRIPTION_ID\nLOCATION=$LOCATION\nRESOURCE_GROUP=$RESOURCE_GROUP\nWORKSPACE=$WORKSPACE"

# define how to wait  
wait_for_completion () {
    operation_id=$1
    status="unknown"

    if [[ $operation_id == "" || -z $operation_id  || $operation_id == "null" ]]; then
        echo "operation id cannot be empty"
        exit 1
    fi

    while [[ $status != "Succeeded" && $status != "Failed" ]]
    do
        echo "Getting operation status from: $operation_id"
        operation_result=$(curl --location --request GET $operation_id --header "Authorization: Bearer $TOKEN")
        # TODO error handling here
        status=$(echo $operation_result | jq -r '.status')
        echo "Current operation status: $status"
        sleep 5
    done

    if [[ $status == "Failed" ]]
    then
        error=$(echo $operation_result | jq -r '.error')
        echo "Error: $error"
    fi
}

# <get_storage_details>
# Get values for storage account
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores?api-version=$API_VERSION&isDefault=true" \
--header "Authorization: Bearer $TOKEN")
AZUREML_DEFAULT_DATASTORE=$(echo $response | jq -r '.value[0].name')
AZUREML_DEFAULT_CONTAINER=$(echo $response | jq -r '.value[0].properties.containerName')
export AZURE_STORAGE_ACCOUNT=$(echo $response | jq -r '.value[0].properties.accountName')
# </get_storage_details>

# <upload_code>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/score -s cli/endpoints/online/model-1/onlinescoring --account-name $AZURE_STORAGE_ACCOUNT
# </upload_code>

# <create_code>
az deployment group create -g $RESOURCE_GROUP \
--template-file arm-templates/code-version.json \
--parameters \
workspaceName=$WORKSPACE \
codeAssetName="score-sklearn" \
codeUri="https://$AZURE_STORAGE_ACCOUNT.blob.core.windows.net/$AZUREML_DEFAULT_CONTAINER/score"
# </create_code>

# <upload_model>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/model -s cli/endpoints/online/model-1/model --account-name $AZURE_STORAGE_ACCOUNT
# </upload_model>

# <create_model>
az deployment group create -g $RESOURCE_GROUP \
--template-file arm-templates/model-version.json \
--parameters \
workspaceName=$WORKSPACE \
modelAssetName="sklearn" \
modelUri="azureml://subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/workspaces/$WORKSPACE/datastores/$AZUREML_DEFAULT_DATASTORE/paths/model/sklearn_regression_model.pkl"
# </create_model>

# <read_condafile>
CONDA_FILE=$(cat cli/endpoints/online/model-1/environment/conda.yml)
# </read_condafile>

# <create_environment>
ENV_VERSION=$RANDOM
az deployment group create -g $RESOURCE_GROUP \
--template-file arm-templates/environment-version.json \
--parameters \
workspaceName=$WORKSPACE \
environmentAssetName=sklearn-env \
environmentAssetVersion=$ENV_VERSION \
dockerImage=mcr.microsoft.com/azureml/openmpi3.1.2-ubuntu18.04:20210727.v1 \
condaFile="$CONDA_FILE"
# </create_environment>

# <create_endpoint>
az deployment group create -g $RESOURCE_GROUP \
--template-file arm-templates/online-endpoint.json \
--parameters \
workspaceName=$WORKSPACE \
onlineEndpointName=$ENDPOINT_NAME \
identityType=SystemAssigned \
authMode=AMLToken \
location=$LOCATION
# </create_endpoint>

# <get_endpoint>
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/onlineEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN")

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id
# </get_endpoint>

# <create_deployment>
resourceScope="/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices"
az deployment group create -g $RESOURCE_GROUP \
 --template-file arm-templates/online-endpoint-deployment.json \
 --parameters \
 workspaceName=$WORKSPACE \
 location=$LOCATION \
 onlineEndpointName=$ENDPOINT_NAME \
 onlineDeploymentName=blue \
 codeId="$resourceScope/workspaces/$WORKSPACE/codes/score-sklearn/versions/1" \
 scoringScript=score.py \
 environmentId="$resourceScope/workspaces/$WORKSPACE/environments/sklearn-env/versions/$ENV_VERSION" \
 model="$resourceScope/workspaces/$WORKSPACE/models/score-sklearn/versions/1" \
 endpointComputeType=Managed \
 skuName=Standard_F2s_v2 \
 skuCapacity=1
 # </create_deployment>

# <get_deployment>
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/onlineEndpoints/$ENDPOINT_NAME/deployments/blue?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN")

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id

scoringUri=$(echo $response | jq -r '.properties' | jq -r '.scoringUri')
# </get_endpoint>

# <get_endpoint_access_token>
response=$(curl -H "Content-Length: 0" --location --request POST "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/onlineEndpoints/$ENDPOINT_NAME/token?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN")
accessToken=$(echo $response | jq -r '.accessToken')
# </get_endpoint_access_token>

# <score_endpoint>
curl --location --request POST $scoringUri \
--header "Authorization: Bearer $accessToken" \
--header "Content-Type: application/json" \
--data-raw @cli/endpoints/online/model-1/sample-request.json
# </score_endpoint>

# <get_deployment_logs>
curl --location --request POST "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/onlineEndpoints/$ENDPOINT_NAME/deployments/blue/getLogs?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{ \"tail\": 100 }"
# </get_deployment_logs>

# <delete_endpoint>
curl --location --request DELETE "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/onlineEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" || true
# </delete_endpoint>

```


### 2. Create a compute resource for training

An AzureML compute cluster is a fully managed compute resource that can be used to run the training job. In the following examples, a compute cluster named `cpu-compute` is created.

# [Python SDK](#tab/python)

```python
from azure.ai.ml.entities import AmlCompute

# specify aml compute name.
cpu_compute_target = "cpu-cluster"

try:
    ml_client.compute.get(cpu_compute_target)
except Exception:
    print("Creating a new cpu compute target...")
    compute = AmlCompute(
        name=cpu_compute_target, size="STANDARD_D2_V2", min_instances=0, max_instances=4
    )
    ml_client.compute.begin_create_or_update(compute).result()
```

# [Azure CLI](#tab/azurecli)

```azurecli
az ml compute create -n cpu-cluster --type amlcompute --min-instances 0 --max-instances 4
```

# [REST API](#tab/restapi)

```bash
curl -X PUT \
  "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/$COMPUTE_NAME?api-version=$API_VERSION" \
  -H "Authorization:Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "location": "'$LOCATION'",
    "properties": {
        "computeType": "AmlCompute",
        "properties": {
            "vmSize": "Standard_D2_V2",
            "vmPriority": "Dedicated",
            "scaleSettings": {
                "maxNodeCount": 4,
                "minNodeCount": 0,
                "nodeIdleTimeBeforeScaleDown": "PT30M"
            }
        }
    }
}'
```

> [!TIP]
> While a response is returned after a few seconds, this only indicates that the creation request has been accepted. It can take several minutes for the cluster creation to finish.


### 4. Submit the training job

# [Python SDK](#tab/python)

To run this script, you'll use a `command`. The command will be run by submitting it as a `job` to Azure ML. 

```python
from azure.ai.ml import command, Input

# define the command
command_job = command(
    code="./src",
    command="python main.py --iris-csv ${{inputs.iris_csv}} --learning-rate ${{inputs.learning_rate}} --boosting ${{inputs.boosting}}",
    environment="AzureML-lightgbm-3.2-ubuntu18.04-py37-cpu@latest",
    inputs={
        "iris_csv": Input(
            type="uri_file",
            path="https://azuremlexamples.blob.core.windows.net/datasets/iris.csv",
        ),
        "learning_rate": 0.9,
        "boosting": "gbdt",
    },
    compute="cpu-cluster",
)
```

```python
# submit the command
returned_job = ml_client.jobs.create_or_update(command_job)
# get a URL for the status of the job
returned_job.studio_url
```

In the above examples, you configured:
- `code` - path where the code to run the command is located
- `command` -  command that needs to be run
- `environment` - the environment needed to run the training script. In this example, we use a curated or ready-made environment provided by AzureML called `AzureML-lightgbm-3.2-ubuntu18.04-py37-cpu`. We use the latest version of this environment by using the `@latest` directive. You can also use custom environments by specifying a base docker image and specifying a conda yaml on top of it.
- `inputs` - dictionary of inputs using name value pairs to the command. The key is a name for the input within the context of the job and the value is the input value. Inputs are referenced in the `command` using the `${{inputs.<input_name>}}` expression. To use files or folders as inputs, you can use the `Input` class.

For more information, see the [reference documentation](/python/api/azure-ai-ml/azure.ai.ml#azure-ai-ml-command).

When you submit the job, a URL is returned to the job status in the AzureML studio. Use the studio UI to view the job progress. You can also use `returned_job.status` to check the current status of the job.

# [Azure CLI](#tab/azurecli)

The `az ml job create` command used in this example requires a YAML job definition file. The contents of the file used in this example are:

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
code: src
command: >-
  python main.py 
  --iris-csv ${{inputs.iris_csv}}
  --C ${{inputs.C}}
  --kernel ${{inputs.kernel}}
  --coef0 ${{inputs.coef0}}
inputs:
  iris_csv: 
    type: uri_file
    path: wasbs://datasets@azuremlexamples.blob.core.windows.net/iris.csv
  C: 0.8
  kernel: "rbf"
  coef0: 0.1
environment: azureml:AzureML-sklearn-0.24-ubuntu18.04-py37-cpu@latest
compute: azureml:cpu-cluster
display_name: sklearn-iris-example
experiment_name: sklearn-iris-example
description: Train a scikit-learn SVM on the Iris dataset.

```
In the above, you configured:
- `code` - path where the code to run the command is located
- `command` - command that needs to be run
- `inputs` - dictionary of inputs using name value pairs to the command. The key is a name for the input within the context of the job and the value is the input value. Inputs are referenced in the `command` using the `${{inputs.<input_name>}}` expression. 
- `environment` - the environment needed to run the training script. In this example, we use a curated or ready-made environment provided by AzureML called `AzureML-sklearn-0.24-ubuntu18.04-py37-cpu`. We use the latest version of this environment by using the `@latest` directive. You can also use custom environments by specifying a base docker image and specifying a conda yaml on top of it.
To submit the job, use the following command. The run ID (name) of the training job is stored in the `$run_id` variable:

```azurecli
run_id=$(az ml job create -f jobs/single-step/scikit-learn/iris/job.yml --query name -o tsv)
```

You can use the stored run ID to return information about the job. The `--web` parameter opens the AzureML studio web UI where you can drill into details on the job:

```azurecli
# <hello_world>
az ml job create -f jobs/basics/hello-world.yml --web
# </hello_world>

# <hello_world_set>
az ml job create -f jobs/basics/hello-world.yml \
  --set environment.image="python:3.8" \
  --web
# </hello_world_set>

# <hello_world_name>
run_id=$(az ml job create -f jobs/basics/hello-world.yml --query name -o tsv)
# </hello_world_name>

# <hello_world_show>
az ml job show -n $run_id --web
# </hello_world_show>

# <hello_world_org>
az ml job create -f jobs/basics/hello-world-org.yml --web
# </hello_world_org>

run_id=$(az ml job create -f jobs/basics/hello-world-org.yml --query name -o tsv)
# <hello_world_org_set>
az ml job update -n $run_id --set \
  display_name="updated display name" \
  experiment_name="updated experiment name" \
  description="updated description"  \
  tags.hello="updated tag"
# </hello_world_org_set>

# <hello_world_env_var>
az ml job create -f jobs/basics/hello-world-env-var.yml --web
# </hello_world_env_var>

# <hello_mlflow>
az ml job create -f jobs/basics/hello-mlflow.yml --web
# </hello_mlflow>

# <mlflow_uri>
az ml workspace show --query mlflow_tracking_uri -o tsv
# </mlflow_uri>

# <hello_world_input>
az ml job create -f jobs/basics/hello-world-input.yml --web
# </hello_world_input>

# <hello_world_input_set>
az ml job create -f jobs/basics/hello-world-input.yml --set \
  inputs.hello_string="hello there" \
  inputs.hello_number=24 \
  --web
# </hello_world_input_set>

# <hello_sweep>
az ml job create -f jobs/basics/hello-sweep.yml --web
# </hello_sweep>

# <hello_world_output>
az ml job create -f jobs/basics/hello-world-output.yml --web
# </hello_world_output>

run_id=$(az ml job create -f jobs/basics/hello-world-output.yml --query name -o tsv)
if [[ -z "$run_id" ]]
then
  echo "Job creation failed"
  exit 3
fi
status=$(az ml job show -n $run_id --query status -o tsv)
if [[ -z "$status" ]]
then
  echo "Status query failed"
  exit 4
fi
running=("Queued" "Starting" "Preparing" "Running" "Finalizing")
while [[ ${running[*]} =~ $status ]]
do
  sleep 8 
  status=$(az ml job show -n $run_id --query status -o tsv)
  echo $status
done

# <hello_world_output_download>
az ml job download -n $run_id
# </hello_world_output_download>
rm -r $run_id

# <iris_file>
az ml job create -f jobs/basics/hello-iris-file.yml --web
# </iris_file>

# <iris_folder>
az ml job create -f jobs/basics/hello-iris-folder.yml --web
# </iris_folder>

# <iris_datastore_file>
az ml job create -f jobs/basics/hello-iris-datastore-file.yml --web
# </iris_datastore_file>

# <iris_datastore_folder>
az ml job create -f jobs/basics/hello-iris-datastore-folder.yml --web
# </iris_datastore_folder>

# <hello_world_output_data>
az ml job create -f jobs/basics/hello-world-output-data.yml --web
# </hello_world_output_data>

# <hello_pipeline>
az ml job create -f jobs/basics/hello-pipeline.yml --web
# </hello_pipeline>

# <hello_pipeline_io>
az ml job create -f jobs/basics/hello-pipeline-io.yml --web
# </hello_pipeline_io>

# <hello_pipeline_settings>
az ml job create -f jobs/basics/hello-pipeline-settings.yml --web
# </hello_pipeline_settings>

# <hello_pipeline_abc>
az ml job create -f jobs/basics/hello-pipeline-abc.yml --web
# </hello_pipeline_abc>

# <sklearn_iris>
az ml job create -f jobs/single-step/scikit-learn/iris/job.yml --web
# </sklearn_iris>

run_id=$(az ml job create -f jobs/single-step/scikit-learn/iris/job.yml --query name -o tsv)
if [[ -z "$run_id" ]]
then
  echo "Job creation failed"
  exit 3
fi
status=$(az ml job show -n $run_id --query status -o tsv)
if [[ -z "$status" ]]
then
  echo "Status query failed"
  exit 4
fi
running=("Queued" "Starting" "Preparing" "Running" "Finalizing")
while [[ ${running[*]} =~ $status ]]
do
  sleep 8 
  status=$(az ml job show -n $run_id --query status -o tsv)
  echo $status
done

# <sklearn_download_register_model>
az ml model create -n sklearn-iris-example -v 1 -p runs:/$run_id/model --type mlflow_model
# </sklearn_download_register_model>
rm -r $run_id

# <sklearn_sweep>
az ml job create -f jobs/single-step/scikit-learn/iris/job-sweep.yml --web
# </sklearn_sweep>

# <pytorch_cifar>
az ml job create -f jobs/single-step/pytorch/cifar-distributed/job.yml --web
# </pytorch_cifar>

# # <pipeline_cifar>
# az ml job create -f jobs/pipelines/cifar-10/pipeline.yml --web
# # </pipeline_cifar>

```

# [REST API](#tab/restapi)

As part of job submission, the training scripts and data must be uploaded to a cloud storage location that your AzureML workspace can access. 

1. Use the following Azure CLI command to upload the training script. The command specifies the _directory_ that contains the files needed for training, not an individual file. If you'd like to use REST to upload the data instead, see the [Put Blob](/rest/api/storageservices/put-blob) reference:

    ```azurecli
    az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/testjob -s cli/jobs/single-step/scikit-learn/iris/src/ --account-name $AZURE_STORAGE_ACCOUNT
    ```

1. Create a versioned reference to the training data. In this example, the data is already in the cloud and located at `https://azuremlexamples.blob.core.windows.net/datasets/iris.csv`. For more information on referencing data, see [Data in Azure Machine Learning](concept-data.md):

    ```bash
    DATA_VERSION=$RANDOM
    curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/data/iris-data/versions/$DATA_VERSION?api-version=$API_VERSION" \
    --header "Authorization: Bearer $TOKEN" \
    --header "Content-Type: application/json" \
    --data-raw "{
            \"properties\": {
            \"description\": \"Iris dataset\",
            \"dataType\": \"uri_file\",
            \"dataUri\": \"https://azuremlexamples.blob.core.windows.net/datasets/iris.csv\"
        }
    }"
    ```

1. Register a versioned reference to the training script for use with a job. In this example, the script location is the default storage account and container you uploaded to in step 1. The ID of the versioned training code is returned and stored in the `$TRAIN_CODE` variable:

    ```bash
    TRAIN_CODE=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/train-lightgbm/versions/1?api-version=$API_VERSION" \
    --header "Authorization: Bearer $TOKEN" \
    --header "Content-Type: application/json" \
    --data-raw "{
            \"properties\": {
            \"description\": \"Train code\",
            \"codeUri\": \"https://$AZURE_STORAGE_ACCOUNT.blob.core.windows.net/$AZUREML_DEFAULT_CONTAINER/testjob\"
        }
    }" | jq -r '.id')
    ```

1. Create the environment that the cluster will use to run the training script. In this example, we use a curated or ready-made environment provided by AzureML called `AzureML-lightgbm-3.2-ubuntu18.04-py37-cpu`. The following command retrieves a list of the environment versions, with the newest being at the top of the collection. `jq` is used to retrieve the ID of the latest (`[0]`) version, which is then stored into the `$ENVIRONMENT` variable.

    ```bash
    ENVIRONMENT=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/AzureML-lightgbm-3.2-ubuntu18.04-py37-cpu/versions?api-version=$API_VERSION" --header "Authorization: Bearer $TOKEN" | jq -r .value[0].id)
    ```

1. Finally, submit the job. The following example shows how to submit the job, reference the training code ID, environment ID, URL for the input data, and the ID of the compute cluster. The job output location will be stored in the `$JOB_OUTPUT` variable:

    > [!TIP]
    > The job name must be unique. In this example, `uuidgen` is used to generate a unique value for the name.

    ```bash
    run_id=$(uuidgen)
    curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/jobs/$run_id?api-version=$API_VERSION" \
    --header "Authorization: Bearer $TOKEN" \
    --header "Content-Type: application/json" \
    --data-raw "{
        \"properties\": {
            \"jobType\": \"Command\",
            \"codeId\": \"$TRAIN_CODE\",
            \"command\": \"python main.py --iris-csv \$AZURE_ML_INPUT_iris\",
            \"environmentId\": \"$ENVIRONMENT\",
            \"inputs\": {
                \"iris\": {
                    \"jobInputType\": \"uri_file\",
                    \"uri\": \"https://azuremlexamples.blob.core.windows.net/datasets/iris.csv\"
                }
            },
            \"experimentName\": \"lightgbm-iris\",
            \"computeId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/$COMPUTE_NAME\"
        }
    }"
    ```



## Register the trained model

The following examples demonstrate how to register a model in your AzureML workspace.

# [Python SDK](#tab/python)

> [!TIP]
> The `name` property returned by the training job is used as part of the path to the model.

```python
from azure.ai.ml.entities import Model
from azure.ai.ml.constants import ModelType

run_model = Model(
    path="azureml://jobs/{}/outputs/artifacts/paths/model/".format(returned_job.name),
    name="run-model-example",
    description="Model created from run.",
    type=ModelType.MLFLOW
)

ml_client.models.create_or_update(run_model)
```

# [Azure CLI](#tab/azurecli)

> [!TIP]
> The name (stored in the `$run_id` variable) is used as part of the path to the model.

```azurecli
# <hello_world>
az ml job create -f jobs/basics/hello-world.yml --web
# </hello_world>

# <hello_world_set>
az ml job create -f jobs/basics/hello-world.yml \
  --set environment.image="python:3.8" \
  --web
# </hello_world_set>

# <hello_world_name>
run_id=$(az ml job create -f jobs/basics/hello-world.yml --query name -o tsv)
# </hello_world_name>

# <hello_world_show>
az ml job show -n $run_id --web
# </hello_world_show>

# <hello_world_org>
az ml job create -f jobs/basics/hello-world-org.yml --web
# </hello_world_org>

run_id=$(az ml job create -f jobs/basics/hello-world-org.yml --query name -o tsv)
# <hello_world_org_set>
az ml job update -n $run_id --set \
  display_name="updated display name" \
  experiment_name="updated experiment name" \
  description="updated description"  \
  tags.hello="updated tag"
# </hello_world_org_set>

# <hello_world_env_var>
az ml job create -f jobs/basics/hello-world-env-var.yml --web
# </hello_world_env_var>

# <hello_mlflow>
az ml job create -f jobs/basics/hello-mlflow.yml --web
# </hello_mlflow>

# <mlflow_uri>
az ml workspace show --query mlflow_tracking_uri -o tsv
# </mlflow_uri>

# <hello_world_input>
az ml job create -f jobs/basics/hello-world-input.yml --web
# </hello_world_input>

# <hello_world_input_set>
az ml job create -f jobs/basics/hello-world-input.yml --set \
  inputs.hello_string="hello there" \
  inputs.hello_number=24 \
  --web
# </hello_world_input_set>

# <hello_sweep>
az ml job create -f jobs/basics/hello-sweep.yml --web
# </hello_sweep>

# <hello_world_output>
az ml job create -f jobs/basics/hello-world-output.yml --web
# </hello_world_output>

run_id=$(az ml job create -f jobs/basics/hello-world-output.yml --query name -o tsv)
if [[ -z "$run_id" ]]
then
  echo "Job creation failed"
  exit 3
fi
status=$(az ml job show -n $run_id --query status -o tsv)
if [[ -z "$status" ]]
then
  echo "Status query failed"
  exit 4
fi
running=("Queued" "Starting" "Preparing" "Running" "Finalizing")
while [[ ${running[*]} =~ $status ]]
do
  sleep 8 
  status=$(az ml job show -n $run_id --query status -o tsv)
  echo $status
done

# <hello_world_output_download>
az ml job download -n $run_id
# </hello_world_output_download>
rm -r $run_id

# <iris_file>
az ml job create -f jobs/basics/hello-iris-file.yml --web
# </iris_file>

# <iris_folder>
az ml job create -f jobs/basics/hello-iris-folder.yml --web
# </iris_folder>

# <iris_datastore_file>
az ml job create -f jobs/basics/hello-iris-datastore-file.yml --web
# </iris_datastore_file>

# <iris_datastore_folder>
az ml job create -f jobs/basics/hello-iris-datastore-folder.yml --web
# </iris_datastore_folder>

# <hello_world_output_data>
az ml job create -f jobs/basics/hello-world-output-data.yml --web
# </hello_world_output_data>

# <hello_pipeline>
az ml job create -f jobs/basics/hello-pipeline.yml --web
# </hello_pipeline>

# <hello_pipeline_io>
az ml job create -f jobs/basics/hello-pipeline-io.yml --web
# </hello_pipeline_io>

# <hello_pipeline_settings>
az ml job create -f jobs/basics/hello-pipeline-settings.yml --web
# </hello_pipeline_settings>

# <hello_pipeline_abc>
az ml job create -f jobs/basics/hello-pipeline-abc.yml --web
# </hello_pipeline_abc>

# <sklearn_iris>
az ml job create -f jobs/single-step/scikit-learn/iris/job.yml --web
# </sklearn_iris>

run_id=$(az ml job create -f jobs/single-step/scikit-learn/iris/job.yml --query name -o tsv)
if [[ -z "$run_id" ]]
then
  echo "Job creation failed"
  exit 3
fi
status=$(az ml job show -n $run_id --query status -o tsv)
if [[ -z "$status" ]]
then
  echo "Status query failed"
  exit 4
fi
running=("Queued" "Starting" "Preparing" "Running" "Finalizing")
while [[ ${running[*]} =~ $status ]]
do
  sleep 8 
  status=$(az ml job show -n $run_id --query status -o tsv)
  echo $status
done

# <sklearn_download_register_model>
az ml model create -n sklearn-iris-example -v 1 -p runs:/$run_id/model --type mlflow_model
# </sklearn_download_register_model>
rm -r $run_id

# <sklearn_sweep>
az ml job create -f jobs/single-step/scikit-learn/iris/job-sweep.yml --web
# </sklearn_sweep>

# <pytorch_cifar>
az ml job create -f jobs/single-step/pytorch/cifar-distributed/job.yml --web
# </pytorch_cifar>

# # <pipeline_cifar>
# az ml job create -f jobs/pipelines/cifar-10/pipeline.yml --web
# # </pipeline_cifar>

```

# [REST API](#tab/restapi)

> [!TIP]
> The name (stored in the `$run_id` variable) is used as part of the path to the model.

```bash
curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/sklearn/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"modelType\": \"mlflow_model\",
        \"modelUri\":\"runs:/$run_id/model\"
    }
}"
```


## Next steps

Now that you have a trained model, learn [how to deploy it using an online endpoint](how-to-deploy-online-endpoints.md).

For more examples, see the [AzureML examples](https://github.com/azure/azureml-examples) GitHub repository.

For more information on the Azure CLI commands, Python SDK classes, or REST APIs used in this article, see the following reference documentation:

* [Azure CLI `ml` extension](/cli/azure/ml)
* [Python SDK](/python/api/azure-ai-ml/azure.ai.ml)
* [REST API](/rest/api/azureml/)
