
# Deploy models with REST for batch scoring 

[!INCLUDE [cli v2](../../includes/machine-learning-cli-v2.md)]

Learn how to use the Azure Machine Learning REST API to deploy models for batch scoring.



The REST API uses standard HTTP verbs to create, retrieve, update, and delete resources. The REST API works with any language or tool that can make HTTP requests. REST's straightforward structure makes it a good choice in scripting environments and for MLOps automation.

In this article, you learn how to use the new REST APIs to:

> [!div class="checklist"]
> * Create machine learning assets
> * Create a batch endpoint and a batch deployment
> * Invoke a batch endpoint to start a batch scoring job

## Prerequisites

- An **Azure subscription** for which you have administrative rights. If you don't have such a subscription, try the [free or paid personal subscription](https://azure.microsoft.com/free/).
- An [Azure Machine Learning workspace](quickstart-create-resources.md).
- A service principal in your workspace. Administrative REST requests use [service principal authentication](how-to-setup-authentication.md#use-service-principal-authentication).
- A service principal authentication token. Follow the steps in [Retrieve a service principal authentication token](./how-to-manage-rest.md#retrieve-a-service-principal-authentication-token) to retrieve this token. 
- The **curl** utility. The **curl** program is available in the [Windows Subsystem for Linux](/windows/wsl/install-win10) or any UNIX distribution. In PowerShell, **curl** is an alias for **Invoke-WebRequest** and `curl -d "key=val" -X POST uri` becomes `Invoke-WebRequest -Body "key=val" -Method POST -Uri uri`. 
- The [jq](https://stedolan.github.io/jq/) JSON processor.

> [!IMPORTANT]
> The code snippets in this article assume that you are using the Bash shell.
>
> The code snippets are pulled from the `/cli/batch-score-rest.sh` file in the [AzureML Example repository](https://github.com/Azure/azureml-examples).

## Set endpoint name

> [!NOTE]
> Batch endpoint names need to be unique at the Azure region level. For example, there can be only one batch endpoint with the name mybatchendpoint in westus2.

```rest-api
# <create_variables>
SUBSCRIPTION_ID=$(az account show --query id | tr -d '\r"')
RESOURCE_GROUP=$(az group show --query name | tr -d '\r"')
WORKSPACE=$(az configure -l | jq -r '.[] | select(.name=="workspace") | .value')

LOCATION=$(az ml workspace show| jq -r '.location')

API_VERSION="2022-05-01"

TOKEN=$(az account get-access-token --query accessToken -o tsv)
#</create_variables>

# <set_endpoint_name>
export ENDPOINT_NAME=endpt-`echo $RANDOM`
# </set_endpoint_name>

echo "Using: SUBSCRIPTION_ID: $SUBSCRIPTION_ID LOCATION: $LOCATION RESOURCE_GROUP: $RESOURCE_GROUP WORKSPACE: $WORKSPACE ENDPOINT_NAME: $ENDPOINT_NAME"

# <define_wait>  
wait_for_completion () {
    operation_id=$1
    access_token=$2
    status="unknown"

    while [[ $status != "Completed" && $status != "Succeeded" && $status != "Failed" && $status != "Canceled" ]]
    do
        echo "Getting operation status from: $operation_id"
        operation_result=$(curl --location --request GET $operation_id --header "Authorization: Bearer $access_token")
        # TODO error handling here
        status=$(echo $operation_result | jq -r '.status')
        if [[ -z $status || $status == "null" ]]
        then
            status=$(echo $operation_result | jq -r '.properties.status')
        fi

        # Fail early if job submission failed and there is nothing to poll on
        if [[ -z $status || $status == "null" ]]
        then
            echo "No status found on operation, setting to failed."
            status="Failed"
        fi

        echo "Current operation status: $status"
        sleep 10
    done

    if [[ $status == "Failed" ]]
    then
        error=$(echo $operation_result | jq -r '.error')
        echo "Error: $error"
    fi
}
# </define_wait>  

# <get_storage_details>
# Get values for storage account
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores?api-version=$API_VERSION&isDefault=true" \
--header "Authorization: Bearer $TOKEN")
DATASTORE_PATH=$(echo $response | jq -r '.value[0].id')
BLOB_URI_PROTOCOL=$(echo $response | jq -r '.value[0].properties.protocol')
BLOB_URI_ENDPOINT=$(echo $response | jq -r '.value[0].properties.endpoint')
AZUREML_DEFAULT_DATASTORE=$(echo $response | jq -r '.value[0].name')
AZUREML_DEFAULT_CONTAINER=$(echo $response | jq -r '.value[0].properties.containerName')
AZURE_STORAGE_ACCOUNT=$(echo $response | jq -r '.value[0].properties.accountName')
export AZURE_STORAGE_ACCOUNT $(echo $AZURE_STORAGE_ACCOUNT)
STORAGE_RESPONSE=$(echo az storage account show-connection-string --name $AZURE_STORAGE_ACCOUNT)
AZURE_STORAGE_CONNECTION_STRING=$($STORAGE_RESPONSE | jq -r '.connectionString')

BLOB_URI_ROOT="$BLOB_URI_PROTOCOL://$AZURE_STORAGE_ACCOUNT.blob.$BLOB_URI_ENDPOINT/$AZUREML_DEFAULT_CONTAINER"
# </get_storage_details>

# <upload_code>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/score -s endpoints/batch/mnist/code/  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_code>

# <create_code>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
  \"properties\": {
    \"description\": \"Score code\",
    \"codeUri\": \"$BLOB_URI_ROOT/score\"
  }
}")
# </create_code>

# <upload_model>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/model -s endpoints/batch/mnist/model  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_model>

# <create_model>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"modelUri\":\"azureml://subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/workspaces/$WORKSPACE/datastores/$AZUREML_DEFAULT_DATASTORE/paths/model\"
    }
}")
# </create_model>

# <read_condafile>
CONDA_FILE=$(cat endpoints/batch/mnist/environment/conda.json | sed 's/"/\\"/g')
# </read_condafile>
# <create_environment>
ENV_VERSION=$RANDOM
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"condaFile\": $(echo \"$CONDA_FILE\"),
        \"image\": \"mcr.microsoft.com/azureml/openmpi3.1.2-ubuntu18.04:latest\"
    }
}")
# </create_environment>

# <create_compute>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"computeType\": \"AmlCompute\",
        \"properties\": {
            \"osType\": \"Linux\",
            \"vmSize\": \"STANDARD_D2_V2\",
            \"scaleSettings\": {
                \"maxNodeCount\": 5,
                \"minNodeCount\": 0
            },
            \"remoteLoginPortPublicAccess\": \"NotSpecified\"
        },
    },
    \"location\": \"$LOCATION\"
}")
# </create_compute>

#<create_endpoint>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\"
    },
    \"location\": \"$LOCATION\"
}")
#</create_endpoint>

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

# <create_deployment>
DEPLOYMENT_NAME="nonmlflowedp"
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME/deployments/$DEPLOYMENT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"location\": \"$LOCATION\",
    \"properties\": {        
        \"model\": {
            \"referenceType\": \"Id\",
            \"assetId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1\"
        },
        \"codeConfiguration\": {
            \"codeId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1\",
            \"scoringScript\": \"digit_identification.py\"
        },
        \"environmentId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION\",
        \"compute\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster\",
        \"resources\": {
            \"instanceCount\": 1
        },
        \"maxConcurrencyPerInstance\": \"4\",
        \"retrySettings\": {
            \"maxRetries\": 3,
            \"timeout\": \"PT30S\"
        },
        \"errorThreshold\": \"10\",
        \"loggingLevel\": \"info\",
        \"miniBatchSize\": \"5\",
    }
}")
#</create_deployment>
operation_id=$(echo $response | jq -r '.properties.properties.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

#<set_endpoint_defaults>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\",
        \"defaults\": {
            \"deploymentName\": \"$DEPLOYMENT_NAME\"
        }
    },
    \"location\": \"$LOCATION\"
}")

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN
#</set_endpoint_defaults>

# <get_endpoint>
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN")

# <score_endpoint>
SCORING_URI=$(echo $response | jq -r '.properties.scoringUri')
# </get_endpoint>

# <get_access_token>
SCORING_TOKEN=$(az account get-access-token --resource https://ml.azure.com --query accessToken -o tsv)
# </get_access_token>

# <score_endpoint_with_data_in_cloud>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DataUrl\",
            \"Path\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
        }
    }
}")

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# </score_endpoint_with_data_in_cloud>

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>

#<create_dataset>
DATASET_NAME="mnist"
DATASET_VERSION=$RANDOM

response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datasets/$DATASET_NAME/versions/$DATASET_VERSION?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"paths\": [
            {
                \"folder\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
            }
        ]
    }
}")
#</create_dataset>

#<unique_output>
export OUTPUT_FILE_NAME=predictions_`echo $RANDOM`.csv
#</unique_output>

# <score_endpoint_with_dataset>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DatasetVersion\",
            \"datasetName\": \"$DATASET_NAME\",
            \"datasetVersion\": \"$DATASET_VERSION\"
        },
        \"outputDataset\": {
            \"datastoreId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores/workspaceblobstore\",
            \"path\": \"$ENDPOINT_NAME\"
        },
        \"outputFileName\": \"$OUTPUT_FILE_NAME\"
    }
}")
# </score_endpoint_with_dataset>

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>
# </score_endpoint>

# <delete_endpoint>
curl --location --request DELETE "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" || true
# </delete_endpoint>

```

## Azure Machine Learning batch endpoints

[Batch endpoints](concept-endpoints.md#what-are-batch-endpoints) simplify the process of hosting your models for batch scoring, so you can focus on machine learning, not infrastructure. In this article, you'll create a batch endpoint and deployment, and invoking it to start a batch scoring job. But first you'll have to register the assets needed for deployment, including model, code, and environment.

There are many ways to create an Azure Machine Learning batch endpoint, including the Azure CLI, Azure ML SDK for Python, and visually with the studio. The following example creates a batch endpoint and a batch deployment with the REST API.

## Create machine learning assets

First, set up your Azure Machine Learning assets to configure your job.

In the following REST API calls, we use `SUBSCRIPTION_ID`, `RESOURCE_GROUP`, `LOCATION`, and `WORKSPACE` as placeholders. Replace the placeholders with your own values. 

Administrative REST requests a [service principal authentication token](how-to-manage-rest.md#retrieve-a-service-principal-authentication-token). Replace `TOKEN` with your own value. You can retrieve this token with the following command:

```rest-api
# <create_variables>
SUBSCRIPTION_ID=$(az account show --query id | tr -d '\r"')
RESOURCE_GROUP=$(az group show --query name | tr -d '\r"')
WORKSPACE=$(az configure -l | jq -r '.[] | select(.name=="workspace") | .value')

LOCATION=$(az ml workspace show| jq -r '.location')

API_VERSION="2022-05-01"

TOKEN=$(az account get-access-token --query accessToken -o tsv)
#</create_variables>

# <set_endpoint_name>
export ENDPOINT_NAME=endpt-`echo $RANDOM`
# </set_endpoint_name>

echo "Using: SUBSCRIPTION_ID: $SUBSCRIPTION_ID LOCATION: $LOCATION RESOURCE_GROUP: $RESOURCE_GROUP WORKSPACE: $WORKSPACE ENDPOINT_NAME: $ENDPOINT_NAME"

# <define_wait>  
wait_for_completion () {
    operation_id=$1
    access_token=$2
    status="unknown"

    while [[ $status != "Completed" && $status != "Succeeded" && $status != "Failed" && $status != "Canceled" ]]
    do
        echo "Getting operation status from: $operation_id"
        operation_result=$(curl --location --request GET $operation_id --header "Authorization: Bearer $access_token")
        # TODO error handling here
        status=$(echo $operation_result | jq -r '.status')
        if [[ -z $status || $status == "null" ]]
        then
            status=$(echo $operation_result | jq -r '.properties.status')
        fi

        # Fail early if job submission failed and there is nothing to poll on
        if [[ -z $status || $status == "null" ]]
        then
            echo "No status found on operation, setting to failed."
            status="Failed"
        fi

        echo "Current operation status: $status"
        sleep 10
    done

    if [[ $status == "Failed" ]]
    then
        error=$(echo $operation_result | jq -r '.error')
        echo "Error: $error"
    fi
}
# </define_wait>  

# <get_storage_details>
# Get values for storage account
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores?api-version=$API_VERSION&isDefault=true" \
--header "Authorization: Bearer $TOKEN")
DATASTORE_PATH=$(echo $response | jq -r '.value[0].id')
BLOB_URI_PROTOCOL=$(echo $response | jq -r '.value[0].properties.protocol')
BLOB_URI_ENDPOINT=$(echo $response | jq -r '.value[0].properties.endpoint')
AZUREML_DEFAULT_DATASTORE=$(echo $response | jq -r '.value[0].name')
AZUREML_DEFAULT_CONTAINER=$(echo $response | jq -r '.value[0].properties.containerName')
AZURE_STORAGE_ACCOUNT=$(echo $response | jq -r '.value[0].properties.accountName')
export AZURE_STORAGE_ACCOUNT $(echo $AZURE_STORAGE_ACCOUNT)
STORAGE_RESPONSE=$(echo az storage account show-connection-string --name $AZURE_STORAGE_ACCOUNT)
AZURE_STORAGE_CONNECTION_STRING=$($STORAGE_RESPONSE | jq -r '.connectionString')

BLOB_URI_ROOT="$BLOB_URI_PROTOCOL://$AZURE_STORAGE_ACCOUNT.blob.$BLOB_URI_ENDPOINT/$AZUREML_DEFAULT_CONTAINER"
# </get_storage_details>

# <upload_code>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/score -s endpoints/batch/mnist/code/  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_code>

# <create_code>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
  \"properties\": {
    \"description\": \"Score code\",
    \"codeUri\": \"$BLOB_URI_ROOT/score\"
  }
}")
# </create_code>

# <upload_model>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/model -s endpoints/batch/mnist/model  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_model>

# <create_model>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"modelUri\":\"azureml://subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/workspaces/$WORKSPACE/datastores/$AZUREML_DEFAULT_DATASTORE/paths/model\"
    }
}")
# </create_model>

# <read_condafile>
CONDA_FILE=$(cat endpoints/batch/mnist/environment/conda.json | sed 's/"/\\"/g')
# </read_condafile>
# <create_environment>
ENV_VERSION=$RANDOM
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"condaFile\": $(echo \"$CONDA_FILE\"),
        \"image\": \"mcr.microsoft.com/azureml/openmpi3.1.2-ubuntu18.04:latest\"
    }
}")
# </create_environment>

# <create_compute>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"computeType\": \"AmlCompute\",
        \"properties\": {
            \"osType\": \"Linux\",
            \"vmSize\": \"STANDARD_D2_V2\",
            \"scaleSettings\": {
                \"maxNodeCount\": 5,
                \"minNodeCount\": 0
            },
            \"remoteLoginPortPublicAccess\": \"NotSpecified\"
        },
    },
    \"location\": \"$LOCATION\"
}")
# </create_compute>

#<create_endpoint>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\"
    },
    \"location\": \"$LOCATION\"
}")
#</create_endpoint>

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

# <create_deployment>
DEPLOYMENT_NAME="nonmlflowedp"
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME/deployments/$DEPLOYMENT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"location\": \"$LOCATION\",
    \"properties\": {        
        \"model\": {
            \"referenceType\": \"Id\",
            \"assetId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1\"
        },
        \"codeConfiguration\": {
            \"codeId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1\",
            \"scoringScript\": \"digit_identification.py\"
        },
        \"environmentId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION\",
        \"compute\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster\",
        \"resources\": {
            \"instanceCount\": 1
        },
        \"maxConcurrencyPerInstance\": \"4\",
        \"retrySettings\": {
            \"maxRetries\": 3,
            \"timeout\": \"PT30S\"
        },
        \"errorThreshold\": \"10\",
        \"loggingLevel\": \"info\",
        \"miniBatchSize\": \"5\",
    }
}")
#</create_deployment>
operation_id=$(echo $response | jq -r '.properties.properties.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

#<set_endpoint_defaults>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\",
        \"defaults\": {
            \"deploymentName\": \"$DEPLOYMENT_NAME\"
        }
    },
    \"location\": \"$LOCATION\"
}")

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN
#</set_endpoint_defaults>

# <get_endpoint>
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN")

# <score_endpoint>
SCORING_URI=$(echo $response | jq -r '.properties.scoringUri')
# </get_endpoint>

# <get_access_token>
SCORING_TOKEN=$(az account get-access-token --resource https://ml.azure.com --query accessToken -o tsv)
# </get_access_token>

# <score_endpoint_with_data_in_cloud>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DataUrl\",
            \"Path\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
        }
    }
}")

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# </score_endpoint_with_data_in_cloud>

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>

#<create_dataset>
DATASET_NAME="mnist"
DATASET_VERSION=$RANDOM

response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datasets/$DATASET_NAME/versions/$DATASET_VERSION?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"paths\": [
            {
                \"folder\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
            }
        ]
    }
}")
#</create_dataset>

#<unique_output>
export OUTPUT_FILE_NAME=predictions_`echo $RANDOM`.csv
#</unique_output>

# <score_endpoint_with_dataset>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DatasetVersion\",
            \"datasetName\": \"$DATASET_NAME\",
            \"datasetVersion\": \"$DATASET_VERSION\"
        },
        \"outputDataset\": {
            \"datastoreId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores/workspaceblobstore\",
            \"path\": \"$ENDPOINT_NAME\"
        },
        \"outputFileName\": \"$OUTPUT_FILE_NAME\"
    }
}")
# </score_endpoint_with_dataset>

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>
# </score_endpoint>

# <delete_endpoint>
curl --location --request DELETE "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" || true
# </delete_endpoint>

```

The service provider uses the `api-version` argument to ensure compatibility. The `api-version` argument varies from service to service. Set the API version as a variable to accommodate future versions:

```rest-api
# <create_variables>
SUBSCRIPTION_ID=$(az account show --query id | tr -d '\r"')
RESOURCE_GROUP=$(az group show --query name | tr -d '\r"')
WORKSPACE=$(az configure -l | jq -r '.[] | select(.name=="workspace") | .value')

LOCATION=$(az ml workspace show| jq -r '.location')

API_VERSION="2022-05-01"

TOKEN=$(az account get-access-token --query accessToken -o tsv)
#</create_variables>

# <set_endpoint_name>
export ENDPOINT_NAME=endpt-`echo $RANDOM`
# </set_endpoint_name>

echo "Using: SUBSCRIPTION_ID: $SUBSCRIPTION_ID LOCATION: $LOCATION RESOURCE_GROUP: $RESOURCE_GROUP WORKSPACE: $WORKSPACE ENDPOINT_NAME: $ENDPOINT_NAME"

# <define_wait>  
wait_for_completion () {
    operation_id=$1
    access_token=$2
    status="unknown"

    while [[ $status != "Completed" && $status != "Succeeded" && $status != "Failed" && $status != "Canceled" ]]
    do
        echo "Getting operation status from: $operation_id"
        operation_result=$(curl --location --request GET $operation_id --header "Authorization: Bearer $access_token")
        # TODO error handling here
        status=$(echo $operation_result | jq -r '.status')
        if [[ -z $status || $status == "null" ]]
        then
            status=$(echo $operation_result | jq -r '.properties.status')
        fi

        # Fail early if job submission failed and there is nothing to poll on
        if [[ -z $status || $status == "null" ]]
        then
            echo "No status found on operation, setting to failed."
            status="Failed"
        fi

        echo "Current operation status: $status"
        sleep 10
    done

    if [[ $status == "Failed" ]]
    then
        error=$(echo $operation_result | jq -r '.error')
        echo "Error: $error"
    fi
}
# </define_wait>  

# <get_storage_details>
# Get values for storage account
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores?api-version=$API_VERSION&isDefault=true" \
--header "Authorization: Bearer $TOKEN")
DATASTORE_PATH=$(echo $response | jq -r '.value[0].id')
BLOB_URI_PROTOCOL=$(echo $response | jq -r '.value[0].properties.protocol')
BLOB_URI_ENDPOINT=$(echo $response | jq -r '.value[0].properties.endpoint')
AZUREML_DEFAULT_DATASTORE=$(echo $response | jq -r '.value[0].name')
AZUREML_DEFAULT_CONTAINER=$(echo $response | jq -r '.value[0].properties.containerName')
AZURE_STORAGE_ACCOUNT=$(echo $response | jq -r '.value[0].properties.accountName')
export AZURE_STORAGE_ACCOUNT $(echo $AZURE_STORAGE_ACCOUNT)
STORAGE_RESPONSE=$(echo az storage account show-connection-string --name $AZURE_STORAGE_ACCOUNT)
AZURE_STORAGE_CONNECTION_STRING=$($STORAGE_RESPONSE | jq -r '.connectionString')

BLOB_URI_ROOT="$BLOB_URI_PROTOCOL://$AZURE_STORAGE_ACCOUNT.blob.$BLOB_URI_ENDPOINT/$AZUREML_DEFAULT_CONTAINER"
# </get_storage_details>

# <upload_code>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/score -s endpoints/batch/mnist/code/  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_code>

# <create_code>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
  \"properties\": {
    \"description\": \"Score code\",
    \"codeUri\": \"$BLOB_URI_ROOT/score\"
  }
}")
# </create_code>

# <upload_model>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/model -s endpoints/batch/mnist/model  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_model>

# <create_model>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"modelUri\":\"azureml://subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/workspaces/$WORKSPACE/datastores/$AZUREML_DEFAULT_DATASTORE/paths/model\"
    }
}")
# </create_model>

# <read_condafile>
CONDA_FILE=$(cat endpoints/batch/mnist/environment/conda.json | sed 's/"/\\"/g')
# </read_condafile>
# <create_environment>
ENV_VERSION=$RANDOM
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"condaFile\": $(echo \"$CONDA_FILE\"),
        \"image\": \"mcr.microsoft.com/azureml/openmpi3.1.2-ubuntu18.04:latest\"
    }
}")
# </create_environment>

# <create_compute>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"computeType\": \"AmlCompute\",
        \"properties\": {
            \"osType\": \"Linux\",
            \"vmSize\": \"STANDARD_D2_V2\",
            \"scaleSettings\": {
                \"maxNodeCount\": 5,
                \"minNodeCount\": 0
            },
            \"remoteLoginPortPublicAccess\": \"NotSpecified\"
        },
    },
    \"location\": \"$LOCATION\"
}")
# </create_compute>

#<create_endpoint>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\"
    },
    \"location\": \"$LOCATION\"
}")
#</create_endpoint>

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

# <create_deployment>
DEPLOYMENT_NAME="nonmlflowedp"
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME/deployments/$DEPLOYMENT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"location\": \"$LOCATION\",
    \"properties\": {        
        \"model\": {
            \"referenceType\": \"Id\",
            \"assetId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1\"
        },
        \"codeConfiguration\": {
            \"codeId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1\",
            \"scoringScript\": \"digit_identification.py\"
        },
        \"environmentId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION\",
        \"compute\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster\",
        \"resources\": {
            \"instanceCount\": 1
        },
        \"maxConcurrencyPerInstance\": \"4\",
        \"retrySettings\": {
            \"maxRetries\": 3,
            \"timeout\": \"PT30S\"
        },
        \"errorThreshold\": \"10\",
        \"loggingLevel\": \"info\",
        \"miniBatchSize\": \"5\",
    }
}")
#</create_deployment>
operation_id=$(echo $response | jq -r '.properties.properties.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

#<set_endpoint_defaults>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\",
        \"defaults\": {
            \"deploymentName\": \"$DEPLOYMENT_NAME\"
        }
    },
    \"location\": \"$LOCATION\"
}")

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN
#</set_endpoint_defaults>

# <get_endpoint>
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN")

# <score_endpoint>
SCORING_URI=$(echo $response | jq -r '.properties.scoringUri')
# </get_endpoint>

# <get_access_token>
SCORING_TOKEN=$(az account get-access-token --resource https://ml.azure.com --query accessToken -o tsv)
# </get_access_token>

# <score_endpoint_with_data_in_cloud>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DataUrl\",
            \"Path\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
        }
    }
}")

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# </score_endpoint_with_data_in_cloud>

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>

#<create_dataset>
DATASET_NAME="mnist"
DATASET_VERSION=$RANDOM

response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datasets/$DATASET_NAME/versions/$DATASET_VERSION?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"paths\": [
            {
                \"folder\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
            }
        ]
    }
}")
#</create_dataset>

#<unique_output>
export OUTPUT_FILE_NAME=predictions_`echo $RANDOM`.csv
#</unique_output>

# <score_endpoint_with_dataset>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DatasetVersion\",
            \"datasetName\": \"$DATASET_NAME\",
            \"datasetVersion\": \"$DATASET_VERSION\"
        },
        \"outputDataset\": {
            \"datastoreId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores/workspaceblobstore\",
            \"path\": \"$ENDPOINT_NAME\"
        },
        \"outputFileName\": \"$OUTPUT_FILE_NAME\"
    }
}")
# </score_endpoint_with_dataset>

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>
# </score_endpoint>

# <delete_endpoint>
curl --location --request DELETE "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" || true
# </delete_endpoint>

```

### Create compute
Batch scoring runs only on cloud computing resources, not locally. The cloud computing resource is a reusable virtual computer cluster where you can run batch scoring workflows.

Create a compute cluster:

```rest-api
# <create_variables>
SUBSCRIPTION_ID=$(az account show --query id | tr -d '\r"')
RESOURCE_GROUP=$(az group show --query name | tr -d '\r"')
WORKSPACE=$(az configure -l | jq -r '.[] | select(.name=="workspace") | .value')

LOCATION=$(az ml workspace show| jq -r '.location')

API_VERSION="2022-05-01"

TOKEN=$(az account get-access-token --query accessToken -o tsv)
#</create_variables>

# <set_endpoint_name>
export ENDPOINT_NAME=endpt-`echo $RANDOM`
# </set_endpoint_name>

echo "Using: SUBSCRIPTION_ID: $SUBSCRIPTION_ID LOCATION: $LOCATION RESOURCE_GROUP: $RESOURCE_GROUP WORKSPACE: $WORKSPACE ENDPOINT_NAME: $ENDPOINT_NAME"

# <define_wait>  
wait_for_completion () {
    operation_id=$1
    access_token=$2
    status="unknown"

    while [[ $status != "Completed" && $status != "Succeeded" && $status != "Failed" && $status != "Canceled" ]]
    do
        echo "Getting operation status from: $operation_id"
        operation_result=$(curl --location --request GET $operation_id --header "Authorization: Bearer $access_token")
        # TODO error handling here
        status=$(echo $operation_result | jq -r '.status')
        if [[ -z $status || $status == "null" ]]
        then
            status=$(echo $operation_result | jq -r '.properties.status')
        fi

        # Fail early if job submission failed and there is nothing to poll on
        if [[ -z $status || $status == "null" ]]
        then
            echo "No status found on operation, setting to failed."
            status="Failed"
        fi

        echo "Current operation status: $status"
        sleep 10
    done

    if [[ $status == "Failed" ]]
    then
        error=$(echo $operation_result | jq -r '.error')
        echo "Error: $error"
    fi
}
# </define_wait>  

# <get_storage_details>
# Get values for storage account
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores?api-version=$API_VERSION&isDefault=true" \
--header "Authorization: Bearer $TOKEN")
DATASTORE_PATH=$(echo $response | jq -r '.value[0].id')
BLOB_URI_PROTOCOL=$(echo $response | jq -r '.value[0].properties.protocol')
BLOB_URI_ENDPOINT=$(echo $response | jq -r '.value[0].properties.endpoint')
AZUREML_DEFAULT_DATASTORE=$(echo $response | jq -r '.value[0].name')
AZUREML_DEFAULT_CONTAINER=$(echo $response | jq -r '.value[0].properties.containerName')
AZURE_STORAGE_ACCOUNT=$(echo $response | jq -r '.value[0].properties.accountName')
export AZURE_STORAGE_ACCOUNT $(echo $AZURE_STORAGE_ACCOUNT)
STORAGE_RESPONSE=$(echo az storage account show-connection-string --name $AZURE_STORAGE_ACCOUNT)
AZURE_STORAGE_CONNECTION_STRING=$($STORAGE_RESPONSE | jq -r '.connectionString')

BLOB_URI_ROOT="$BLOB_URI_PROTOCOL://$AZURE_STORAGE_ACCOUNT.blob.$BLOB_URI_ENDPOINT/$AZUREML_DEFAULT_CONTAINER"
# </get_storage_details>

# <upload_code>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/score -s endpoints/batch/mnist/code/  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_code>

# <create_code>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
  \"properties\": {
    \"description\": \"Score code\",
    \"codeUri\": \"$BLOB_URI_ROOT/score\"
  }
}")
# </create_code>

# <upload_model>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/model -s endpoints/batch/mnist/model  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_model>

# <create_model>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"modelUri\":\"azureml://subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/workspaces/$WORKSPACE/datastores/$AZUREML_DEFAULT_DATASTORE/paths/model\"
    }
}")
# </create_model>

# <read_condafile>
CONDA_FILE=$(cat endpoints/batch/mnist/environment/conda.json | sed 's/"/\\"/g')
# </read_condafile>
# <create_environment>
ENV_VERSION=$RANDOM
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"condaFile\": $(echo \"$CONDA_FILE\"),
        \"image\": \"mcr.microsoft.com/azureml/openmpi3.1.2-ubuntu18.04:latest\"
    }
}")
# </create_environment>

# <create_compute>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"computeType\": \"AmlCompute\",
        \"properties\": {
            \"osType\": \"Linux\",
            \"vmSize\": \"STANDARD_D2_V2\",
            \"scaleSettings\": {
                \"maxNodeCount\": 5,
                \"minNodeCount\": 0
            },
            \"remoteLoginPortPublicAccess\": \"NotSpecified\"
        },
    },
    \"location\": \"$LOCATION\"
}")
# </create_compute>

#<create_endpoint>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\"
    },
    \"location\": \"$LOCATION\"
}")
#</create_endpoint>

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

# <create_deployment>
DEPLOYMENT_NAME="nonmlflowedp"
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME/deployments/$DEPLOYMENT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"location\": \"$LOCATION\",
    \"properties\": {        
        \"model\": {
            \"referenceType\": \"Id\",
            \"assetId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1\"
        },
        \"codeConfiguration\": {
            \"codeId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1\",
            \"scoringScript\": \"digit_identification.py\"
        },
        \"environmentId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION\",
        \"compute\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster\",
        \"resources\": {
            \"instanceCount\": 1
        },
        \"maxConcurrencyPerInstance\": \"4\",
        \"retrySettings\": {
            \"maxRetries\": 3,
            \"timeout\": \"PT30S\"
        },
        \"errorThreshold\": \"10\",
        \"loggingLevel\": \"info\",
        \"miniBatchSize\": \"5\",
    }
}")
#</create_deployment>
operation_id=$(echo $response | jq -r '.properties.properties.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

#<set_endpoint_defaults>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\",
        \"defaults\": {
            \"deploymentName\": \"$DEPLOYMENT_NAME\"
        }
    },
    \"location\": \"$LOCATION\"
}")

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN
#</set_endpoint_defaults>

# <get_endpoint>
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN")

# <score_endpoint>
SCORING_URI=$(echo $response | jq -r '.properties.scoringUri')
# </get_endpoint>

# <get_access_token>
SCORING_TOKEN=$(az account get-access-token --resource https://ml.azure.com --query accessToken -o tsv)
# </get_access_token>

# <score_endpoint_with_data_in_cloud>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DataUrl\",
            \"Path\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
        }
    }
}")

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# </score_endpoint_with_data_in_cloud>

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>

#<create_dataset>
DATASET_NAME="mnist"
DATASET_VERSION=$RANDOM

response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datasets/$DATASET_NAME/versions/$DATASET_VERSION?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"paths\": [
            {
                \"folder\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
            }
        ]
    }
}")
#</create_dataset>

#<unique_output>
export OUTPUT_FILE_NAME=predictions_`echo $RANDOM`.csv
#</unique_output>

# <score_endpoint_with_dataset>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DatasetVersion\",
            \"datasetName\": \"$DATASET_NAME\",
            \"datasetVersion\": \"$DATASET_VERSION\"
        },
        \"outputDataset\": {
            \"datastoreId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores/workspaceblobstore\",
            \"path\": \"$ENDPOINT_NAME\"
        },
        \"outputFileName\": \"$OUTPUT_FILE_NAME\"
    }
}")
# </score_endpoint_with_dataset>

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>
# </score_endpoint>

# <delete_endpoint>
curl --location --request DELETE "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" || true
# </delete_endpoint>

```

> [!TIP]
> If you want to use an existing compute instead, you must specify the full Azure Resource Manager ID when [creating the batch deployment](#create-batch-deployment). The full ID uses the format `/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/<your-compute-name>`.

### Get storage account details

To register the model and code, first they need to be uploaded to a storage account. The details of the storage account are available in the data store. In this example, you get the default datastore and Azure Storage account for your workspace. Query your workspace with a GET request to get a JSON file with the information.

You can use the tool [jq](https://stedolan.github.io/jq/) to parse the JSON result and get the required values. You can also use the Azure portal to find the same information:

```rest-api
# <create_variables>
SUBSCRIPTION_ID=$(az account show --query id | tr -d '\r"')
RESOURCE_GROUP=$(az group show --query name | tr -d '\r"')
WORKSPACE=$(az configure -l | jq -r '.[] | select(.name=="workspace") | .value')

LOCATION=$(az ml workspace show| jq -r '.location')

API_VERSION="2022-05-01"

TOKEN=$(az account get-access-token --query accessToken -o tsv)
#</create_variables>

# <set_endpoint_name>
export ENDPOINT_NAME=endpt-`echo $RANDOM`
# </set_endpoint_name>

echo "Using: SUBSCRIPTION_ID: $SUBSCRIPTION_ID LOCATION: $LOCATION RESOURCE_GROUP: $RESOURCE_GROUP WORKSPACE: $WORKSPACE ENDPOINT_NAME: $ENDPOINT_NAME"

# <define_wait>  
wait_for_completion () {
    operation_id=$1
    access_token=$2
    status="unknown"

    while [[ $status != "Completed" && $status != "Succeeded" && $status != "Failed" && $status != "Canceled" ]]
    do
        echo "Getting operation status from: $operation_id"
        operation_result=$(curl --location --request GET $operation_id --header "Authorization: Bearer $access_token")
        # TODO error handling here
        status=$(echo $operation_result | jq -r '.status')
        if [[ -z $status || $status == "null" ]]
        then
            status=$(echo $operation_result | jq -r '.properties.status')
        fi

        # Fail early if job submission failed and there is nothing to poll on
        if [[ -z $status || $status == "null" ]]
        then
            echo "No status found on operation, setting to failed."
            status="Failed"
        fi

        echo "Current operation status: $status"
        sleep 10
    done

    if [[ $status == "Failed" ]]
    then
        error=$(echo $operation_result | jq -r '.error')
        echo "Error: $error"
    fi
}
# </define_wait>  

# <get_storage_details>
# Get values for storage account
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores?api-version=$API_VERSION&isDefault=true" \
--header "Authorization: Bearer $TOKEN")
DATASTORE_PATH=$(echo $response | jq -r '.value[0].id')
BLOB_URI_PROTOCOL=$(echo $response | jq -r '.value[0].properties.protocol')
BLOB_URI_ENDPOINT=$(echo $response | jq -r '.value[0].properties.endpoint')
AZUREML_DEFAULT_DATASTORE=$(echo $response | jq -r '.value[0].name')
AZUREML_DEFAULT_CONTAINER=$(echo $response | jq -r '.value[0].properties.containerName')
AZURE_STORAGE_ACCOUNT=$(echo $response | jq -r '.value[0].properties.accountName')
export AZURE_STORAGE_ACCOUNT $(echo $AZURE_STORAGE_ACCOUNT)
STORAGE_RESPONSE=$(echo az storage account show-connection-string --name $AZURE_STORAGE_ACCOUNT)
AZURE_STORAGE_CONNECTION_STRING=$($STORAGE_RESPONSE | jq -r '.connectionString')

BLOB_URI_ROOT="$BLOB_URI_PROTOCOL://$AZURE_STORAGE_ACCOUNT.blob.$BLOB_URI_ENDPOINT/$AZUREML_DEFAULT_CONTAINER"
# </get_storage_details>

# <upload_code>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/score -s endpoints/batch/mnist/code/  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_code>

# <create_code>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
  \"properties\": {
    \"description\": \"Score code\",
    \"codeUri\": \"$BLOB_URI_ROOT/score\"
  }
}")
# </create_code>

# <upload_model>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/model -s endpoints/batch/mnist/model  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_model>

# <create_model>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"modelUri\":\"azureml://subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/workspaces/$WORKSPACE/datastores/$AZUREML_DEFAULT_DATASTORE/paths/model\"
    }
}")
# </create_model>

# <read_condafile>
CONDA_FILE=$(cat endpoints/batch/mnist/environment/conda.json | sed 's/"/\\"/g')
# </read_condafile>
# <create_environment>
ENV_VERSION=$RANDOM
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"condaFile\": $(echo \"$CONDA_FILE\"),
        \"image\": \"mcr.microsoft.com/azureml/openmpi3.1.2-ubuntu18.04:latest\"
    }
}")
# </create_environment>

# <create_compute>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"computeType\": \"AmlCompute\",
        \"properties\": {
            \"osType\": \"Linux\",
            \"vmSize\": \"STANDARD_D2_V2\",
            \"scaleSettings\": {
                \"maxNodeCount\": 5,
                \"minNodeCount\": 0
            },
            \"remoteLoginPortPublicAccess\": \"NotSpecified\"
        },
    },
    \"location\": \"$LOCATION\"
}")
# </create_compute>

#<create_endpoint>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\"
    },
    \"location\": \"$LOCATION\"
}")
#</create_endpoint>

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

# <create_deployment>
DEPLOYMENT_NAME="nonmlflowedp"
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME/deployments/$DEPLOYMENT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"location\": \"$LOCATION\",
    \"properties\": {        
        \"model\": {
            \"referenceType\": \"Id\",
            \"assetId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1\"
        },
        \"codeConfiguration\": {
            \"codeId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1\",
            \"scoringScript\": \"digit_identification.py\"
        },
        \"environmentId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION\",
        \"compute\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster\",
        \"resources\": {
            \"instanceCount\": 1
        },
        \"maxConcurrencyPerInstance\": \"4\",
        \"retrySettings\": {
            \"maxRetries\": 3,
            \"timeout\": \"PT30S\"
        },
        \"errorThreshold\": \"10\",
        \"loggingLevel\": \"info\",
        \"miniBatchSize\": \"5\",
    }
}")
#</create_deployment>
operation_id=$(echo $response | jq -r '.properties.properties.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

#<set_endpoint_defaults>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\",
        \"defaults\": {
            \"deploymentName\": \"$DEPLOYMENT_NAME\"
        }
    },
    \"location\": \"$LOCATION\"
}")

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN
#</set_endpoint_defaults>

# <get_endpoint>
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN")

# <score_endpoint>
SCORING_URI=$(echo $response | jq -r '.properties.scoringUri')
# </get_endpoint>

# <get_access_token>
SCORING_TOKEN=$(az account get-access-token --resource https://ml.azure.com --query accessToken -o tsv)
# </get_access_token>

# <score_endpoint_with_data_in_cloud>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DataUrl\",
            \"Path\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
        }
    }
}")

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# </score_endpoint_with_data_in_cloud>

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>

#<create_dataset>
DATASET_NAME="mnist"
DATASET_VERSION=$RANDOM

response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datasets/$DATASET_NAME/versions/$DATASET_VERSION?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"paths\": [
            {
                \"folder\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
            }
        ]
    }
}")
#</create_dataset>

#<unique_output>
export OUTPUT_FILE_NAME=predictions_`echo $RANDOM`.csv
#</unique_output>

# <score_endpoint_with_dataset>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DatasetVersion\",
            \"datasetName\": \"$DATASET_NAME\",
            \"datasetVersion\": \"$DATASET_VERSION\"
        },
        \"outputDataset\": {
            \"datastoreId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores/workspaceblobstore\",
            \"path\": \"$ENDPOINT_NAME\"
        },
        \"outputFileName\": \"$OUTPUT_FILE_NAME\"
    }
}")
# </score_endpoint_with_dataset>

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>
# </score_endpoint>

# <delete_endpoint>
curl --location --request DELETE "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" || true
# </delete_endpoint>

```

### Upload & register code

Now that you have the datastore, you can upload the scoring script. For more information about how to author the scoring script, see [Understanding the scoring script](batch-inference/how-to-batch-scoring-script.md#understanding-the-scoring-script). Use the Azure Storage CLI to upload a blob into your default container:

```rest-api
# <create_variables>
SUBSCRIPTION_ID=$(az account show --query id | tr -d '\r"')
RESOURCE_GROUP=$(az group show --query name | tr -d '\r"')
WORKSPACE=$(az configure -l | jq -r '.[] | select(.name=="workspace") | .value')

LOCATION=$(az ml workspace show| jq -r '.location')

API_VERSION="2022-05-01"

TOKEN=$(az account get-access-token --query accessToken -o tsv)
#</create_variables>

# <set_endpoint_name>
export ENDPOINT_NAME=endpt-`echo $RANDOM`
# </set_endpoint_name>

echo "Using: SUBSCRIPTION_ID: $SUBSCRIPTION_ID LOCATION: $LOCATION RESOURCE_GROUP: $RESOURCE_GROUP WORKSPACE: $WORKSPACE ENDPOINT_NAME: $ENDPOINT_NAME"

# <define_wait>  
wait_for_completion () {
    operation_id=$1
    access_token=$2
    status="unknown"

    while [[ $status != "Completed" && $status != "Succeeded" && $status != "Failed" && $status != "Canceled" ]]
    do
        echo "Getting operation status from: $operation_id"
        operation_result=$(curl --location --request GET $operation_id --header "Authorization: Bearer $access_token")
        # TODO error handling here
        status=$(echo $operation_result | jq -r '.status')
        if [[ -z $status || $status == "null" ]]
        then
            status=$(echo $operation_result | jq -r '.properties.status')
        fi

        # Fail early if job submission failed and there is nothing to poll on
        if [[ -z $status || $status == "null" ]]
        then
            echo "No status found on operation, setting to failed."
            status="Failed"
        fi

        echo "Current operation status: $status"
        sleep 10
    done

    if [[ $status == "Failed" ]]
    then
        error=$(echo $operation_result | jq -r '.error')
        echo "Error: $error"
    fi
}
# </define_wait>  

# <get_storage_details>
# Get values for storage account
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores?api-version=$API_VERSION&isDefault=true" \
--header "Authorization: Bearer $TOKEN")
DATASTORE_PATH=$(echo $response | jq -r '.value[0].id')
BLOB_URI_PROTOCOL=$(echo $response | jq -r '.value[0].properties.protocol')
BLOB_URI_ENDPOINT=$(echo $response | jq -r '.value[0].properties.endpoint')
AZUREML_DEFAULT_DATASTORE=$(echo $response | jq -r '.value[0].name')
AZUREML_DEFAULT_CONTAINER=$(echo $response | jq -r '.value[0].properties.containerName')
AZURE_STORAGE_ACCOUNT=$(echo $response | jq -r '.value[0].properties.accountName')
export AZURE_STORAGE_ACCOUNT $(echo $AZURE_STORAGE_ACCOUNT)
STORAGE_RESPONSE=$(echo az storage account show-connection-string --name $AZURE_STORAGE_ACCOUNT)
AZURE_STORAGE_CONNECTION_STRING=$($STORAGE_RESPONSE | jq -r '.connectionString')

BLOB_URI_ROOT="$BLOB_URI_PROTOCOL://$AZURE_STORAGE_ACCOUNT.blob.$BLOB_URI_ENDPOINT/$AZUREML_DEFAULT_CONTAINER"
# </get_storage_details>

# <upload_code>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/score -s endpoints/batch/mnist/code/  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_code>

# <create_code>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
  \"properties\": {
    \"description\": \"Score code\",
    \"codeUri\": \"$BLOB_URI_ROOT/score\"
  }
}")
# </create_code>

# <upload_model>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/model -s endpoints/batch/mnist/model  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_model>

# <create_model>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"modelUri\":\"azureml://subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/workspaces/$WORKSPACE/datastores/$AZUREML_DEFAULT_DATASTORE/paths/model\"
    }
}")
# </create_model>

# <read_condafile>
CONDA_FILE=$(cat endpoints/batch/mnist/environment/conda.json | sed 's/"/\\"/g')
# </read_condafile>
# <create_environment>
ENV_VERSION=$RANDOM
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"condaFile\": $(echo \"$CONDA_FILE\"),
        \"image\": \"mcr.microsoft.com/azureml/openmpi3.1.2-ubuntu18.04:latest\"
    }
}")
# </create_environment>

# <create_compute>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"computeType\": \"AmlCompute\",
        \"properties\": {
            \"osType\": \"Linux\",
            \"vmSize\": \"STANDARD_D2_V2\",
            \"scaleSettings\": {
                \"maxNodeCount\": 5,
                \"minNodeCount\": 0
            },
            \"remoteLoginPortPublicAccess\": \"NotSpecified\"
        },
    },
    \"location\": \"$LOCATION\"
}")
# </create_compute>

#<create_endpoint>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\"
    },
    \"location\": \"$LOCATION\"
}")
#</create_endpoint>

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

# <create_deployment>
DEPLOYMENT_NAME="nonmlflowedp"
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME/deployments/$DEPLOYMENT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"location\": \"$LOCATION\",
    \"properties\": {        
        \"model\": {
            \"referenceType\": \"Id\",
            \"assetId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1\"
        },
        \"codeConfiguration\": {
            \"codeId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1\",
            \"scoringScript\": \"digit_identification.py\"
        },
        \"environmentId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION\",
        \"compute\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster\",
        \"resources\": {
            \"instanceCount\": 1
        },
        \"maxConcurrencyPerInstance\": \"4\",
        \"retrySettings\": {
            \"maxRetries\": 3,
            \"timeout\": \"PT30S\"
        },
        \"errorThreshold\": \"10\",
        \"loggingLevel\": \"info\",
        \"miniBatchSize\": \"5\",
    }
}")
#</create_deployment>
operation_id=$(echo $response | jq -r '.properties.properties.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

#<set_endpoint_defaults>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\",
        \"defaults\": {
            \"deploymentName\": \"$DEPLOYMENT_NAME\"
        }
    },
    \"location\": \"$LOCATION\"
}")

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN
#</set_endpoint_defaults>

# <get_endpoint>
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN")

# <score_endpoint>
SCORING_URI=$(echo $response | jq -r '.properties.scoringUri')
# </get_endpoint>

# <get_access_token>
SCORING_TOKEN=$(az account get-access-token --resource https://ml.azure.com --query accessToken -o tsv)
# </get_access_token>

# <score_endpoint_with_data_in_cloud>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DataUrl\",
            \"Path\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
        }
    }
}")

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# </score_endpoint_with_data_in_cloud>

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>

#<create_dataset>
DATASET_NAME="mnist"
DATASET_VERSION=$RANDOM

response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datasets/$DATASET_NAME/versions/$DATASET_VERSION?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"paths\": [
            {
                \"folder\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
            }
        ]
    }
}")
#</create_dataset>

#<unique_output>
export OUTPUT_FILE_NAME=predictions_`echo $RANDOM`.csv
#</unique_output>

# <score_endpoint_with_dataset>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DatasetVersion\",
            \"datasetName\": \"$DATASET_NAME\",
            \"datasetVersion\": \"$DATASET_VERSION\"
        },
        \"outputDataset\": {
            \"datastoreId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores/workspaceblobstore\",
            \"path\": \"$ENDPOINT_NAME\"
        },
        \"outputFileName\": \"$OUTPUT_FILE_NAME\"
    }
}")
# </score_endpoint_with_dataset>

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>
# </score_endpoint>

# <delete_endpoint>
curl --location --request DELETE "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" || true
# </delete_endpoint>

```

> [!TIP]
> You can also use other methods to upload, such as the Azure portal or [Azure Storage Explorer](https://azure.microsoft.com/features/storage-explorer/).

Once you upload your code, you can specify your code with a PUT request:

```rest-api
# <create_variables>
SUBSCRIPTION_ID=$(az account show --query id | tr -d '\r"')
RESOURCE_GROUP=$(az group show --query name | tr -d '\r"')
WORKSPACE=$(az configure -l | jq -r '.[] | select(.name=="workspace") | .value')

LOCATION=$(az ml workspace show| jq -r '.location')

API_VERSION="2022-05-01"

TOKEN=$(az account get-access-token --query accessToken -o tsv)
#</create_variables>

# <set_endpoint_name>
export ENDPOINT_NAME=endpt-`echo $RANDOM`
# </set_endpoint_name>

echo "Using: SUBSCRIPTION_ID: $SUBSCRIPTION_ID LOCATION: $LOCATION RESOURCE_GROUP: $RESOURCE_GROUP WORKSPACE: $WORKSPACE ENDPOINT_NAME: $ENDPOINT_NAME"

# <define_wait>  
wait_for_completion () {
    operation_id=$1
    access_token=$2
    status="unknown"

    while [[ $status != "Completed" && $status != "Succeeded" && $status != "Failed" && $status != "Canceled" ]]
    do
        echo "Getting operation status from: $operation_id"
        operation_result=$(curl --location --request GET $operation_id --header "Authorization: Bearer $access_token")
        # TODO error handling here
        status=$(echo $operation_result | jq -r '.status')
        if [[ -z $status || $status == "null" ]]
        then
            status=$(echo $operation_result | jq -r '.properties.status')
        fi

        # Fail early if job submission failed and there is nothing to poll on
        if [[ -z $status || $status == "null" ]]
        then
            echo "No status found on operation, setting to failed."
            status="Failed"
        fi

        echo "Current operation status: $status"
        sleep 10
    done

    if [[ $status == "Failed" ]]
    then
        error=$(echo $operation_result | jq -r '.error')
        echo "Error: $error"
    fi
}
# </define_wait>  

# <get_storage_details>
# Get values for storage account
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores?api-version=$API_VERSION&isDefault=true" \
--header "Authorization: Bearer $TOKEN")
DATASTORE_PATH=$(echo $response | jq -r '.value[0].id')
BLOB_URI_PROTOCOL=$(echo $response | jq -r '.value[0].properties.protocol')
BLOB_URI_ENDPOINT=$(echo $response | jq -r '.value[0].properties.endpoint')
AZUREML_DEFAULT_DATASTORE=$(echo $response | jq -r '.value[0].name')
AZUREML_DEFAULT_CONTAINER=$(echo $response | jq -r '.value[0].properties.containerName')
AZURE_STORAGE_ACCOUNT=$(echo $response | jq -r '.value[0].properties.accountName')
export AZURE_STORAGE_ACCOUNT $(echo $AZURE_STORAGE_ACCOUNT)
STORAGE_RESPONSE=$(echo az storage account show-connection-string --name $AZURE_STORAGE_ACCOUNT)
AZURE_STORAGE_CONNECTION_STRING=$($STORAGE_RESPONSE | jq -r '.connectionString')

BLOB_URI_ROOT="$BLOB_URI_PROTOCOL://$AZURE_STORAGE_ACCOUNT.blob.$BLOB_URI_ENDPOINT/$AZUREML_DEFAULT_CONTAINER"
# </get_storage_details>

# <upload_code>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/score -s endpoints/batch/mnist/code/  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_code>

# <create_code>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
  \"properties\": {
    \"description\": \"Score code\",
    \"codeUri\": \"$BLOB_URI_ROOT/score\"
  }
}")
# </create_code>

# <upload_model>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/model -s endpoints/batch/mnist/model  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_model>

# <create_model>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"modelUri\":\"azureml://subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/workspaces/$WORKSPACE/datastores/$AZUREML_DEFAULT_DATASTORE/paths/model\"
    }
}")
# </create_model>

# <read_condafile>
CONDA_FILE=$(cat endpoints/batch/mnist/environment/conda.json | sed 's/"/\\"/g')
# </read_condafile>
# <create_environment>
ENV_VERSION=$RANDOM
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"condaFile\": $(echo \"$CONDA_FILE\"),
        \"image\": \"mcr.microsoft.com/azureml/openmpi3.1.2-ubuntu18.04:latest\"
    }
}")
# </create_environment>

# <create_compute>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"computeType\": \"AmlCompute\",
        \"properties\": {
            \"osType\": \"Linux\",
            \"vmSize\": \"STANDARD_D2_V2\",
            \"scaleSettings\": {
                \"maxNodeCount\": 5,
                \"minNodeCount\": 0
            },
            \"remoteLoginPortPublicAccess\": \"NotSpecified\"
        },
    },
    \"location\": \"$LOCATION\"
}")
# </create_compute>

#<create_endpoint>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\"
    },
    \"location\": \"$LOCATION\"
}")
#</create_endpoint>

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

# <create_deployment>
DEPLOYMENT_NAME="nonmlflowedp"
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME/deployments/$DEPLOYMENT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"location\": \"$LOCATION\",
    \"properties\": {        
        \"model\": {
            \"referenceType\": \"Id\",
            \"assetId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1\"
        },
        \"codeConfiguration\": {
            \"codeId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1\",
            \"scoringScript\": \"digit_identification.py\"
        },
        \"environmentId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION\",
        \"compute\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster\",
        \"resources\": {
            \"instanceCount\": 1
        },
        \"maxConcurrencyPerInstance\": \"4\",
        \"retrySettings\": {
            \"maxRetries\": 3,
            \"timeout\": \"PT30S\"
        },
        \"errorThreshold\": \"10\",
        \"loggingLevel\": \"info\",
        \"miniBatchSize\": \"5\",
    }
}")
#</create_deployment>
operation_id=$(echo $response | jq -r '.properties.properties.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

#<set_endpoint_defaults>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\",
        \"defaults\": {
            \"deploymentName\": \"$DEPLOYMENT_NAME\"
        }
    },
    \"location\": \"$LOCATION\"
}")

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN
#</set_endpoint_defaults>

# <get_endpoint>
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN")

# <score_endpoint>
SCORING_URI=$(echo $response | jq -r '.properties.scoringUri')
# </get_endpoint>

# <get_access_token>
SCORING_TOKEN=$(az account get-access-token --resource https://ml.azure.com --query accessToken -o tsv)
# </get_access_token>

# <score_endpoint_with_data_in_cloud>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DataUrl\",
            \"Path\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
        }
    }
}")

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# </score_endpoint_with_data_in_cloud>

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>

#<create_dataset>
DATASET_NAME="mnist"
DATASET_VERSION=$RANDOM

response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datasets/$DATASET_NAME/versions/$DATASET_VERSION?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"paths\": [
            {
                \"folder\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
            }
        ]
    }
}")
#</create_dataset>

#<unique_output>
export OUTPUT_FILE_NAME=predictions_`echo $RANDOM`.csv
#</unique_output>

# <score_endpoint_with_dataset>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DatasetVersion\",
            \"datasetName\": \"$DATASET_NAME\",
            \"datasetVersion\": \"$DATASET_VERSION\"
        },
        \"outputDataset\": {
            \"datastoreId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores/workspaceblobstore\",
            \"path\": \"$ENDPOINT_NAME\"
        },
        \"outputFileName\": \"$OUTPUT_FILE_NAME\"
    }
}")
# </score_endpoint_with_dataset>

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>
# </score_endpoint>

# <delete_endpoint>
curl --location --request DELETE "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" || true
# </delete_endpoint>

```

### Upload and register model

Similar to the code, upload the model files:

```rest-api
# <create_variables>
SUBSCRIPTION_ID=$(az account show --query id | tr -d '\r"')
RESOURCE_GROUP=$(az group show --query name | tr -d '\r"')
WORKSPACE=$(az configure -l | jq -r '.[] | select(.name=="workspace") | .value')

LOCATION=$(az ml workspace show| jq -r '.location')

API_VERSION="2022-05-01"

TOKEN=$(az account get-access-token --query accessToken -o tsv)
#</create_variables>

# <set_endpoint_name>
export ENDPOINT_NAME=endpt-`echo $RANDOM`
# </set_endpoint_name>

echo "Using: SUBSCRIPTION_ID: $SUBSCRIPTION_ID LOCATION: $LOCATION RESOURCE_GROUP: $RESOURCE_GROUP WORKSPACE: $WORKSPACE ENDPOINT_NAME: $ENDPOINT_NAME"

# <define_wait>  
wait_for_completion () {
    operation_id=$1
    access_token=$2
    status="unknown"

    while [[ $status != "Completed" && $status != "Succeeded" && $status != "Failed" && $status != "Canceled" ]]
    do
        echo "Getting operation status from: $operation_id"
        operation_result=$(curl --location --request GET $operation_id --header "Authorization: Bearer $access_token")
        # TODO error handling here
        status=$(echo $operation_result | jq -r '.status')
        if [[ -z $status || $status == "null" ]]
        then
            status=$(echo $operation_result | jq -r '.properties.status')
        fi

        # Fail early if job submission failed and there is nothing to poll on
        if [[ -z $status || $status == "null" ]]
        then
            echo "No status found on operation, setting to failed."
            status="Failed"
        fi

        echo "Current operation status: $status"
        sleep 10
    done

    if [[ $status == "Failed" ]]
    then
        error=$(echo $operation_result | jq -r '.error')
        echo "Error: $error"
    fi
}
# </define_wait>  

# <get_storage_details>
# Get values for storage account
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores?api-version=$API_VERSION&isDefault=true" \
--header "Authorization: Bearer $TOKEN")
DATASTORE_PATH=$(echo $response | jq -r '.value[0].id')
BLOB_URI_PROTOCOL=$(echo $response | jq -r '.value[0].properties.protocol')
BLOB_URI_ENDPOINT=$(echo $response | jq -r '.value[0].properties.endpoint')
AZUREML_DEFAULT_DATASTORE=$(echo $response | jq -r '.value[0].name')
AZUREML_DEFAULT_CONTAINER=$(echo $response | jq -r '.value[0].properties.containerName')
AZURE_STORAGE_ACCOUNT=$(echo $response | jq -r '.value[0].properties.accountName')
export AZURE_STORAGE_ACCOUNT $(echo $AZURE_STORAGE_ACCOUNT)
STORAGE_RESPONSE=$(echo az storage account show-connection-string --name $AZURE_STORAGE_ACCOUNT)
AZURE_STORAGE_CONNECTION_STRING=$($STORAGE_RESPONSE | jq -r '.connectionString')

BLOB_URI_ROOT="$BLOB_URI_PROTOCOL://$AZURE_STORAGE_ACCOUNT.blob.$BLOB_URI_ENDPOINT/$AZUREML_DEFAULT_CONTAINER"
# </get_storage_details>

# <upload_code>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/score -s endpoints/batch/mnist/code/  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_code>

# <create_code>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
  \"properties\": {
    \"description\": \"Score code\",
    \"codeUri\": \"$BLOB_URI_ROOT/score\"
  }
}")
# </create_code>

# <upload_model>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/model -s endpoints/batch/mnist/model  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_model>

# <create_model>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"modelUri\":\"azureml://subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/workspaces/$WORKSPACE/datastores/$AZUREML_DEFAULT_DATASTORE/paths/model\"
    }
}")
# </create_model>

# <read_condafile>
CONDA_FILE=$(cat endpoints/batch/mnist/environment/conda.json | sed 's/"/\\"/g')
# </read_condafile>
# <create_environment>
ENV_VERSION=$RANDOM
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"condaFile\": $(echo \"$CONDA_FILE\"),
        \"image\": \"mcr.microsoft.com/azureml/openmpi3.1.2-ubuntu18.04:latest\"
    }
}")
# </create_environment>

# <create_compute>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"computeType\": \"AmlCompute\",
        \"properties\": {
            \"osType\": \"Linux\",
            \"vmSize\": \"STANDARD_D2_V2\",
            \"scaleSettings\": {
                \"maxNodeCount\": 5,
                \"minNodeCount\": 0
            },
            \"remoteLoginPortPublicAccess\": \"NotSpecified\"
        },
    },
    \"location\": \"$LOCATION\"
}")
# </create_compute>

#<create_endpoint>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\"
    },
    \"location\": \"$LOCATION\"
}")
#</create_endpoint>

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

# <create_deployment>
DEPLOYMENT_NAME="nonmlflowedp"
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME/deployments/$DEPLOYMENT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"location\": \"$LOCATION\",
    \"properties\": {        
        \"model\": {
            \"referenceType\": \"Id\",
            \"assetId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1\"
        },
        \"codeConfiguration\": {
            \"codeId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1\",
            \"scoringScript\": \"digit_identification.py\"
        },
        \"environmentId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION\",
        \"compute\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster\",
        \"resources\": {
            \"instanceCount\": 1
        },
        \"maxConcurrencyPerInstance\": \"4\",
        \"retrySettings\": {
            \"maxRetries\": 3,
            \"timeout\": \"PT30S\"
        },
        \"errorThreshold\": \"10\",
        \"loggingLevel\": \"info\",
        \"miniBatchSize\": \"5\",
    }
}")
#</create_deployment>
operation_id=$(echo $response | jq -r '.properties.properties.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

#<set_endpoint_defaults>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\",
        \"defaults\": {
            \"deploymentName\": \"$DEPLOYMENT_NAME\"
        }
    },
    \"location\": \"$LOCATION\"
}")

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN
#</set_endpoint_defaults>

# <get_endpoint>
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN")

# <score_endpoint>
SCORING_URI=$(echo $response | jq -r '.properties.scoringUri')
# </get_endpoint>

# <get_access_token>
SCORING_TOKEN=$(az account get-access-token --resource https://ml.azure.com --query accessToken -o tsv)
# </get_access_token>

# <score_endpoint_with_data_in_cloud>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DataUrl\",
            \"Path\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
        }
    }
}")

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# </score_endpoint_with_data_in_cloud>

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>

#<create_dataset>
DATASET_NAME="mnist"
DATASET_VERSION=$RANDOM

response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datasets/$DATASET_NAME/versions/$DATASET_VERSION?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"paths\": [
            {
                \"folder\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
            }
        ]
    }
}")
#</create_dataset>

#<unique_output>
export OUTPUT_FILE_NAME=predictions_`echo $RANDOM`.csv
#</unique_output>

# <score_endpoint_with_dataset>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DatasetVersion\",
            \"datasetName\": \"$DATASET_NAME\",
            \"datasetVersion\": \"$DATASET_VERSION\"
        },
        \"outputDataset\": {
            \"datastoreId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores/workspaceblobstore\",
            \"path\": \"$ENDPOINT_NAME\"
        },
        \"outputFileName\": \"$OUTPUT_FILE_NAME\"
    }
}")
# </score_endpoint_with_dataset>

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>
# </score_endpoint>

# <delete_endpoint>
curl --location --request DELETE "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" || true
# </delete_endpoint>

```

Now, register the model:

```rest-api
# <create_variables>
SUBSCRIPTION_ID=$(az account show --query id | tr -d '\r"')
RESOURCE_GROUP=$(az group show --query name | tr -d '\r"')
WORKSPACE=$(az configure -l | jq -r '.[] | select(.name=="workspace") | .value')

LOCATION=$(az ml workspace show| jq -r '.location')

API_VERSION="2022-05-01"

TOKEN=$(az account get-access-token --query accessToken -o tsv)
#</create_variables>

# <set_endpoint_name>
export ENDPOINT_NAME=endpt-`echo $RANDOM`
# </set_endpoint_name>

echo "Using: SUBSCRIPTION_ID: $SUBSCRIPTION_ID LOCATION: $LOCATION RESOURCE_GROUP: $RESOURCE_GROUP WORKSPACE: $WORKSPACE ENDPOINT_NAME: $ENDPOINT_NAME"

# <define_wait>  
wait_for_completion () {
    operation_id=$1
    access_token=$2
    status="unknown"

    while [[ $status != "Completed" && $status != "Succeeded" && $status != "Failed" && $status != "Canceled" ]]
    do
        echo "Getting operation status from: $operation_id"
        operation_result=$(curl --location --request GET $operation_id --header "Authorization: Bearer $access_token")
        # TODO error handling here
        status=$(echo $operation_result | jq -r '.status')
        if [[ -z $status || $status == "null" ]]
        then
            status=$(echo $operation_result | jq -r '.properties.status')
        fi

        # Fail early if job submission failed and there is nothing to poll on
        if [[ -z $status || $status == "null" ]]
        then
            echo "No status found on operation, setting to failed."
            status="Failed"
        fi

        echo "Current operation status: $status"
        sleep 10
    done

    if [[ $status == "Failed" ]]
    then
        error=$(echo $operation_result | jq -r '.error')
        echo "Error: $error"
    fi
}
# </define_wait>  

# <get_storage_details>
# Get values for storage account
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores?api-version=$API_VERSION&isDefault=true" \
--header "Authorization: Bearer $TOKEN")
DATASTORE_PATH=$(echo $response | jq -r '.value[0].id')
BLOB_URI_PROTOCOL=$(echo $response | jq -r '.value[0].properties.protocol')
BLOB_URI_ENDPOINT=$(echo $response | jq -r '.value[0].properties.endpoint')
AZUREML_DEFAULT_DATASTORE=$(echo $response | jq -r '.value[0].name')
AZUREML_DEFAULT_CONTAINER=$(echo $response | jq -r '.value[0].properties.containerName')
AZURE_STORAGE_ACCOUNT=$(echo $response | jq -r '.value[0].properties.accountName')
export AZURE_STORAGE_ACCOUNT $(echo $AZURE_STORAGE_ACCOUNT)
STORAGE_RESPONSE=$(echo az storage account show-connection-string --name $AZURE_STORAGE_ACCOUNT)
AZURE_STORAGE_CONNECTION_STRING=$($STORAGE_RESPONSE | jq -r '.connectionString')

BLOB_URI_ROOT="$BLOB_URI_PROTOCOL://$AZURE_STORAGE_ACCOUNT.blob.$BLOB_URI_ENDPOINT/$AZUREML_DEFAULT_CONTAINER"
# </get_storage_details>

# <upload_code>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/score -s endpoints/batch/mnist/code/  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_code>

# <create_code>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
  \"properties\": {
    \"description\": \"Score code\",
    \"codeUri\": \"$BLOB_URI_ROOT/score\"
  }
}")
# </create_code>

# <upload_model>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/model -s endpoints/batch/mnist/model  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_model>

# <create_model>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"modelUri\":\"azureml://subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/workspaces/$WORKSPACE/datastores/$AZUREML_DEFAULT_DATASTORE/paths/model\"
    }
}")
# </create_model>

# <read_condafile>
CONDA_FILE=$(cat endpoints/batch/mnist/environment/conda.json | sed 's/"/\\"/g')
# </read_condafile>
# <create_environment>
ENV_VERSION=$RANDOM
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"condaFile\": $(echo \"$CONDA_FILE\"),
        \"image\": \"mcr.microsoft.com/azureml/openmpi3.1.2-ubuntu18.04:latest\"
    }
}")
# </create_environment>

# <create_compute>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"computeType\": \"AmlCompute\",
        \"properties\": {
            \"osType\": \"Linux\",
            \"vmSize\": \"STANDARD_D2_V2\",
            \"scaleSettings\": {
                \"maxNodeCount\": 5,
                \"minNodeCount\": 0
            },
            \"remoteLoginPortPublicAccess\": \"NotSpecified\"
        },
    },
    \"location\": \"$LOCATION\"
}")
# </create_compute>

#<create_endpoint>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\"
    },
    \"location\": \"$LOCATION\"
}")
#</create_endpoint>

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

# <create_deployment>
DEPLOYMENT_NAME="nonmlflowedp"
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME/deployments/$DEPLOYMENT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"location\": \"$LOCATION\",
    \"properties\": {        
        \"model\": {
            \"referenceType\": \"Id\",
            \"assetId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1\"
        },
        \"codeConfiguration\": {
            \"codeId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1\",
            \"scoringScript\": \"digit_identification.py\"
        },
        \"environmentId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION\",
        \"compute\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster\",
        \"resources\": {
            \"instanceCount\": 1
        },
        \"maxConcurrencyPerInstance\": \"4\",
        \"retrySettings\": {
            \"maxRetries\": 3,
            \"timeout\": \"PT30S\"
        },
        \"errorThreshold\": \"10\",
        \"loggingLevel\": \"info\",
        \"miniBatchSize\": \"5\",
    }
}")
#</create_deployment>
operation_id=$(echo $response | jq -r '.properties.properties.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

#<set_endpoint_defaults>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\",
        \"defaults\": {
            \"deploymentName\": \"$DEPLOYMENT_NAME\"
        }
    },
    \"location\": \"$LOCATION\"
}")

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN
#</set_endpoint_defaults>

# <get_endpoint>
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN")

# <score_endpoint>
SCORING_URI=$(echo $response | jq -r '.properties.scoringUri')
# </get_endpoint>

# <get_access_token>
SCORING_TOKEN=$(az account get-access-token --resource https://ml.azure.com --query accessToken -o tsv)
# </get_access_token>

# <score_endpoint_with_data_in_cloud>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DataUrl\",
            \"Path\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
        }
    }
}")

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# </score_endpoint_with_data_in_cloud>

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>

#<create_dataset>
DATASET_NAME="mnist"
DATASET_VERSION=$RANDOM

response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datasets/$DATASET_NAME/versions/$DATASET_VERSION?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"paths\": [
            {
                \"folder\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
            }
        ]
    }
}")
#</create_dataset>

#<unique_output>
export OUTPUT_FILE_NAME=predictions_`echo $RANDOM`.csv
#</unique_output>

# <score_endpoint_with_dataset>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DatasetVersion\",
            \"datasetName\": \"$DATASET_NAME\",
            \"datasetVersion\": \"$DATASET_VERSION\"
        },
        \"outputDataset\": {
            \"datastoreId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores/workspaceblobstore\",
            \"path\": \"$ENDPOINT_NAME\"
        },
        \"outputFileName\": \"$OUTPUT_FILE_NAME\"
    }
}")
# </score_endpoint_with_dataset>

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>
# </score_endpoint>

# <delete_endpoint>
curl --location --request DELETE "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" || true
# </delete_endpoint>

```

### Create environment
The deployment needs to run in an environment that has the required dependencies. Create the environment with a PUT request. Use a docker image from Microsoft Container Registry. You can configure the docker image with `image` and add conda dependencies with `condaFile`.

Run the following code to read the `condaFile` defined in json. The source file is at `/cli/endpoints/batch/mnist/environment/conda.json` in the example repository:

```rest-api
# <create_variables>
SUBSCRIPTION_ID=$(az account show --query id | tr -d '\r"')
RESOURCE_GROUP=$(az group show --query name | tr -d '\r"')
WORKSPACE=$(az configure -l | jq -r '.[] | select(.name=="workspace") | .value')

LOCATION=$(az ml workspace show| jq -r '.location')

API_VERSION="2022-05-01"

TOKEN=$(az account get-access-token --query accessToken -o tsv)
#</create_variables>

# <set_endpoint_name>
export ENDPOINT_NAME=endpt-`echo $RANDOM`
# </set_endpoint_name>

echo "Using: SUBSCRIPTION_ID: $SUBSCRIPTION_ID LOCATION: $LOCATION RESOURCE_GROUP: $RESOURCE_GROUP WORKSPACE: $WORKSPACE ENDPOINT_NAME: $ENDPOINT_NAME"

# <define_wait>  
wait_for_completion () {
    operation_id=$1
    access_token=$2
    status="unknown"

    while [[ $status != "Completed" && $status != "Succeeded" && $status != "Failed" && $status != "Canceled" ]]
    do
        echo "Getting operation status from: $operation_id"
        operation_result=$(curl --location --request GET $operation_id --header "Authorization: Bearer $access_token")
        # TODO error handling here
        status=$(echo $operation_result | jq -r '.status')
        if [[ -z $status || $status == "null" ]]
        then
            status=$(echo $operation_result | jq -r '.properties.status')
        fi

        # Fail early if job submission failed and there is nothing to poll on
        if [[ -z $status || $status == "null" ]]
        then
            echo "No status found on operation, setting to failed."
            status="Failed"
        fi

        echo "Current operation status: $status"
        sleep 10
    done

    if [[ $status == "Failed" ]]
    then
        error=$(echo $operation_result | jq -r '.error')
        echo "Error: $error"
    fi
}
# </define_wait>  

# <get_storage_details>
# Get values for storage account
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores?api-version=$API_VERSION&isDefault=true" \
--header "Authorization: Bearer $TOKEN")
DATASTORE_PATH=$(echo $response | jq -r '.value[0].id')
BLOB_URI_PROTOCOL=$(echo $response | jq -r '.value[0].properties.protocol')
BLOB_URI_ENDPOINT=$(echo $response | jq -r '.value[0].properties.endpoint')
AZUREML_DEFAULT_DATASTORE=$(echo $response | jq -r '.value[0].name')
AZUREML_DEFAULT_CONTAINER=$(echo $response | jq -r '.value[0].properties.containerName')
AZURE_STORAGE_ACCOUNT=$(echo $response | jq -r '.value[0].properties.accountName')
export AZURE_STORAGE_ACCOUNT $(echo $AZURE_STORAGE_ACCOUNT)
STORAGE_RESPONSE=$(echo az storage account show-connection-string --name $AZURE_STORAGE_ACCOUNT)
AZURE_STORAGE_CONNECTION_STRING=$($STORAGE_RESPONSE | jq -r '.connectionString')

BLOB_URI_ROOT="$BLOB_URI_PROTOCOL://$AZURE_STORAGE_ACCOUNT.blob.$BLOB_URI_ENDPOINT/$AZUREML_DEFAULT_CONTAINER"
# </get_storage_details>

# <upload_code>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/score -s endpoints/batch/mnist/code/  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_code>

# <create_code>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
  \"properties\": {
    \"description\": \"Score code\",
    \"codeUri\": \"$BLOB_URI_ROOT/score\"
  }
}")
# </create_code>

# <upload_model>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/model -s endpoints/batch/mnist/model  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_model>

# <create_model>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"modelUri\":\"azureml://subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/workspaces/$WORKSPACE/datastores/$AZUREML_DEFAULT_DATASTORE/paths/model\"
    }
}")
# </create_model>

# <read_condafile>
CONDA_FILE=$(cat endpoints/batch/mnist/environment/conda.json | sed 's/"/\\"/g')
# </read_condafile>
# <create_environment>
ENV_VERSION=$RANDOM
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"condaFile\": $(echo \"$CONDA_FILE\"),
        \"image\": \"mcr.microsoft.com/azureml/openmpi3.1.2-ubuntu18.04:latest\"
    }
}")
# </create_environment>

# <create_compute>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"computeType\": \"AmlCompute\",
        \"properties\": {
            \"osType\": \"Linux\",
            \"vmSize\": \"STANDARD_D2_V2\",
            \"scaleSettings\": {
                \"maxNodeCount\": 5,
                \"minNodeCount\": 0
            },
            \"remoteLoginPortPublicAccess\": \"NotSpecified\"
        },
    },
    \"location\": \"$LOCATION\"
}")
# </create_compute>

#<create_endpoint>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\"
    },
    \"location\": \"$LOCATION\"
}")
#</create_endpoint>

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

# <create_deployment>
DEPLOYMENT_NAME="nonmlflowedp"
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME/deployments/$DEPLOYMENT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"location\": \"$LOCATION\",
    \"properties\": {        
        \"model\": {
            \"referenceType\": \"Id\",
            \"assetId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1\"
        },
        \"codeConfiguration\": {
            \"codeId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1\",
            \"scoringScript\": \"digit_identification.py\"
        },
        \"environmentId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION\",
        \"compute\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster\",
        \"resources\": {
            \"instanceCount\": 1
        },
        \"maxConcurrencyPerInstance\": \"4\",
        \"retrySettings\": {
            \"maxRetries\": 3,
            \"timeout\": \"PT30S\"
        },
        \"errorThreshold\": \"10\",
        \"loggingLevel\": \"info\",
        \"miniBatchSize\": \"5\",
    }
}")
#</create_deployment>
operation_id=$(echo $response | jq -r '.properties.properties.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

#<set_endpoint_defaults>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\",
        \"defaults\": {
            \"deploymentName\": \"$DEPLOYMENT_NAME\"
        }
    },
    \"location\": \"$LOCATION\"
}")

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN
#</set_endpoint_defaults>

# <get_endpoint>
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN")

# <score_endpoint>
SCORING_URI=$(echo $response | jq -r '.properties.scoringUri')
# </get_endpoint>

# <get_access_token>
SCORING_TOKEN=$(az account get-access-token --resource https://ml.azure.com --query accessToken -o tsv)
# </get_access_token>

# <score_endpoint_with_data_in_cloud>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DataUrl\",
            \"Path\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
        }
    }
}")

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# </score_endpoint_with_data_in_cloud>

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>

#<create_dataset>
DATASET_NAME="mnist"
DATASET_VERSION=$RANDOM

response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datasets/$DATASET_NAME/versions/$DATASET_VERSION?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"paths\": [
            {
                \"folder\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
            }
        ]
    }
}")
#</create_dataset>

#<unique_output>
export OUTPUT_FILE_NAME=predictions_`echo $RANDOM`.csv
#</unique_output>

# <score_endpoint_with_dataset>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DatasetVersion\",
            \"datasetName\": \"$DATASET_NAME\",
            \"datasetVersion\": \"$DATASET_VERSION\"
        },
        \"outputDataset\": {
            \"datastoreId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores/workspaceblobstore\",
            \"path\": \"$ENDPOINT_NAME\"
        },
        \"outputFileName\": \"$OUTPUT_FILE_NAME\"
    }
}")
# </score_endpoint_with_dataset>

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>
# </score_endpoint>

# <delete_endpoint>
curl --location --request DELETE "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" || true
# </delete_endpoint>

```

Now, run the following snippet to create an environment:

```rest-api
# <create_variables>
SUBSCRIPTION_ID=$(az account show --query id | tr -d '\r"')
RESOURCE_GROUP=$(az group show --query name | tr -d '\r"')
WORKSPACE=$(az configure -l | jq -r '.[] | select(.name=="workspace") | .value')

LOCATION=$(az ml workspace show| jq -r '.location')

API_VERSION="2022-05-01"

TOKEN=$(az account get-access-token --query accessToken -o tsv)
#</create_variables>

# <set_endpoint_name>
export ENDPOINT_NAME=endpt-`echo $RANDOM`
# </set_endpoint_name>

echo "Using: SUBSCRIPTION_ID: $SUBSCRIPTION_ID LOCATION: $LOCATION RESOURCE_GROUP: $RESOURCE_GROUP WORKSPACE: $WORKSPACE ENDPOINT_NAME: $ENDPOINT_NAME"

# <define_wait>  
wait_for_completion () {
    operation_id=$1
    access_token=$2
    status="unknown"

    while [[ $status != "Completed" && $status != "Succeeded" && $status != "Failed" && $status != "Canceled" ]]
    do
        echo "Getting operation status from: $operation_id"
        operation_result=$(curl --location --request GET $operation_id --header "Authorization: Bearer $access_token")
        # TODO error handling here
        status=$(echo $operation_result | jq -r '.status')
        if [[ -z $status || $status == "null" ]]
        then
            status=$(echo $operation_result | jq -r '.properties.status')
        fi

        # Fail early if job submission failed and there is nothing to poll on
        if [[ -z $status || $status == "null" ]]
        then
            echo "No status found on operation, setting to failed."
            status="Failed"
        fi

        echo "Current operation status: $status"
        sleep 10
    done

    if [[ $status == "Failed" ]]
    then
        error=$(echo $operation_result | jq -r '.error')
        echo "Error: $error"
    fi
}
# </define_wait>  

# <get_storage_details>
# Get values for storage account
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores?api-version=$API_VERSION&isDefault=true" \
--header "Authorization: Bearer $TOKEN")
DATASTORE_PATH=$(echo $response | jq -r '.value[0].id')
BLOB_URI_PROTOCOL=$(echo $response | jq -r '.value[0].properties.protocol')
BLOB_URI_ENDPOINT=$(echo $response | jq -r '.value[0].properties.endpoint')
AZUREML_DEFAULT_DATASTORE=$(echo $response | jq -r '.value[0].name')
AZUREML_DEFAULT_CONTAINER=$(echo $response | jq -r '.value[0].properties.containerName')
AZURE_STORAGE_ACCOUNT=$(echo $response | jq -r '.value[0].properties.accountName')
export AZURE_STORAGE_ACCOUNT $(echo $AZURE_STORAGE_ACCOUNT)
STORAGE_RESPONSE=$(echo az storage account show-connection-string --name $AZURE_STORAGE_ACCOUNT)
AZURE_STORAGE_CONNECTION_STRING=$($STORAGE_RESPONSE | jq -r '.connectionString')

BLOB_URI_ROOT="$BLOB_URI_PROTOCOL://$AZURE_STORAGE_ACCOUNT.blob.$BLOB_URI_ENDPOINT/$AZUREML_DEFAULT_CONTAINER"
# </get_storage_details>

# <upload_code>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/score -s endpoints/batch/mnist/code/  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_code>

# <create_code>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
  \"properties\": {
    \"description\": \"Score code\",
    \"codeUri\": \"$BLOB_URI_ROOT/score\"
  }
}")
# </create_code>

# <upload_model>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/model -s endpoints/batch/mnist/model  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_model>

# <create_model>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"modelUri\":\"azureml://subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/workspaces/$WORKSPACE/datastores/$AZUREML_DEFAULT_DATASTORE/paths/model\"
    }
}")
# </create_model>

# <read_condafile>
CONDA_FILE=$(cat endpoints/batch/mnist/environment/conda.json | sed 's/"/\\"/g')
# </read_condafile>
# <create_environment>
ENV_VERSION=$RANDOM
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"condaFile\": $(echo \"$CONDA_FILE\"),
        \"image\": \"mcr.microsoft.com/azureml/openmpi3.1.2-ubuntu18.04:latest\"
    }
}")
# </create_environment>

# <create_compute>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"computeType\": \"AmlCompute\",
        \"properties\": {
            \"osType\": \"Linux\",
            \"vmSize\": \"STANDARD_D2_V2\",
            \"scaleSettings\": {
                \"maxNodeCount\": 5,
                \"minNodeCount\": 0
            },
            \"remoteLoginPortPublicAccess\": \"NotSpecified\"
        },
    },
    \"location\": \"$LOCATION\"
}")
# </create_compute>

#<create_endpoint>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\"
    },
    \"location\": \"$LOCATION\"
}")
#</create_endpoint>

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

# <create_deployment>
DEPLOYMENT_NAME="nonmlflowedp"
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME/deployments/$DEPLOYMENT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"location\": \"$LOCATION\",
    \"properties\": {        
        \"model\": {
            \"referenceType\": \"Id\",
            \"assetId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1\"
        },
        \"codeConfiguration\": {
            \"codeId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1\",
            \"scoringScript\": \"digit_identification.py\"
        },
        \"environmentId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION\",
        \"compute\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster\",
        \"resources\": {
            \"instanceCount\": 1
        },
        \"maxConcurrencyPerInstance\": \"4\",
        \"retrySettings\": {
            \"maxRetries\": 3,
            \"timeout\": \"PT30S\"
        },
        \"errorThreshold\": \"10\",
        \"loggingLevel\": \"info\",
        \"miniBatchSize\": \"5\",
    }
}")
#</create_deployment>
operation_id=$(echo $response | jq -r '.properties.properties.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

#<set_endpoint_defaults>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\",
        \"defaults\": {
            \"deploymentName\": \"$DEPLOYMENT_NAME\"
        }
    },
    \"location\": \"$LOCATION\"
}")

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN
#</set_endpoint_defaults>

# <get_endpoint>
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN")

# <score_endpoint>
SCORING_URI=$(echo $response | jq -r '.properties.scoringUri')
# </get_endpoint>

# <get_access_token>
SCORING_TOKEN=$(az account get-access-token --resource https://ml.azure.com --query accessToken -o tsv)
# </get_access_token>

# <score_endpoint_with_data_in_cloud>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DataUrl\",
            \"Path\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
        }
    }
}")

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# </score_endpoint_with_data_in_cloud>

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>

#<create_dataset>
DATASET_NAME="mnist"
DATASET_VERSION=$RANDOM

response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datasets/$DATASET_NAME/versions/$DATASET_VERSION?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"paths\": [
            {
                \"folder\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
            }
        ]
    }
}")
#</create_dataset>

#<unique_output>
export OUTPUT_FILE_NAME=predictions_`echo $RANDOM`.csv
#</unique_output>

# <score_endpoint_with_dataset>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DatasetVersion\",
            \"datasetName\": \"$DATASET_NAME\",
            \"datasetVersion\": \"$DATASET_VERSION\"
        },
        \"outputDataset\": {
            \"datastoreId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores/workspaceblobstore\",
            \"path\": \"$ENDPOINT_NAME\"
        },
        \"outputFileName\": \"$OUTPUT_FILE_NAME\"
    }
}")
# </score_endpoint_with_dataset>

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>
# </score_endpoint>

# <delete_endpoint>
curl --location --request DELETE "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" || true
# </delete_endpoint>

```

## Deploy with batch endpoints

Next, create a batch endpoint, a batch deployment, and set the default deployment for the endpoint.

### Create batch endpoint

Create the batch endpoint:

```rest-api
# <create_variables>
SUBSCRIPTION_ID=$(az account show --query id | tr -d '\r"')
RESOURCE_GROUP=$(az group show --query name | tr -d '\r"')
WORKSPACE=$(az configure -l | jq -r '.[] | select(.name=="workspace") | .value')

LOCATION=$(az ml workspace show| jq -r '.location')

API_VERSION="2022-05-01"

TOKEN=$(az account get-access-token --query accessToken -o tsv)
#</create_variables>

# <set_endpoint_name>
export ENDPOINT_NAME=endpt-`echo $RANDOM`
# </set_endpoint_name>

echo "Using: SUBSCRIPTION_ID: $SUBSCRIPTION_ID LOCATION: $LOCATION RESOURCE_GROUP: $RESOURCE_GROUP WORKSPACE: $WORKSPACE ENDPOINT_NAME: $ENDPOINT_NAME"

# <define_wait>  
wait_for_completion () {
    operation_id=$1
    access_token=$2
    status="unknown"

    while [[ $status != "Completed" && $status != "Succeeded" && $status != "Failed" && $status != "Canceled" ]]
    do
        echo "Getting operation status from: $operation_id"
        operation_result=$(curl --location --request GET $operation_id --header "Authorization: Bearer $access_token")
        # TODO error handling here
        status=$(echo $operation_result | jq -r '.status')
        if [[ -z $status || $status == "null" ]]
        then
            status=$(echo $operation_result | jq -r '.properties.status')
        fi

        # Fail early if job submission failed and there is nothing to poll on
        if [[ -z $status || $status == "null" ]]
        then
            echo "No status found on operation, setting to failed."
            status="Failed"
        fi

        echo "Current operation status: $status"
        sleep 10
    done

    if [[ $status == "Failed" ]]
    then
        error=$(echo $operation_result | jq -r '.error')
        echo "Error: $error"
    fi
}
# </define_wait>  

# <get_storage_details>
# Get values for storage account
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores?api-version=$API_VERSION&isDefault=true" \
--header "Authorization: Bearer $TOKEN")
DATASTORE_PATH=$(echo $response | jq -r '.value[0].id')
BLOB_URI_PROTOCOL=$(echo $response | jq -r '.value[0].properties.protocol')
BLOB_URI_ENDPOINT=$(echo $response | jq -r '.value[0].properties.endpoint')
AZUREML_DEFAULT_DATASTORE=$(echo $response | jq -r '.value[0].name')
AZUREML_DEFAULT_CONTAINER=$(echo $response | jq -r '.value[0].properties.containerName')
AZURE_STORAGE_ACCOUNT=$(echo $response | jq -r '.value[0].properties.accountName')
export AZURE_STORAGE_ACCOUNT $(echo $AZURE_STORAGE_ACCOUNT)
STORAGE_RESPONSE=$(echo az storage account show-connection-string --name $AZURE_STORAGE_ACCOUNT)
AZURE_STORAGE_CONNECTION_STRING=$($STORAGE_RESPONSE | jq -r '.connectionString')

BLOB_URI_ROOT="$BLOB_URI_PROTOCOL://$AZURE_STORAGE_ACCOUNT.blob.$BLOB_URI_ENDPOINT/$AZUREML_DEFAULT_CONTAINER"
# </get_storage_details>

# <upload_code>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/score -s endpoints/batch/mnist/code/  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_code>

# <create_code>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
  \"properties\": {
    \"description\": \"Score code\",
    \"codeUri\": \"$BLOB_URI_ROOT/score\"
  }
}")
# </create_code>

# <upload_model>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/model -s endpoints/batch/mnist/model  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_model>

# <create_model>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"modelUri\":\"azureml://subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/workspaces/$WORKSPACE/datastores/$AZUREML_DEFAULT_DATASTORE/paths/model\"
    }
}")
# </create_model>

# <read_condafile>
CONDA_FILE=$(cat endpoints/batch/mnist/environment/conda.json | sed 's/"/\\"/g')
# </read_condafile>
# <create_environment>
ENV_VERSION=$RANDOM
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"condaFile\": $(echo \"$CONDA_FILE\"),
        \"image\": \"mcr.microsoft.com/azureml/openmpi3.1.2-ubuntu18.04:latest\"
    }
}")
# </create_environment>

# <create_compute>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"computeType\": \"AmlCompute\",
        \"properties\": {
            \"osType\": \"Linux\",
            \"vmSize\": \"STANDARD_D2_V2\",
            \"scaleSettings\": {
                \"maxNodeCount\": 5,
                \"minNodeCount\": 0
            },
            \"remoteLoginPortPublicAccess\": \"NotSpecified\"
        },
    },
    \"location\": \"$LOCATION\"
}")
# </create_compute>

#<create_endpoint>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\"
    },
    \"location\": \"$LOCATION\"
}")
#</create_endpoint>

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

# <create_deployment>
DEPLOYMENT_NAME="nonmlflowedp"
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME/deployments/$DEPLOYMENT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"location\": \"$LOCATION\",
    \"properties\": {        
        \"model\": {
            \"referenceType\": \"Id\",
            \"assetId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1\"
        },
        \"codeConfiguration\": {
            \"codeId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1\",
            \"scoringScript\": \"digit_identification.py\"
        },
        \"environmentId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION\",
        \"compute\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster\",
        \"resources\": {
            \"instanceCount\": 1
        },
        \"maxConcurrencyPerInstance\": \"4\",
        \"retrySettings\": {
            \"maxRetries\": 3,
            \"timeout\": \"PT30S\"
        },
        \"errorThreshold\": \"10\",
        \"loggingLevel\": \"info\",
        \"miniBatchSize\": \"5\",
    }
}")
#</create_deployment>
operation_id=$(echo $response | jq -r '.properties.properties.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

#<set_endpoint_defaults>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\",
        \"defaults\": {
            \"deploymentName\": \"$DEPLOYMENT_NAME\"
        }
    },
    \"location\": \"$LOCATION\"
}")

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN
#</set_endpoint_defaults>

# <get_endpoint>
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN")

# <score_endpoint>
SCORING_URI=$(echo $response | jq -r '.properties.scoringUri')
# </get_endpoint>

# <get_access_token>
SCORING_TOKEN=$(az account get-access-token --resource https://ml.azure.com --query accessToken -o tsv)
# </get_access_token>

# <score_endpoint_with_data_in_cloud>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DataUrl\",
            \"Path\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
        }
    }
}")

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# </score_endpoint_with_data_in_cloud>

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>

#<create_dataset>
DATASET_NAME="mnist"
DATASET_VERSION=$RANDOM

response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datasets/$DATASET_NAME/versions/$DATASET_VERSION?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"paths\": [
            {
                \"folder\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
            }
        ]
    }
}")
#</create_dataset>

#<unique_output>
export OUTPUT_FILE_NAME=predictions_`echo $RANDOM`.csv
#</unique_output>

# <score_endpoint_with_dataset>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DatasetVersion\",
            \"datasetName\": \"$DATASET_NAME\",
            \"datasetVersion\": \"$DATASET_VERSION\"
        },
        \"outputDataset\": {
            \"datastoreId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores/workspaceblobstore\",
            \"path\": \"$ENDPOINT_NAME\"
        },
        \"outputFileName\": \"$OUTPUT_FILE_NAME\"
    }
}")
# </score_endpoint_with_dataset>

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>
# </score_endpoint>

# <delete_endpoint>
curl --location --request DELETE "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" || true
# </delete_endpoint>

```

### Create batch deployment

Create a batch deployment under the endpoint:

```rest-api
# <create_variables>
SUBSCRIPTION_ID=$(az account show --query id | tr -d '\r"')
RESOURCE_GROUP=$(az group show --query name | tr -d '\r"')
WORKSPACE=$(az configure -l | jq -r '.[] | select(.name=="workspace") | .value')

LOCATION=$(az ml workspace show| jq -r '.location')

API_VERSION="2022-05-01"

TOKEN=$(az account get-access-token --query accessToken -o tsv)
#</create_variables>

# <set_endpoint_name>
export ENDPOINT_NAME=endpt-`echo $RANDOM`
# </set_endpoint_name>

echo "Using: SUBSCRIPTION_ID: $SUBSCRIPTION_ID LOCATION: $LOCATION RESOURCE_GROUP: $RESOURCE_GROUP WORKSPACE: $WORKSPACE ENDPOINT_NAME: $ENDPOINT_NAME"

# <define_wait>  
wait_for_completion () {
    operation_id=$1
    access_token=$2
    status="unknown"

    while [[ $status != "Completed" && $status != "Succeeded" && $status != "Failed" && $status != "Canceled" ]]
    do
        echo "Getting operation status from: $operation_id"
        operation_result=$(curl --location --request GET $operation_id --header "Authorization: Bearer $access_token")
        # TODO error handling here
        status=$(echo $operation_result | jq -r '.status')
        if [[ -z $status || $status == "null" ]]
        then
            status=$(echo $operation_result | jq -r '.properties.status')
        fi

        # Fail early if job submission failed and there is nothing to poll on
        if [[ -z $status || $status == "null" ]]
        then
            echo "No status found on operation, setting to failed."
            status="Failed"
        fi

        echo "Current operation status: $status"
        sleep 10
    done

    if [[ $status == "Failed" ]]
    then
        error=$(echo $operation_result | jq -r '.error')
        echo "Error: $error"
    fi
}
# </define_wait>  

# <get_storage_details>
# Get values for storage account
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores?api-version=$API_VERSION&isDefault=true" \
--header "Authorization: Bearer $TOKEN")
DATASTORE_PATH=$(echo $response | jq -r '.value[0].id')
BLOB_URI_PROTOCOL=$(echo $response | jq -r '.value[0].properties.protocol')
BLOB_URI_ENDPOINT=$(echo $response | jq -r '.value[0].properties.endpoint')
AZUREML_DEFAULT_DATASTORE=$(echo $response | jq -r '.value[0].name')
AZUREML_DEFAULT_CONTAINER=$(echo $response | jq -r '.value[0].properties.containerName')
AZURE_STORAGE_ACCOUNT=$(echo $response | jq -r '.value[0].properties.accountName')
export AZURE_STORAGE_ACCOUNT $(echo $AZURE_STORAGE_ACCOUNT)
STORAGE_RESPONSE=$(echo az storage account show-connection-string --name $AZURE_STORAGE_ACCOUNT)
AZURE_STORAGE_CONNECTION_STRING=$($STORAGE_RESPONSE | jq -r '.connectionString')

BLOB_URI_ROOT="$BLOB_URI_PROTOCOL://$AZURE_STORAGE_ACCOUNT.blob.$BLOB_URI_ENDPOINT/$AZUREML_DEFAULT_CONTAINER"
# </get_storage_details>

# <upload_code>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/score -s endpoints/batch/mnist/code/  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_code>

# <create_code>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
  \"properties\": {
    \"description\": \"Score code\",
    \"codeUri\": \"$BLOB_URI_ROOT/score\"
  }
}")
# </create_code>

# <upload_model>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/model -s endpoints/batch/mnist/model  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_model>

# <create_model>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"modelUri\":\"azureml://subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/workspaces/$WORKSPACE/datastores/$AZUREML_DEFAULT_DATASTORE/paths/model\"
    }
}")
# </create_model>

# <read_condafile>
CONDA_FILE=$(cat endpoints/batch/mnist/environment/conda.json | sed 's/"/\\"/g')
# </read_condafile>
# <create_environment>
ENV_VERSION=$RANDOM
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"condaFile\": $(echo \"$CONDA_FILE\"),
        \"image\": \"mcr.microsoft.com/azureml/openmpi3.1.2-ubuntu18.04:latest\"
    }
}")
# </create_environment>

# <create_compute>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"computeType\": \"AmlCompute\",
        \"properties\": {
            \"osType\": \"Linux\",
            \"vmSize\": \"STANDARD_D2_V2\",
            \"scaleSettings\": {
                \"maxNodeCount\": 5,
                \"minNodeCount\": 0
            },
            \"remoteLoginPortPublicAccess\": \"NotSpecified\"
        },
    },
    \"location\": \"$LOCATION\"
}")
# </create_compute>

#<create_endpoint>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\"
    },
    \"location\": \"$LOCATION\"
}")
#</create_endpoint>

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

# <create_deployment>
DEPLOYMENT_NAME="nonmlflowedp"
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME/deployments/$DEPLOYMENT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"location\": \"$LOCATION\",
    \"properties\": {        
        \"model\": {
            \"referenceType\": \"Id\",
            \"assetId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1\"
        },
        \"codeConfiguration\": {
            \"codeId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1\",
            \"scoringScript\": \"digit_identification.py\"
        },
        \"environmentId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION\",
        \"compute\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster\",
        \"resources\": {
            \"instanceCount\": 1
        },
        \"maxConcurrencyPerInstance\": \"4\",
        \"retrySettings\": {
            \"maxRetries\": 3,
            \"timeout\": \"PT30S\"
        },
        \"errorThreshold\": \"10\",
        \"loggingLevel\": \"info\",
        \"miniBatchSize\": \"5\",
    }
}")
#</create_deployment>
operation_id=$(echo $response | jq -r '.properties.properties.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

#<set_endpoint_defaults>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\",
        \"defaults\": {
            \"deploymentName\": \"$DEPLOYMENT_NAME\"
        }
    },
    \"location\": \"$LOCATION\"
}")

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN
#</set_endpoint_defaults>

# <get_endpoint>
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN")

# <score_endpoint>
SCORING_URI=$(echo $response | jq -r '.properties.scoringUri')
# </get_endpoint>

# <get_access_token>
SCORING_TOKEN=$(az account get-access-token --resource https://ml.azure.com --query accessToken -o tsv)
# </get_access_token>

# <score_endpoint_with_data_in_cloud>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DataUrl\",
            \"Path\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
        }
    }
}")

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# </score_endpoint_with_data_in_cloud>

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>

#<create_dataset>
DATASET_NAME="mnist"
DATASET_VERSION=$RANDOM

response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datasets/$DATASET_NAME/versions/$DATASET_VERSION?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"paths\": [
            {
                \"folder\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
            }
        ]
    }
}")
#</create_dataset>

#<unique_output>
export OUTPUT_FILE_NAME=predictions_`echo $RANDOM`.csv
#</unique_output>

# <score_endpoint_with_dataset>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DatasetVersion\",
            \"datasetName\": \"$DATASET_NAME\",
            \"datasetVersion\": \"$DATASET_VERSION\"
        },
        \"outputDataset\": {
            \"datastoreId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores/workspaceblobstore\",
            \"path\": \"$ENDPOINT_NAME\"
        },
        \"outputFileName\": \"$OUTPUT_FILE_NAME\"
    }
}")
# </score_endpoint_with_dataset>

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>
# </score_endpoint>

# <delete_endpoint>
curl --location --request DELETE "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" || true
# </delete_endpoint>

```

### Set the default batch deployment under the endpoint

There's only one default batch deployment under one endpoint, which will be used when invoke to run batch scoring job.

```rest-api
# <create_variables>
SUBSCRIPTION_ID=$(az account show --query id | tr -d '\r"')
RESOURCE_GROUP=$(az group show --query name | tr -d '\r"')
WORKSPACE=$(az configure -l | jq -r '.[] | select(.name=="workspace") | .value')

LOCATION=$(az ml workspace show| jq -r '.location')

API_VERSION="2022-05-01"

TOKEN=$(az account get-access-token --query accessToken -o tsv)
#</create_variables>

# <set_endpoint_name>
export ENDPOINT_NAME=endpt-`echo $RANDOM`
# </set_endpoint_name>

echo "Using: SUBSCRIPTION_ID: $SUBSCRIPTION_ID LOCATION: $LOCATION RESOURCE_GROUP: $RESOURCE_GROUP WORKSPACE: $WORKSPACE ENDPOINT_NAME: $ENDPOINT_NAME"

# <define_wait>  
wait_for_completion () {
    operation_id=$1
    access_token=$2
    status="unknown"

    while [[ $status != "Completed" && $status != "Succeeded" && $status != "Failed" && $status != "Canceled" ]]
    do
        echo "Getting operation status from: $operation_id"
        operation_result=$(curl --location --request GET $operation_id --header "Authorization: Bearer $access_token")
        # TODO error handling here
        status=$(echo $operation_result | jq -r '.status')
        if [[ -z $status || $status == "null" ]]
        then
            status=$(echo $operation_result | jq -r '.properties.status')
        fi

        # Fail early if job submission failed and there is nothing to poll on
        if [[ -z $status || $status == "null" ]]
        then
            echo "No status found on operation, setting to failed."
            status="Failed"
        fi

        echo "Current operation status: $status"
        sleep 10
    done

    if [[ $status == "Failed" ]]
    then
        error=$(echo $operation_result | jq -r '.error')
        echo "Error: $error"
    fi
}
# </define_wait>  

# <get_storage_details>
# Get values for storage account
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores?api-version=$API_VERSION&isDefault=true" \
--header "Authorization: Bearer $TOKEN")
DATASTORE_PATH=$(echo $response | jq -r '.value[0].id')
BLOB_URI_PROTOCOL=$(echo $response | jq -r '.value[0].properties.protocol')
BLOB_URI_ENDPOINT=$(echo $response | jq -r '.value[0].properties.endpoint')
AZUREML_DEFAULT_DATASTORE=$(echo $response | jq -r '.value[0].name')
AZUREML_DEFAULT_CONTAINER=$(echo $response | jq -r '.value[0].properties.containerName')
AZURE_STORAGE_ACCOUNT=$(echo $response | jq -r '.value[0].properties.accountName')
export AZURE_STORAGE_ACCOUNT $(echo $AZURE_STORAGE_ACCOUNT)
STORAGE_RESPONSE=$(echo az storage account show-connection-string --name $AZURE_STORAGE_ACCOUNT)
AZURE_STORAGE_CONNECTION_STRING=$($STORAGE_RESPONSE | jq -r '.connectionString')

BLOB_URI_ROOT="$BLOB_URI_PROTOCOL://$AZURE_STORAGE_ACCOUNT.blob.$BLOB_URI_ENDPOINT/$AZUREML_DEFAULT_CONTAINER"
# </get_storage_details>

# <upload_code>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/score -s endpoints/batch/mnist/code/  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_code>

# <create_code>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
  \"properties\": {
    \"description\": \"Score code\",
    \"codeUri\": \"$BLOB_URI_ROOT/score\"
  }
}")
# </create_code>

# <upload_model>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/model -s endpoints/batch/mnist/model  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_model>

# <create_model>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"modelUri\":\"azureml://subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/workspaces/$WORKSPACE/datastores/$AZUREML_DEFAULT_DATASTORE/paths/model\"
    }
}")
# </create_model>

# <read_condafile>
CONDA_FILE=$(cat endpoints/batch/mnist/environment/conda.json | sed 's/"/\\"/g')
# </read_condafile>
# <create_environment>
ENV_VERSION=$RANDOM
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"condaFile\": $(echo \"$CONDA_FILE\"),
        \"image\": \"mcr.microsoft.com/azureml/openmpi3.1.2-ubuntu18.04:latest\"
    }
}")
# </create_environment>

# <create_compute>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"computeType\": \"AmlCompute\",
        \"properties\": {
            \"osType\": \"Linux\",
            \"vmSize\": \"STANDARD_D2_V2\",
            \"scaleSettings\": {
                \"maxNodeCount\": 5,
                \"minNodeCount\": 0
            },
            \"remoteLoginPortPublicAccess\": \"NotSpecified\"
        },
    },
    \"location\": \"$LOCATION\"
}")
# </create_compute>

#<create_endpoint>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\"
    },
    \"location\": \"$LOCATION\"
}")
#</create_endpoint>

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

# <create_deployment>
DEPLOYMENT_NAME="nonmlflowedp"
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME/deployments/$DEPLOYMENT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"location\": \"$LOCATION\",
    \"properties\": {        
        \"model\": {
            \"referenceType\": \"Id\",
            \"assetId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1\"
        },
        \"codeConfiguration\": {
            \"codeId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1\",
            \"scoringScript\": \"digit_identification.py\"
        },
        \"environmentId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION\",
        \"compute\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster\",
        \"resources\": {
            \"instanceCount\": 1
        },
        \"maxConcurrencyPerInstance\": \"4\",
        \"retrySettings\": {
            \"maxRetries\": 3,
            \"timeout\": \"PT30S\"
        },
        \"errorThreshold\": \"10\",
        \"loggingLevel\": \"info\",
        \"miniBatchSize\": \"5\",
    }
}")
#</create_deployment>
operation_id=$(echo $response | jq -r '.properties.properties.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

#<set_endpoint_defaults>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\",
        \"defaults\": {
            \"deploymentName\": \"$DEPLOYMENT_NAME\"
        }
    },
    \"location\": \"$LOCATION\"
}")

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN
#</set_endpoint_defaults>

# <get_endpoint>
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN")

# <score_endpoint>
SCORING_URI=$(echo $response | jq -r '.properties.scoringUri')
# </get_endpoint>

# <get_access_token>
SCORING_TOKEN=$(az account get-access-token --resource https://ml.azure.com --query accessToken -o tsv)
# </get_access_token>

# <score_endpoint_with_data_in_cloud>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DataUrl\",
            \"Path\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
        }
    }
}")

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# </score_endpoint_with_data_in_cloud>

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>

#<create_dataset>
DATASET_NAME="mnist"
DATASET_VERSION=$RANDOM

response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datasets/$DATASET_NAME/versions/$DATASET_VERSION?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"paths\": [
            {
                \"folder\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
            }
        ]
    }
}")
#</create_dataset>

#<unique_output>
export OUTPUT_FILE_NAME=predictions_`echo $RANDOM`.csv
#</unique_output>

# <score_endpoint_with_dataset>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DatasetVersion\",
            \"datasetName\": \"$DATASET_NAME\",
            \"datasetVersion\": \"$DATASET_VERSION\"
        },
        \"outputDataset\": {
            \"datastoreId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores/workspaceblobstore\",
            \"path\": \"$ENDPOINT_NAME\"
        },
        \"outputFileName\": \"$OUTPUT_FILE_NAME\"
    }
}")
# </score_endpoint_with_dataset>

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>
# </score_endpoint>

# <delete_endpoint>
curl --location --request DELETE "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" || true
# </delete_endpoint>

```

## Run batch scoring

Invoking a batch endpoint triggers a batch scoring job. A job `id` is returned in the response, and can be used to track the batch scoring progress. In the following snippets, `jq` is used to get the job `id`.

### Invoke the batch endpoint to start a batch scoring job

#### Getting the Scoring URI and access token

Get the scoring uri and access token to invoke the batch endpoint. First get the scoring uri:

```rest-api
# <create_variables>
SUBSCRIPTION_ID=$(az account show --query id | tr -d '\r"')
RESOURCE_GROUP=$(az group show --query name | tr -d '\r"')
WORKSPACE=$(az configure -l | jq -r '.[] | select(.name=="workspace") | .value')

LOCATION=$(az ml workspace show| jq -r '.location')

API_VERSION="2022-05-01"

TOKEN=$(az account get-access-token --query accessToken -o tsv)
#</create_variables>

# <set_endpoint_name>
export ENDPOINT_NAME=endpt-`echo $RANDOM`
# </set_endpoint_name>

echo "Using: SUBSCRIPTION_ID: $SUBSCRIPTION_ID LOCATION: $LOCATION RESOURCE_GROUP: $RESOURCE_GROUP WORKSPACE: $WORKSPACE ENDPOINT_NAME: $ENDPOINT_NAME"

# <define_wait>  
wait_for_completion () {
    operation_id=$1
    access_token=$2
    status="unknown"

    while [[ $status != "Completed" && $status != "Succeeded" && $status != "Failed" && $status != "Canceled" ]]
    do
        echo "Getting operation status from: $operation_id"
        operation_result=$(curl --location --request GET $operation_id --header "Authorization: Bearer $access_token")
        # TODO error handling here
        status=$(echo $operation_result | jq -r '.status')
        if [[ -z $status || $status == "null" ]]
        then
            status=$(echo $operation_result | jq -r '.properties.status')
        fi

        # Fail early if job submission failed and there is nothing to poll on
        if [[ -z $status || $status == "null" ]]
        then
            echo "No status found on operation, setting to failed."
            status="Failed"
        fi

        echo "Current operation status: $status"
        sleep 10
    done

    if [[ $status == "Failed" ]]
    then
        error=$(echo $operation_result | jq -r '.error')
        echo "Error: $error"
    fi
}
# </define_wait>  

# <get_storage_details>
# Get values for storage account
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores?api-version=$API_VERSION&isDefault=true" \
--header "Authorization: Bearer $TOKEN")
DATASTORE_PATH=$(echo $response | jq -r '.value[0].id')
BLOB_URI_PROTOCOL=$(echo $response | jq -r '.value[0].properties.protocol')
BLOB_URI_ENDPOINT=$(echo $response | jq -r '.value[0].properties.endpoint')
AZUREML_DEFAULT_DATASTORE=$(echo $response | jq -r '.value[0].name')
AZUREML_DEFAULT_CONTAINER=$(echo $response | jq -r '.value[0].properties.containerName')
AZURE_STORAGE_ACCOUNT=$(echo $response | jq -r '.value[0].properties.accountName')
export AZURE_STORAGE_ACCOUNT $(echo $AZURE_STORAGE_ACCOUNT)
STORAGE_RESPONSE=$(echo az storage account show-connection-string --name $AZURE_STORAGE_ACCOUNT)
AZURE_STORAGE_CONNECTION_STRING=$($STORAGE_RESPONSE | jq -r '.connectionString')

BLOB_URI_ROOT="$BLOB_URI_PROTOCOL://$AZURE_STORAGE_ACCOUNT.blob.$BLOB_URI_ENDPOINT/$AZUREML_DEFAULT_CONTAINER"
# </get_storage_details>

# <upload_code>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/score -s endpoints/batch/mnist/code/  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_code>

# <create_code>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
  \"properties\": {
    \"description\": \"Score code\",
    \"codeUri\": \"$BLOB_URI_ROOT/score\"
  }
}")
# </create_code>

# <upload_model>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/model -s endpoints/batch/mnist/model  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_model>

# <create_model>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"modelUri\":\"azureml://subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/workspaces/$WORKSPACE/datastores/$AZUREML_DEFAULT_DATASTORE/paths/model\"
    }
}")
# </create_model>

# <read_condafile>
CONDA_FILE=$(cat endpoints/batch/mnist/environment/conda.json | sed 's/"/\\"/g')
# </read_condafile>
# <create_environment>
ENV_VERSION=$RANDOM
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"condaFile\": $(echo \"$CONDA_FILE\"),
        \"image\": \"mcr.microsoft.com/azureml/openmpi3.1.2-ubuntu18.04:latest\"
    }
}")
# </create_environment>

# <create_compute>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"computeType\": \"AmlCompute\",
        \"properties\": {
            \"osType\": \"Linux\",
            \"vmSize\": \"STANDARD_D2_V2\",
            \"scaleSettings\": {
                \"maxNodeCount\": 5,
                \"minNodeCount\": 0
            },
            \"remoteLoginPortPublicAccess\": \"NotSpecified\"
        },
    },
    \"location\": \"$LOCATION\"
}")
# </create_compute>

#<create_endpoint>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\"
    },
    \"location\": \"$LOCATION\"
}")
#</create_endpoint>

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

# <create_deployment>
DEPLOYMENT_NAME="nonmlflowedp"
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME/deployments/$DEPLOYMENT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"location\": \"$LOCATION\",
    \"properties\": {        
        \"model\": {
            \"referenceType\": \"Id\",
            \"assetId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1\"
        },
        \"codeConfiguration\": {
            \"codeId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1\",
            \"scoringScript\": \"digit_identification.py\"
        },
        \"environmentId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION\",
        \"compute\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster\",
        \"resources\": {
            \"instanceCount\": 1
        },
        \"maxConcurrencyPerInstance\": \"4\",
        \"retrySettings\": {
            \"maxRetries\": 3,
            \"timeout\": \"PT30S\"
        },
        \"errorThreshold\": \"10\",
        \"loggingLevel\": \"info\",
        \"miniBatchSize\": \"5\",
    }
}")
#</create_deployment>
operation_id=$(echo $response | jq -r '.properties.properties.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

#<set_endpoint_defaults>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\",
        \"defaults\": {
            \"deploymentName\": \"$DEPLOYMENT_NAME\"
        }
    },
    \"location\": \"$LOCATION\"
}")

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN
#</set_endpoint_defaults>

# <get_endpoint>
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN")

# <score_endpoint>
SCORING_URI=$(echo $response | jq -r '.properties.scoringUri')
# </get_endpoint>

# <get_access_token>
SCORING_TOKEN=$(az account get-access-token --resource https://ml.azure.com --query accessToken -o tsv)
# </get_access_token>

# <score_endpoint_with_data_in_cloud>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DataUrl\",
            \"Path\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
        }
    }
}")

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# </score_endpoint_with_data_in_cloud>

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>

#<create_dataset>
DATASET_NAME="mnist"
DATASET_VERSION=$RANDOM

response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datasets/$DATASET_NAME/versions/$DATASET_VERSION?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"paths\": [
            {
                \"folder\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
            }
        ]
    }
}")
#</create_dataset>

#<unique_output>
export OUTPUT_FILE_NAME=predictions_`echo $RANDOM`.csv
#</unique_output>

# <score_endpoint_with_dataset>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DatasetVersion\",
            \"datasetName\": \"$DATASET_NAME\",
            \"datasetVersion\": \"$DATASET_VERSION\"
        },
        \"outputDataset\": {
            \"datastoreId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores/workspaceblobstore\",
            \"path\": \"$ENDPOINT_NAME\"
        },
        \"outputFileName\": \"$OUTPUT_FILE_NAME\"
    }
}")
# </score_endpoint_with_dataset>

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>
# </score_endpoint>

# <delete_endpoint>
curl --location --request DELETE "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" || true
# </delete_endpoint>

```

Get the batch endpoint access token:

```rest-api
# <create_variables>
SUBSCRIPTION_ID=$(az account show --query id | tr -d '\r"')
RESOURCE_GROUP=$(az group show --query name | tr -d '\r"')
WORKSPACE=$(az configure -l | jq -r '.[] | select(.name=="workspace") | .value')

LOCATION=$(az ml workspace show| jq -r '.location')

API_VERSION="2022-05-01"

TOKEN=$(az account get-access-token --query accessToken -o tsv)
#</create_variables>

# <set_endpoint_name>
export ENDPOINT_NAME=endpt-`echo $RANDOM`
# </set_endpoint_name>

echo "Using: SUBSCRIPTION_ID: $SUBSCRIPTION_ID LOCATION: $LOCATION RESOURCE_GROUP: $RESOURCE_GROUP WORKSPACE: $WORKSPACE ENDPOINT_NAME: $ENDPOINT_NAME"

# <define_wait>  
wait_for_completion () {
    operation_id=$1
    access_token=$2
    status="unknown"

    while [[ $status != "Completed" && $status != "Succeeded" && $status != "Failed" && $status != "Canceled" ]]
    do
        echo "Getting operation status from: $operation_id"
        operation_result=$(curl --location --request GET $operation_id --header "Authorization: Bearer $access_token")
        # TODO error handling here
        status=$(echo $operation_result | jq -r '.status')
        if [[ -z $status || $status == "null" ]]
        then
            status=$(echo $operation_result | jq -r '.properties.status')
        fi

        # Fail early if job submission failed and there is nothing to poll on
        if [[ -z $status || $status == "null" ]]
        then
            echo "No status found on operation, setting to failed."
            status="Failed"
        fi

        echo "Current operation status: $status"
        sleep 10
    done

    if [[ $status == "Failed" ]]
    then
        error=$(echo $operation_result | jq -r '.error')
        echo "Error: $error"
    fi
}
# </define_wait>  

# <get_storage_details>
# Get values for storage account
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores?api-version=$API_VERSION&isDefault=true" \
--header "Authorization: Bearer $TOKEN")
DATASTORE_PATH=$(echo $response | jq -r '.value[0].id')
BLOB_URI_PROTOCOL=$(echo $response | jq -r '.value[0].properties.protocol')
BLOB_URI_ENDPOINT=$(echo $response | jq -r '.value[0].properties.endpoint')
AZUREML_DEFAULT_DATASTORE=$(echo $response | jq -r '.value[0].name')
AZUREML_DEFAULT_CONTAINER=$(echo $response | jq -r '.value[0].properties.containerName')
AZURE_STORAGE_ACCOUNT=$(echo $response | jq -r '.value[0].properties.accountName')
export AZURE_STORAGE_ACCOUNT $(echo $AZURE_STORAGE_ACCOUNT)
STORAGE_RESPONSE=$(echo az storage account show-connection-string --name $AZURE_STORAGE_ACCOUNT)
AZURE_STORAGE_CONNECTION_STRING=$($STORAGE_RESPONSE | jq -r '.connectionString')

BLOB_URI_ROOT="$BLOB_URI_PROTOCOL://$AZURE_STORAGE_ACCOUNT.blob.$BLOB_URI_ENDPOINT/$AZUREML_DEFAULT_CONTAINER"
# </get_storage_details>

# <upload_code>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/score -s endpoints/batch/mnist/code/  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_code>

# <create_code>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
  \"properties\": {
    \"description\": \"Score code\",
    \"codeUri\": \"$BLOB_URI_ROOT/score\"
  }
}")
# </create_code>

# <upload_model>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/model -s endpoints/batch/mnist/model  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_model>

# <create_model>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"modelUri\":\"azureml://subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/workspaces/$WORKSPACE/datastores/$AZUREML_DEFAULT_DATASTORE/paths/model\"
    }
}")
# </create_model>

# <read_condafile>
CONDA_FILE=$(cat endpoints/batch/mnist/environment/conda.json | sed 's/"/\\"/g')
# </read_condafile>
# <create_environment>
ENV_VERSION=$RANDOM
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"condaFile\": $(echo \"$CONDA_FILE\"),
        \"image\": \"mcr.microsoft.com/azureml/openmpi3.1.2-ubuntu18.04:latest\"
    }
}")
# </create_environment>

# <create_compute>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"computeType\": \"AmlCompute\",
        \"properties\": {
            \"osType\": \"Linux\",
            \"vmSize\": \"STANDARD_D2_V2\",
            \"scaleSettings\": {
                \"maxNodeCount\": 5,
                \"minNodeCount\": 0
            },
            \"remoteLoginPortPublicAccess\": \"NotSpecified\"
        },
    },
    \"location\": \"$LOCATION\"
}")
# </create_compute>

#<create_endpoint>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\"
    },
    \"location\": \"$LOCATION\"
}")
#</create_endpoint>

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

# <create_deployment>
DEPLOYMENT_NAME="nonmlflowedp"
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME/deployments/$DEPLOYMENT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"location\": \"$LOCATION\",
    \"properties\": {        
        \"model\": {
            \"referenceType\": \"Id\",
            \"assetId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1\"
        },
        \"codeConfiguration\": {
            \"codeId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1\",
            \"scoringScript\": \"digit_identification.py\"
        },
        \"environmentId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION\",
        \"compute\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster\",
        \"resources\": {
            \"instanceCount\": 1
        },
        \"maxConcurrencyPerInstance\": \"4\",
        \"retrySettings\": {
            \"maxRetries\": 3,
            \"timeout\": \"PT30S\"
        },
        \"errorThreshold\": \"10\",
        \"loggingLevel\": \"info\",
        \"miniBatchSize\": \"5\",
    }
}")
#</create_deployment>
operation_id=$(echo $response | jq -r '.properties.properties.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

#<set_endpoint_defaults>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\",
        \"defaults\": {
            \"deploymentName\": \"$DEPLOYMENT_NAME\"
        }
    },
    \"location\": \"$LOCATION\"
}")

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN
#</set_endpoint_defaults>

# <get_endpoint>
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN")

# <score_endpoint>
SCORING_URI=$(echo $response | jq -r '.properties.scoringUri')
# </get_endpoint>

# <get_access_token>
SCORING_TOKEN=$(az account get-access-token --resource https://ml.azure.com --query accessToken -o tsv)
# </get_access_token>

# <score_endpoint_with_data_in_cloud>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DataUrl\",
            \"Path\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
        }
    }
}")

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# </score_endpoint_with_data_in_cloud>

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>

#<create_dataset>
DATASET_NAME="mnist"
DATASET_VERSION=$RANDOM

response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datasets/$DATASET_NAME/versions/$DATASET_VERSION?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"paths\": [
            {
                \"folder\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
            }
        ]
    }
}")
#</create_dataset>

#<unique_output>
export OUTPUT_FILE_NAME=predictions_`echo $RANDOM`.csv
#</unique_output>

# <score_endpoint_with_dataset>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DatasetVersion\",
            \"datasetName\": \"$DATASET_NAME\",
            \"datasetVersion\": \"$DATASET_VERSION\"
        },
        \"outputDataset\": {
            \"datastoreId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores/workspaceblobstore\",
            \"path\": \"$ENDPOINT_NAME\"
        },
        \"outputFileName\": \"$OUTPUT_FILE_NAME\"
    }
}")
# </score_endpoint_with_dataset>

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>
# </score_endpoint>

# <delete_endpoint>
curl --location --request DELETE "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" || true
# </delete_endpoint>

```

#### Invoke the batch endpoint with different input options

It's time to invoke the batch endpoint to start a batch scoring job. If your data is a folder (potentially with multiple files) publicly available from the web, you can use the following snippet:

```rest-api
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
    	\"InputData\": {
    		\"mnistinput\": {
    			\"JobInputType\" : \"UriFolder\",
    			\"Uri\":  \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
    		}
        }
    }
}")

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})
```

Now, let's look at other options for invoking the batch endpoint. When it comes to input data, there are multiple scenarios you can choose from, depending on the input type (whether you are specifying a folder or a single file), and the URI type (whether you are using a path on Azure Machine Learning registered datastore, a reference to Azure Machine Learning registered V2 data asset, or a public URI).

- An `InputData` property has `JobInputType` and `Uri` keys. When you are specifying a single file, use `"JobInputType": "UriFile"`, and when you are specifying a folder, use `'JobInputType": "UriFolder"`.

- When the file or folder is on Azure ML registered datastore, the syntax for the `Uri` is  `azureml://datastores/<datastore-name>/paths/<path-on-datastore>` for folder, and `azureml://datastores/<datastore-name>/paths/<path-on-datastore>/<file-name>` for a specific file. You can also use the longer form to represent the same path, such as `azureml://subscriptions/<subscription-id>/resourceGroups/<resource-group-name>/workspaces/<workspace-name>/datastores/<datastore-name>/paths/<path-on-datastore>/`.

- When the file or folder is registered as V2 data asset as `uri_folder` or `uri_file`, the syntax for the `Uri` is `\"azureml://locations/<location-name>/workspaces/<workspace-name>/data/<data-name>/versions/<data-version>"` (Asset ID form) or `\"/subscriptions/<subscription-id>/resourcegroups/<resource-group-name>/providers/Microsoft.MachineLearningServices/workspaces/<workspace-name>/data/<data-name>/versions/<data-version>\"` (ARM ID form).

- When the file or folder is a publicly accessible path, the syntax for the URI is `https://<public-path>` for folder, `https://<public-path>/<file-name>` for a specific file.

> [!NOTE]
> For more information about data URI, see [Azure Machine Learning data reference URI](reference-yaml-core-syntax.md#azure-ml-data-reference-uri).

Below are some examples using different types of input data.

- If your data is a folder on the Azure ML registered datastore, you can either:

    - Use the short form to represent the URI:

    ```rest-api
    response=$(curl --location --request POST $SCORING_URI \
    --header "Authorization: Bearer $SCORING_TOKEN" \
    --header "Content-Type: application/json" \
    --data-raw "{
        \"properties\": {
            \"InputData\": {
                \"mnistInput\": {
                    \"JobInputType\" : \"UriFolder\",
                    \"Uri": \"azureml://datastores/workspaceblobstore/paths/$ENDPOINT_NAME/mnist\"
                }
            }
        }
    }")
    
    JOB_ID=$(echo $response | jq -r '.id')
    JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})
    ```

    - Or use the long form for the same URI:

    ```rest-api
    response=$(curl --location --request POST $SCORING_URI \
    --header "Authorization: Bearer $SCORING_TOKEN" \
    --header "Content-Type: application/json" \
    --data-raw "{
        \"properties\": {
        	\"InputData\": {
        		\"mnistinput\": {
        			\"JobInputType\" : \"UriFolder\",
        			\"Uri\": \"azureml://subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/workspaces/$WORKSPACE/datastores/workspaceblobstore/paths/$ENDPOINT_NAME/mnist\"
        		}
            }
        }
    }")
    
    JOB_ID=$(echo $response | jq -r '.id')
    JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})
    ```

- If you want to manage your data as Azure ML registered V2 data asset as `uri_folder`, you can follow the two steps below:

    1. Create the V2 data asset:

    ```rest-api
    DATA_NAME="mnist"
    DATA_VERSION=$RANDOM
    
    response=$(curl --location --request PUT https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/data/$DATA_NAME/versions/$DATA_VERSION?api-version=$API_VERSION \
    --header "Content-Type: application/json" \
    --header "Authorization: Bearer $TOKEN" \
    --data-raw "{
        \"properties\": {
            \"dataType\": \"uri_folder\",
      \"dataUri\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\",
      \"description\": \"Mnist data asset\"
        }
    }")
    ```

    2. Reference the data asset in the batch scoring job:

    ```rest-api
    response=$(curl --location --request POST $SCORING_URI \
    --header "Authorization: Bearer $SCORING_TOKEN" \
    --header "Content-Type: application/json" \
    --data-raw "{
        \"properties\": {
            \"InputData\": {
                \"mnistInput\": {
                    \"JobInputType\" : \"UriFolder\",
                    \"Uri": \"azureml://locations/$LOCATION_NAME/workspaces/$WORKSPACE_NAME/data/$DATA_NAME/versions/$DATA_VERSION/\"
                }
            }
        }
    }")
    
    JOB_ID=$(echo $response | jq -r '.id')
    JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})
    ```

- If your data is a single file publicly available from the web, you can use the following snippet:

    ```rest-api
    response=$(curl --location --request POST $SCORING_URI \
    --header "Authorization: Bearer $SCORING_TOKEN" \
    --header "Content-Type: application/json" \
    --data-raw "{
        \"properties\": {
            \"InputData\": {
                \"mnistInput\": {
                    \"JobInputType\" : \"UriFile\",
                    \"Uri": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist/0.png\"
                }
            }
        }
    }")
    
    JOB_ID=$(echo $response | jq -r '.id')
    JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})
    ```

> [!NOTE]
> We strongly recommend using the latest REST API version for batch scoring.
> - If you want to use local data, you can upload it to Azure Machine Learning registered datastore and use REST API for Cloud data.
> - If you are using existing V1 FileDataset for batch endpoint, we recommend migrating them to V2 data assets and refer to them directly when invoking batch endpoints. Currently only data assets of type `uri_folder` or `uri_file` are supported. Batch endpoints created with GA CLIv2 (2.4.0 and newer) or GA REST API (2022-05-01 and newer) will not support V1 Dataset.
> - You can also extract the URI or path on datastore extracted from V1 FileDataset by using `az ml dataset show` command with `--query` parameter and use that information for invoke.
> - While Batch endpoints created with earlier APIs will continue to support V1 FileDataset, we will be adding further V2 data assets support with the latest API versions for even more usability and flexibility. For more information on V2 data assets, see [Work with data using SDK v2](how-to-read-write-data-v2.md). For more information on the new V2 experience, see [What is v2](concept-v2.md).

#### Configure the output location and overwrite settings

The batch scoring results are by default stored in the workspace's default blob store within a folder named by job name (a system-generated GUID). You can configure where to store the scoring outputs when you invoke the batch endpoint. Use `OutputData` to configure the output file path on an Azure Machine Learning registered datastore. `OutputData` has `JobOutputType` and `Uri` keys. `UriFile` is the only supported value for `JobOutputType`. The syntax for `Uri` is the same as that of `InputData`, i.e., `azureml://datastores/<datastore-name>/paths/<path-on-datastore>/<file-name>`.

Following is the example snippet for configuring the output location for the batch scoring results.

```rest-api
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"InputData\":
        {
            \"mnistInput\": {
                \"JobInputType\" : \"UriFolder\",
                \"Uri": \"azureml://datastores/workspaceblobstore/paths/$ENDPOINT_NAME/mnist\"
            }
        },
        \"OutputData\":
        {
            \"mnistOutput\": {
                \"JobOutputType\": \"UriFile\",
                \"Uri\": \"azureml://datastores/workspaceblobstore/paths/$ENDPOINT_NAME/mnistOutput/$OUTPUT_FILE_NAME\"
            }
        }
    }
}")

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})
```

> [!IMPORTANT]
> You must use a unique output location. If the output file exists, the batch scoring job will fail. 

### Check the batch scoring job

Batch scoring jobs usually take some time to process the entire set of inputs. Monitor the job status and check the results after it's completed:

> [!TIP]
> The example invokes the default deployment of the batch endpoint. To invoke a non-default deployment, use the `azureml-model-deployment` HTTP header and set the value to the deployment name. For example, using a parameter of `--header "azureml-model-deployment: $DEPLOYMENT_NAME"` with curl.

```rest-api
# <create_variables>
SUBSCRIPTION_ID=$(az account show --query id | tr -d '\r"')
RESOURCE_GROUP=$(az group show --query name | tr -d '\r"')
WORKSPACE=$(az configure -l | jq -r '.[] | select(.name=="workspace") | .value')

LOCATION=$(az ml workspace show| jq -r '.location')

API_VERSION="2022-05-01"

TOKEN=$(az account get-access-token --query accessToken -o tsv)
#</create_variables>

# <set_endpoint_name>
export ENDPOINT_NAME=endpt-`echo $RANDOM`
# </set_endpoint_name>

echo "Using: SUBSCRIPTION_ID: $SUBSCRIPTION_ID LOCATION: $LOCATION RESOURCE_GROUP: $RESOURCE_GROUP WORKSPACE: $WORKSPACE ENDPOINT_NAME: $ENDPOINT_NAME"

# <define_wait>  
wait_for_completion () {
    operation_id=$1
    access_token=$2
    status="unknown"

    while [[ $status != "Completed" && $status != "Succeeded" && $status != "Failed" && $status != "Canceled" ]]
    do
        echo "Getting operation status from: $operation_id"
        operation_result=$(curl --location --request GET $operation_id --header "Authorization: Bearer $access_token")
        # TODO error handling here
        status=$(echo $operation_result | jq -r '.status')
        if [[ -z $status || $status == "null" ]]
        then
            status=$(echo $operation_result | jq -r '.properties.status')
        fi

        # Fail early if job submission failed and there is nothing to poll on
        if [[ -z $status || $status == "null" ]]
        then
            echo "No status found on operation, setting to failed."
            status="Failed"
        fi

        echo "Current operation status: $status"
        sleep 10
    done

    if [[ $status == "Failed" ]]
    then
        error=$(echo $operation_result | jq -r '.error')
        echo "Error: $error"
    fi
}
# </define_wait>  

# <get_storage_details>
# Get values for storage account
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores?api-version=$API_VERSION&isDefault=true" \
--header "Authorization: Bearer $TOKEN")
DATASTORE_PATH=$(echo $response | jq -r '.value[0].id')
BLOB_URI_PROTOCOL=$(echo $response | jq -r '.value[0].properties.protocol')
BLOB_URI_ENDPOINT=$(echo $response | jq -r '.value[0].properties.endpoint')
AZUREML_DEFAULT_DATASTORE=$(echo $response | jq -r '.value[0].name')
AZUREML_DEFAULT_CONTAINER=$(echo $response | jq -r '.value[0].properties.containerName')
AZURE_STORAGE_ACCOUNT=$(echo $response | jq -r '.value[0].properties.accountName')
export AZURE_STORAGE_ACCOUNT $(echo $AZURE_STORAGE_ACCOUNT)
STORAGE_RESPONSE=$(echo az storage account show-connection-string --name $AZURE_STORAGE_ACCOUNT)
AZURE_STORAGE_CONNECTION_STRING=$($STORAGE_RESPONSE | jq -r '.connectionString')

BLOB_URI_ROOT="$BLOB_URI_PROTOCOL://$AZURE_STORAGE_ACCOUNT.blob.$BLOB_URI_ENDPOINT/$AZUREML_DEFAULT_CONTAINER"
# </get_storage_details>

# <upload_code>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/score -s endpoints/batch/mnist/code/  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_code>

# <create_code>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
  \"properties\": {
    \"description\": \"Score code\",
    \"codeUri\": \"$BLOB_URI_ROOT/score\"
  }
}")
# </create_code>

# <upload_model>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/model -s endpoints/batch/mnist/model  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_model>

# <create_model>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"modelUri\":\"azureml://subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/workspaces/$WORKSPACE/datastores/$AZUREML_DEFAULT_DATASTORE/paths/model\"
    }
}")
# </create_model>

# <read_condafile>
CONDA_FILE=$(cat endpoints/batch/mnist/environment/conda.json | sed 's/"/\\"/g')
# </read_condafile>
# <create_environment>
ENV_VERSION=$RANDOM
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"condaFile\": $(echo \"$CONDA_FILE\"),
        \"image\": \"mcr.microsoft.com/azureml/openmpi3.1.2-ubuntu18.04:latest\"
    }
}")
# </create_environment>

# <create_compute>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"computeType\": \"AmlCompute\",
        \"properties\": {
            \"osType\": \"Linux\",
            \"vmSize\": \"STANDARD_D2_V2\",
            \"scaleSettings\": {
                \"maxNodeCount\": 5,
                \"minNodeCount\": 0
            },
            \"remoteLoginPortPublicAccess\": \"NotSpecified\"
        },
    },
    \"location\": \"$LOCATION\"
}")
# </create_compute>

#<create_endpoint>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\"
    },
    \"location\": \"$LOCATION\"
}")
#</create_endpoint>

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

# <create_deployment>
DEPLOYMENT_NAME="nonmlflowedp"
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME/deployments/$DEPLOYMENT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"location\": \"$LOCATION\",
    \"properties\": {        
        \"model\": {
            \"referenceType\": \"Id\",
            \"assetId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1\"
        },
        \"codeConfiguration\": {
            \"codeId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1\",
            \"scoringScript\": \"digit_identification.py\"
        },
        \"environmentId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION\",
        \"compute\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster\",
        \"resources\": {
            \"instanceCount\": 1
        },
        \"maxConcurrencyPerInstance\": \"4\",
        \"retrySettings\": {
            \"maxRetries\": 3,
            \"timeout\": \"PT30S\"
        },
        \"errorThreshold\": \"10\",
        \"loggingLevel\": \"info\",
        \"miniBatchSize\": \"5\",
    }
}")
#</create_deployment>
operation_id=$(echo $response | jq -r '.properties.properties.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

#<set_endpoint_defaults>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\",
        \"defaults\": {
            \"deploymentName\": \"$DEPLOYMENT_NAME\"
        }
    },
    \"location\": \"$LOCATION\"
}")

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN
#</set_endpoint_defaults>

# <get_endpoint>
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN")

# <score_endpoint>
SCORING_URI=$(echo $response | jq -r '.properties.scoringUri')
# </get_endpoint>

# <get_access_token>
SCORING_TOKEN=$(az account get-access-token --resource https://ml.azure.com --query accessToken -o tsv)
# </get_access_token>

# <score_endpoint_with_data_in_cloud>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DataUrl\",
            \"Path\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
        }
    }
}")

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# </score_endpoint_with_data_in_cloud>

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>

#<create_dataset>
DATASET_NAME="mnist"
DATASET_VERSION=$RANDOM

response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datasets/$DATASET_NAME/versions/$DATASET_VERSION?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"paths\": [
            {
                \"folder\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
            }
        ]
    }
}")
#</create_dataset>

#<unique_output>
export OUTPUT_FILE_NAME=predictions_`echo $RANDOM`.csv
#</unique_output>

# <score_endpoint_with_dataset>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DatasetVersion\",
            \"datasetName\": \"$DATASET_NAME\",
            \"datasetVersion\": \"$DATASET_VERSION\"
        },
        \"outputDataset\": {
            \"datastoreId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores/workspaceblobstore\",
            \"path\": \"$ENDPOINT_NAME\"
        },
        \"outputFileName\": \"$OUTPUT_FILE_NAME\"
    }
}")
# </score_endpoint_with_dataset>

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>
# </score_endpoint>

# <delete_endpoint>
curl --location --request DELETE "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" || true
# </delete_endpoint>

```

### Check batch scoring results

For information on checking the results, see [Check batch scoring results](batch-inference/how-to-use-batch-endpoint.md#check-batch-scoring-results).

## Delete the batch endpoint

If you aren't going use the batch endpoint, you should delete it with the below command (it deletes the batch endpoint and all the underlying deployments):

```rest-api
# <create_variables>
SUBSCRIPTION_ID=$(az account show --query id | tr -d '\r"')
RESOURCE_GROUP=$(az group show --query name | tr -d '\r"')
WORKSPACE=$(az configure -l | jq -r '.[] | select(.name=="workspace") | .value')

LOCATION=$(az ml workspace show| jq -r '.location')

API_VERSION="2022-05-01"

TOKEN=$(az account get-access-token --query accessToken -o tsv)
#</create_variables>

# <set_endpoint_name>
export ENDPOINT_NAME=endpt-`echo $RANDOM`
# </set_endpoint_name>

echo "Using: SUBSCRIPTION_ID: $SUBSCRIPTION_ID LOCATION: $LOCATION RESOURCE_GROUP: $RESOURCE_GROUP WORKSPACE: $WORKSPACE ENDPOINT_NAME: $ENDPOINT_NAME"

# <define_wait>  
wait_for_completion () {
    operation_id=$1
    access_token=$2
    status="unknown"

    while [[ $status != "Completed" && $status != "Succeeded" && $status != "Failed" && $status != "Canceled" ]]
    do
        echo "Getting operation status from: $operation_id"
        operation_result=$(curl --location --request GET $operation_id --header "Authorization: Bearer $access_token")
        # TODO error handling here
        status=$(echo $operation_result | jq -r '.status')
        if [[ -z $status || $status == "null" ]]
        then
            status=$(echo $operation_result | jq -r '.properties.status')
        fi

        # Fail early if job submission failed and there is nothing to poll on
        if [[ -z $status || $status == "null" ]]
        then
            echo "No status found on operation, setting to failed."
            status="Failed"
        fi

        echo "Current operation status: $status"
        sleep 10
    done

    if [[ $status == "Failed" ]]
    then
        error=$(echo $operation_result | jq -r '.error')
        echo "Error: $error"
    fi
}
# </define_wait>  

# <get_storage_details>
# Get values for storage account
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores?api-version=$API_VERSION&isDefault=true" \
--header "Authorization: Bearer $TOKEN")
DATASTORE_PATH=$(echo $response | jq -r '.value[0].id')
BLOB_URI_PROTOCOL=$(echo $response | jq -r '.value[0].properties.protocol')
BLOB_URI_ENDPOINT=$(echo $response | jq -r '.value[0].properties.endpoint')
AZUREML_DEFAULT_DATASTORE=$(echo $response | jq -r '.value[0].name')
AZUREML_DEFAULT_CONTAINER=$(echo $response | jq -r '.value[0].properties.containerName')
AZURE_STORAGE_ACCOUNT=$(echo $response | jq -r '.value[0].properties.accountName')
export AZURE_STORAGE_ACCOUNT $(echo $AZURE_STORAGE_ACCOUNT)
STORAGE_RESPONSE=$(echo az storage account show-connection-string --name $AZURE_STORAGE_ACCOUNT)
AZURE_STORAGE_CONNECTION_STRING=$($STORAGE_RESPONSE | jq -r '.connectionString')

BLOB_URI_ROOT="$BLOB_URI_PROTOCOL://$AZURE_STORAGE_ACCOUNT.blob.$BLOB_URI_ENDPOINT/$AZUREML_DEFAULT_CONTAINER"
# </get_storage_details>

# <upload_code>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/score -s endpoints/batch/mnist/code/  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_code>

# <create_code>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
  \"properties\": {
    \"description\": \"Score code\",
    \"codeUri\": \"$BLOB_URI_ROOT/score\"
  }
}")
# </create_code>

# <upload_model>
az storage blob upload-batch -d $AZUREML_DEFAULT_CONTAINER/model -s endpoints/batch/mnist/model  --connection-string $AZURE_STORAGE_CONNECTION_STRING
# </upload_model>

# <create_model>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"modelUri\":\"azureml://subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/workspaces/$WORKSPACE/datastores/$AZUREML_DEFAULT_DATASTORE/paths/model\"
    }
}")
# </create_model>

# <read_condafile>
CONDA_FILE=$(cat endpoints/batch/mnist/environment/conda.json | sed 's/"/\\"/g')
# </read_condafile>
# <create_environment>
ENV_VERSION=$RANDOM
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"condaFile\": $(echo \"$CONDA_FILE\"),
        \"image\": \"mcr.microsoft.com/azureml/openmpi3.1.2-ubuntu18.04:latest\"
    }
}")
# </create_environment>

# <create_compute>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster?api-version=$API_VERSION" \
--header "Authorization: Bearer $TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\":{
        \"computeType\": \"AmlCompute\",
        \"properties\": {
            \"osType\": \"Linux\",
            \"vmSize\": \"STANDARD_D2_V2\",
            \"scaleSettings\": {
                \"maxNodeCount\": 5,
                \"minNodeCount\": 0
            },
            \"remoteLoginPortPublicAccess\": \"NotSpecified\"
        },
    },
    \"location\": \"$LOCATION\"
}")
# </create_compute>

#<create_endpoint>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\"
    },
    \"location\": \"$LOCATION\"
}")
#</create_endpoint>

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

# <create_deployment>
DEPLOYMENT_NAME="nonmlflowedp"
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME/deployments/$DEPLOYMENT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"location\": \"$LOCATION\",
    \"properties\": {        
        \"model\": {
            \"referenceType\": \"Id\",
            \"assetId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/models/mnist/versions/1\"
        },
        \"codeConfiguration\": {
            \"codeId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/codes/score-mnist/versions/1\",
            \"scoringScript\": \"digit_identification.py\"
        },
        \"environmentId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/environments/mnist-env/versions/$ENV_VERSION\",
        \"compute\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/computes/batch-cluster\",
        \"resources\": {
            \"instanceCount\": 1
        },
        \"maxConcurrencyPerInstance\": \"4\",
        \"retrySettings\": {
            \"maxRetries\": 3,
            \"timeout\": \"PT30S\"
        },
        \"errorThreshold\": \"10\",
        \"loggingLevel\": \"info\",
        \"miniBatchSize\": \"5\",
    }
}")
#</create_deployment>
operation_id=$(echo $response | jq -r '.properties.properties.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN

#<set_endpoint_defaults>
response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"authMode\": \"aadToken\",
        \"defaults\": {
            \"deploymentName\": \"$DEPLOYMENT_NAME\"
        }
    },
    \"location\": \"$LOCATION\"
}")

operation_id=$(echo $response | jq -r '.properties' | jq -r '.properties' | jq -r '.AzureAsyncOperationUri')
wait_for_completion $operation_id $TOKEN
#</set_endpoint_defaults>

# <get_endpoint>
response=$(curl --location --request GET "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN")

# <score_endpoint>
SCORING_URI=$(echo $response | jq -r '.properties.scoringUri')
# </get_endpoint>

# <get_access_token>
SCORING_TOKEN=$(az account get-access-token --resource https://ml.azure.com --query accessToken -o tsv)
# </get_access_token>

# <score_endpoint_with_data_in_cloud>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DataUrl\",
            \"Path\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
        }
    }
}")

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# </score_endpoint_with_data_in_cloud>

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>

#<create_dataset>
DATASET_NAME="mnist"
DATASET_VERSION=$RANDOM

response=$(curl --location --request PUT "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datasets/$DATASET_NAME/versions/$DATASET_VERSION?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" \
--data-raw "{
    \"properties\": {
        \"paths\": [
            {
                \"folder\": \"https://pipelinedata.blob.core.windows.net/sampledata/mnist\"
            }
        ]
    }
}")
#</create_dataset>

#<unique_output>
export OUTPUT_FILE_NAME=predictions_`echo $RANDOM`.csv
#</unique_output>

# <score_endpoint_with_dataset>
response=$(curl --location --request POST $SCORING_URI \
--header "Authorization: Bearer $SCORING_TOKEN" \
--header "Content-Type: application/json" \
--data-raw "{
    \"properties\": {
        \"dataset\": {
            \"dataInputType\": \"DatasetVersion\",
            \"datasetName\": \"$DATASET_NAME\",
            \"datasetVersion\": \"$DATASET_VERSION\"
        },
        \"outputDataset\": {
            \"datastoreId\": \"/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/datastores/workspaceblobstore\",
            \"path\": \"$ENDPOINT_NAME\"
        },
        \"outputFileName\": \"$OUTPUT_FILE_NAME\"
    }
}")
# </score_endpoint_with_dataset>

JOB_ID=$(echo $response | jq -r '.id')
JOB_ID_SUFFIX=$(echo ${JOB_ID##/*/})

# <check_job>
wait_for_completion $SCORING_URI/$JOB_ID_SUFFIX $SCORING_TOKEN
# </check_job>
# </score_endpoint>

# <delete_endpoint>
curl --location --request DELETE "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RESOURCE_GROUP/providers/Microsoft.MachineLearningServices/workspaces/$WORKSPACE/batchEndpoints/$ENDPOINT_NAME?api-version=$API_VERSION" \
--header "Content-Type: application/json" \
--header "Authorization: Bearer $TOKEN" || true
# </delete_endpoint>

```

## Next steps

* Learn [how to deploy your model for batch scoring](batch-inference/how-to-use-batch-endpoint.md).
* Learn to [Troubleshoot batch endpoints](batch-inference/how-to-troubleshoot-batch-endpoints.md)
