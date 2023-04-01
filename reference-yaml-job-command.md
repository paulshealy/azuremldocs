
# CLI (v2) command job YAML schema

[!INCLUDE [cli v2](../../includes/machine-learning-cli-v2.md)]

The source JSON schema can be found at https://azuremlschemas.azureedge.net/latest/commandJob.schema.json.



[!INCLUDE [schema note](../../includes/machine-learning-preview-old-json-schema-note.md)]

## YAML syntax

| Key | Type | Description | Allowed values | Default value |
| --- | ---- | ----------- | -------------- | ------------- |
| `$schema` | string | The YAML schema. If you use the Azure Machine Learning VS Code extension to author the YAML file, including `$schema` at the top of your file enables you to invoke schema and resource completions. | | |
| `type` | const | The type of job. | `command` | `command` |
| `name` | string | Name of the job. Must be unique across all jobs in the workspace. If omitted, Azure ML will autogenerate a GUID for the name. | | |
| `display_name` | string | Display name of the job in the studio UI. Can be non-unique within the workspace. If omitted, Azure ML will autogenerate a human-readable adjective-noun identifier for the display name. | | |
| `experiment_name` | string | Experiment name to organize the job under. Each job's run record will be organized under the corresponding experiment in the studio's "Experiments" tab. If omitted, Azure ML will default it to the name of the working directory where the job was created. | | |
| `description` | string | Description of the job. | | |
| `tags` | object | Dictionary of tags for the job. | | |
| `command` | string | **Required (if not using `component` field).** The command to execute. | | |
| `code` | string | Local path to the source code directory to be uploaded and used for the job. | | |
| `environment` | string or object | **Required (if not using `component` field).** The environment to use for the job. This can be either a reference to an existing versioned environment in the workspace or an inline environment specification. <br><br> To reference an existing environment use the `azureml:<environment_name>:<environment_version>` syntax or `azureml:<environment_name>@latest` (to reference the latest version of an environment). <br><br> To define an environment inline please follow the [Environment schema](reference-yaml-environment.md#yaml-syntax). Exclude the `name` and `version` properties as they are not supported for inline environments. | | |
| `environment_variables` | object | Dictionary of environment variable key-value pairs to set on the process where the command is executed. | | |
| `distribution` | object | The distribution configuration for distributed training scenarios. One of [MpiConfiguration](#mpiconfiguration), [PyTorchConfiguration](#pytorchconfiguration), or [TensorFlowConfiguration](#tensorflowconfiguration). | | |
| `compute` | string | Name of the compute target to execute the job on. This can be either a reference to an existing compute in the workspace (using the `azureml:<compute_name>` syntax) or `local` to designate local execution. **Note:** jobs in pipeline didn't support `local` as `compute` | | `local` |
| `resources.instance_count` | integer | The number of nodes to use for the job. | | `1` |
| `resources.instance_type` | string | The instance type to use for the job. Applicable for jobs running on Azure Arc-enabled Kubernetes compute (where the compute target specified in the `compute` field is of `type: kubernentes`). If omitted, this will default to the default instance type for the Kubernetes cluster. For more information, see [Create and select Kubernetes instance types](how-to-attach-kubernetes-anywhere.md). | | |
| `limits.timeout` | integer | The maximum time in seconds the job is allowed to run. Once this limit is reached the system will cancel the job. | | |
| `inputs` | object | Dictionary of inputs to the job. The key is a name for the input within the context of the job and the value is the input value. <br><br> Inputs can be referenced in the `command` using the `${{ inputs.<input_name> }}` expression. | | |
| `inputs.<input_name>` | number, integer, boolean, string or object | One of a literal value (of type number, integer, boolean, or string) or an object containing a [job input data specification](#job-inputs). | | |
| `outputs` | object | Dictionary of output configurations of the job. The key is a name for the output within the context of the job and the value is the output configuration. <br><br> Outputs can be referenced in the `command` using the `${{ outputs.<output_name> }}` expression. | |
| `outputs.<output_name>` | object | You can leave the object empty, in which case by default the output will be of type `uri_folder` and Azure ML will system-generate an output location for the output. File(s) to the output directory will be written via read-write mount. If you want to specify a different mode for the output, provide an object containing the [job output specification](#job-outputs). | |
| `identity` | object | The identity is used for data accessing. It can be [UserIdentityConfiguration](#useridentityconfiguration), [ManagedIdentityConfiguration](#managedidentityconfiguration) or None. If it's UserIdentityConfiguration the identity of job submitter will be used to access input data and write result to output folder, otherwise, the managed identity of the compute target will be used. | |

### Distribution configurations

#### MpiConfiguration

| Key | Type | Description | Allowed values |
| --- | ---- | ----------- | -------------- |
| `type` | const | **Required.** Distribution type.  | `mpi` |
| `process_count_per_instance` | integer | **Required.** The number of processes per node to launch for the job.  | |

#### PyTorchConfiguration

| Key | Type | Description | Allowed values | Default value |
| --- | ---- | ----------- | -------------- | ------------- |
| `type` | const | **Required.** Distribution type.  | `pytorch` | |
| `process_count_per_instance` | integer | The number of processes per node to launch for the job. | |  `1` |

#### TensorFlowConfiguration

| Key | Type | Description | Allowed values | Default value |
| --- | ---- | ----------- | -------------- | ------------- |
| `type` | const | **Required.** Distribution type.  | `tensorflow` |
| `worker_count` | integer | The number of workers to launch for the job. | | Defaults to `resources.instance_count`. |
| `parameter_server_count` | integer | The number of parameter servers to launch for the job. | | `0` |

### Job inputs

| Key | Type | Description | Allowed values | Default value |
| --- | ---- | ----------- | -------------- | ------------- |
| `type` | string | The type of job input. Specify `uri_file` for input data that points to a single file source, or `uri_folder` for input data that points to a folder source. | `uri_file`, `uri_folder`, `mlflow_model`, `custom_model`| `uri_folder` |
| `path` | string | The path to the data to use as input. This can be specified in a few ways: <br><br> - A local path to the data source file or folder, e.g. `path: ./iris.csv`. The data will get uploaded during job submission. <br><br> - A URI of a cloud path to the file or folder to use as the input. Supported URI types are `azureml`, `https`, `wasbs`, `abfss`, `adl`. See [Core yaml syntax](reference-yaml-core-syntax.md) for more information on how to use the `azureml://` URI format. <br><br> - An existing registered Azure ML data asset to use as the input. To reference a registered data asset use the `azureml:<data_name>:<data_version>` syntax or `azureml:<data_name>@latest` (to reference the latest version of that data asset), e.g. `path: azureml:cifar10-data:1` or `path: azureml:cifar10-data@latest`. | | |
| `mode` | string | Mode of how the data should be delivered to the compute target. <br><br> For read-only mount (`ro_mount`), the data will be consumed as a mount path. A folder will be mounted as a folder and a file will be mounted as a file. Azure ML will resolve the input to the mount path. <br><br> For `download` mode the data will be downloaded to the compute target. Azure ML will resolve the input to the downloaded path. <br><br> If you only want the URL of the storage location of the data artifact(s) rather than mounting or downloading the data itself, you can use the `direct` mode. This will pass in the URL of the storage location as the job input. Note that in this case you are fully responsible for handling credentials to access the storage. | `ro_mount`, `download`, `direct` | `ro_mount` |

### Job outputs

| Key | Type | Description | Allowed values | Default value |
| --- | ---- | ----------- | -------------- | ------------- |
| `type` | string | The type of job output. For the default `uri_folder` type, the output will correspond to a folder. | `uri_folder` , `mlflow_model`, `custom_model`| `uri_folder` |
| `mode` | string | Mode of how output file(s) will get delivered to the destination storage. For read-write mount mode (`rw_mount`) the output directory will be a mounted directory. For upload mode the file(s) written will get uploaded at the end of the job. | `rw_mount`, `upload` | `rw_mount` |

### Identity configurations

#### UserIdentityConfiguration

| Key | Type | Description | Allowed values |
| --- | ---- | ----------- | -------------- |
| `type` | const | **Required.** Identity type.  | `user_identity` |

#### ManagedIdentityConfiguration

| Key | Type | Description | Allowed values |
| --- | ---- | ----------- | -------------- |
| `type` | const | **Required.** Identity type.  | `managed` or `managed_identity` |

## Remarks

The `az ml job` command can be used for managing Azure Machine Learning jobs.

## Examples

Examples are available in the [examples GitHub repository](https://github.com/Azure/azureml-examples/tree/main/cli/jobs). Several are shown below.

## YAML: hello world

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
command: echo "hello world"
environment:
  image: library/python:latest
compute: azureml:cpu-cluster

```

## YAML: display name, experiment name, description, and tags

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
command: echo "hello world"
environment:
  image: library/python:latest
compute: azureml:cpu-cluster
tags:
  hello: world
display_name: hello-world-example
experiment_name: hello-world-example
description: |
  # Azure Machine Learning "hello world" job

  This is a "hello world" job running in the cloud via Azure Machine Learning!

  ## Description

  Markdown is supported in the studio for job descriptions! You can edit the description there or via CLI.

```

## YAML: environment variables

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
command: echo $hello_env_var
environment:
  image: library/python:latest
compute: azureml:cpu-cluster
environment_variables:
  hello_env_var: "hello world"

```

## YAML: source code

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
command: ls
code: src
environment:
  image: library/python:latest
compute: azureml:cpu-cluster

```

## YAML: literal inputs

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
command: |
  echo ${{inputs.hello_string}}
  echo ${{inputs.hello_number}}
environment:
  image: library/python:latest
inputs:
  hello_string: "hello world"
  hello_number: 42
compute: azureml:cpu-cluster

```

## YAML: write to default outputs

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
command: echo "hello world" > ./outputs/helloworld.txt
environment:
  image: library/python:latest
compute: azureml:cpu-cluster

```

## YAML: write to named data output

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
command: echo "hello world" > ${{outputs.hello_output}}/helloworld.txt
outputs:
  hello_output:
environment:
  image: python
compute: azureml:cpu-cluster

```

## YAML: datastore URI file input

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
command: |
  echo "--iris-csv: ${{inputs.iris_csv}}"
  python hello-iris.py --iris-csv ${{inputs.iris_csv}}
code: src
inputs:
  iris_csv:
    type: uri_file 
    path: azureml://datastores/workspaceblobstore/paths/example-data/iris.csv
environment: azureml:AzureML-sklearn-1.0-ubuntu20.04-py38-cpu@latest
compute: azureml:cpu-cluster

```

## YAML: datastore URI folder input

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
command: |
  ls ${{inputs.data_dir}}
  echo "--iris-csv: ${{inputs.data_dir}}/iris.csv"
  python hello-iris.py --iris-csv ${{inputs.data_dir}}/iris.csv
code: src
inputs:
  data_dir:
    type: uri_folder 
    path: azureml://datastores/workspaceblobstore/paths/example-data/
environment: azureml:AzureML-sklearn-1.0-ubuntu20.04-py38-cpu@latest
compute: azureml:cpu-cluster

```

## YAML: URI file input

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
command: |
  echo "--iris-csv: ${{inputs.iris_csv}}"
  python hello-iris.py --iris-csv ${{inputs.iris_csv}}
code: src
inputs:
  iris_csv:
    type: uri_file 
    path: https://azuremlexamples.blob.core.windows.net/datasets/iris.csv
environment: azureml:AzureML-sklearn-1.0-ubuntu20.04-py38-cpu@latest
compute: azureml:cpu-cluster

```

## YAML: URI folder input

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
command: |
  ls ${{inputs.data_dir}}
  echo "--iris-csv: ${{inputs.data_dir}}/iris.csv"
  python hello-iris.py --iris-csv ${{inputs.data_dir}}/iris.csv
code: src
inputs:
  data_dir:
    type: uri_folder 
    path: wasbs://datasets@azuremlexamples.blob.core.windows.net/
environment: azureml:AzureML-sklearn-1.0-ubuntu20.04-py38-cpu@latest
compute: azureml:cpu-cluster

```

## YAML: Notebook via papermill

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
command: |
  pip install ipykernel papermill
  papermill hello-notebook.ipynb outputs/out.ipynb -k python
code: src
environment:
  image: library/python:latest
compute: azureml:cpu-cluster

```

## YAML: basic Python model training

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

## YAML: basic R model training with local Docker build context

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
command: >
  Rscript train.R 
  --data_folder ${{inputs.iris}}
code: src
inputs:
  iris: 
    type: uri_file
    path: https://azuremlexamples.blob.core.windows.net/datasets/iris.csv
environment:
  build:
    path: docker-context
compute: azureml:cpu-cluster
display_name: r-iris-example
experiment_name: r-iris-example
description: Train an R model on the Iris dataset.

```

## YAML: distributed PyTorch

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
code: src
command: >-
  python train.py 
  --epochs ${{inputs.epochs}}
  --learning-rate ${{inputs.learning_rate}}
  --data-dir ${{inputs.cifar}}
inputs:
  epochs: 1
  learning_rate: 0.2
  cifar: 
     type: uri_folder
     path: azureml:cifar-10-example@latest
environment: azureml:AzureML-pytorch-1.9-ubuntu18.04-py37-cuda11-gpu@latest
compute: azureml:gpu-cluster
distribution:
  type: pytorch 
  process_count_per_instance: 1
resources:
  instance_count: 2
display_name: pytorch-cifar-distributed-example
experiment_name: pytorch-cifar-distributed-example
description: Train a basic convolutional neural network (CNN) with PyTorch on the CIFAR-10 dataset, distributed via PyTorch.

```

## YAML: distributed TensorFlow

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
code: src
command: >-
  python train.py 
  --epochs ${{inputs.epochs}}
  --model-dir ${{inputs.model_dir}}
inputs:
  epochs: 1
  model_dir: outputs/keras-model
environment: azureml:AzureML-tensorflow-2.4-ubuntu18.04-py37-cuda11-gpu@latest
compute: azureml:gpu-cluster
resources:
  instance_count: 2
distribution:
  type: tensorflow
  worker_count: 2
display_name: tensorflow-mnist-distributed-example
experiment_name: tensorflow-mnist-distributed-example
description: Train a basic neural network with TensorFlow on the MNIST dataset, distributed via TensorFlow.

```

## YAML: distributed MPI

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/commandJob.schema.json
code: src
command: >-
  python train.py
  --epochs ${{inputs.epochs}}
inputs:
  epochs: 1
environment: azureml:AzureML-tensorflow-2.7-ubuntu20.04-py38-cuda11-gpu@latest
compute: azureml:gpu-cluster
resources:
  instance_count: 2
distribution:
  type: mpi
  process_count_per_instance: 1
display_name: tensorflow-mnist-distributed-horovod-example
experiment_name: tensorflow-mnist-distributed-horovod-example
description: Train a basic neural network with TensorFlow on the MNIST dataset, distributed via Horovod.

```

## Next steps

- [Install and use the CLI (v2)](how-to-configure-cli.md)
