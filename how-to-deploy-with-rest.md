
# Deploy models with REST

Learn how to use the Azure Machine Learning REST API to deploy models.

The REST API uses standard HTTP verbs to create, retrieve, update, and delete resources. The REST API works with any language or tool that can make HTTP requests. REST's straightforward structure makes it a good choice in scripting environments and for MLOps automation.

In this article, you learn how to use the new REST APIs to:

> [!div class="checklist"]
> * Create machine learning assets
> * Create a basic training job 
> * Create a hyperparameter tuning sweep job

## Prerequisites

- An **Azure subscription** for which you have administrative rights. If you don't have such a subscription, try the [free or paid personal subscription](https://azure.microsoft.com/free/).
- An [Azure Machine Learning workspace](quickstart-create-resources.md).
- A service principal in your workspace. Administrative REST requests use [service principal authentication](how-to-setup-authentication.md#use-service-principal-authentication).
- A service principal authentication token. Follow the steps in [Retrieve a service principal authentication token](./how-to-manage-rest.md#retrieve-a-service-principal-authentication-token) to retrieve this token. 
- The **curl** utility. The **curl** program is available in the [Windows Subsystem for Linux](/windows/wsl/install-win10) or any UNIX distribution. In PowerShell, **curl** is an alias for **Invoke-WebRequest** and `curl -d "key=val" -X POST uri` becomes `Invoke-WebRequest -Body "key=val" -Method POST -Uri uri`. 

## Set endpoint name

> [!NOTE]
> Endpoint names need to be unique at the Azure region level. For example, there can be only one endpoint with the name my-endpoint in westus2.

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

## Azure Machine Learning online endpoints

Online endpoints allow you to deploy your model without having to create and manage the underlying infrastructure as well as Kubernetes clusters. In this article, you'll create an online endpoint and deployment, and validate it by invoking it. But first you'll have to register the assets needed for deployment, including model, code, and environment.

There are many ways to create an Azure Machine Learning online endpoint [including the Azure CLI](how-to-deploy-online-endpoints.md), and visually with [the studio](how-to-use-managed-online-endpoint-studio.md). The following example an online endpoint with the REST API.

## Create machine learning assets

First, set up your Azure Machine Learning assets to configure your job.

In the following REST API calls, we use `SUBSCRIPTION_ID`, `RESOURCE_GROUP`, `LOCATION`, and `WORKSPACE` as placeholders. Replace the placeholders with your own values. 

Administrative REST requests a [service principal authentication token](how-to-manage-rest.md#retrieve-a-service-principal-authentication-token). Replace `TOKEN` with your own value. You can retrieve this token with the following command:

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

### Get storage account details

To register the model and code, first they need to be uploaded to a storage account. The details of the storage account are available in the data store. In this example, you get the default datastore and Azure Storage account for your workspace. Query your workspace with a GET request to get a JSON file with the information.

You can use the tool [jq](https://stedolan.github.io/jq/) to parse the JSON result and get the required values. You can also use the Azure portal to find the same information:

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

### Upload & register code

Now that you have the datastore, you can upload the scoring script. Use the Azure Storage CLI to upload a blob into your default container:

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

> [!TIP]
> You can also use other methods to upload, such as the Azure portal or [Azure Storage Explorer](https://azure.microsoft.com/features/storage-explorer/).

Once you upload your code, you can specify your code with a PUT request and refer to the datastore with `datastoreId`:

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

### Upload and register model

Similar to the code, Upload the model files:

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

Now, register the model:

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

### Create environment
The deployment needs to run in an environment that has the required dependencies. Create the environment with a PUT request. Use a docker image from Microsoft Container Registry. You can configure the docker image with `Docker` and add conda dependencies with `condaFile`.

In the following snippet, the contents of a Conda environment (YAML file) has been read into an environment variable:

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

### Create endpoint

Create the online endpoint:

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

### Create deployment

Create a deployment under the endpoint:

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

### Invoke the endpoint to score data with your model

We need the scoring uri and access token to invoke the endpoint. First get the scoring uri:

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

Get the endpoint access token:

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

Now, invoke the endpoint using curl:

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

### Check the logs

Check the deployment logs:

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

### Delete the endpoint

If you aren't going use the deployment, you should delete it with the below command (it deletes the endpoint and all the underlying deployments):

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

## Next steps

* Learn how to deploy your model [using the Azure CLI](how-to-deploy-online-endpoints.md).
* Learn how to deploy your model [using studio](how-to-use-managed-online-endpoint-studio.md).
* Learn to [Troubleshoot online endpoints deployment and scoring](how-to-troubleshoot-managed-online-endpoints.md)
* Learn how to [Access Azure resources with a online endpoint and managed identity](how-to-access-resources-from-endpoints-managed-identities.md)
* Learn how to [monitor online endpoints](how-to-monitor-online-endpoints.md).
* Learn [safe rollout for online endpoints](how-to-safely-rollout-online-endpoints.md).
* [View costs for an Azure Machine Learning managed online endpoint](how-to-view-online-endpoints-costs.md).
* [Managed online endpoints SKU list](reference-managed-online-endpoints-vm-sku-list.md).
* Learn about limits on managed online endpoints in [Manage and increase quotas for resources with Azure Machine Learning](how-to-manage-quotas.md#azure-machine-learning-managed-online-endpoints).
