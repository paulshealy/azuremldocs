
# Train scikit-learn models at scale with Azure Machine Learning

[!INCLUDE [sdk v2](../../includes/machine-learning-sdk-v2.md)]
> [!div class="op_single_selector" title1="Select the Azure Machine Learning SDK version you are using:"]
> * [v1](v1/how-to-train-scikit-learn.md)
> * [v2 (current version)](how-to-train-scikit-learn.md)

In this article, learn how to run your scikit-learn training scripts with Azure Machine Learning Python SDK v2.

The example scripts in this article are used to classify iris flower images to build a machine learning model based on scikit-learn's [iris dataset](https://archive.ics.uci.edu/ml/datasets/iris).

Whether you're training a machine learning scikit-learn model from the ground-up or you're bringing an existing model into the cloud, you can use Azure Machine Learning to scale out open-source training jobs using elastic cloud compute resources. You can build, deploy, version, and monitor production-grade models with Azure Machine Learning.

## Prerequisites

You can run the code for this article in either an Azure Machine Learning compute instance, or your own Jupyter Notebook.

 - Azure Machine Learning compute instance
    - Complete the [Quickstart: Get started with Azure Machine Learning](quickstart-create-resources.md) to create a compute instance. Every compute instance includes a dedicated notebook server pre-loaded with the SDK and the notebooks sample repository.
    - Select the notebook tab in the Azure Machine Learning studio. In the samples training folder, find a completed and expanded notebook by navigating to this directory: **v2  > sdk > jobs > single-step > scikit-learn > train-hyperparameter-tune-deploy-with-sklearn**.
    - You can use the pre-populated code in the sample training folder to complete this tutorial.

 - Your Jupyter notebook server.
    - [Install the Azure Machine Learning SDK (v2)](https://aka.ms/sdk-v2-install).


## Set up the job

This section sets up the job for training by loading the required Python packages, connecting to a workspace, creating a compute resource to run a command job, and creating an environment to run the job.

### Connect to the workspace

First, you'll need to connect to your AzureML workspace. The [AzureML workspace](concept-workspace.md) is the top-level resource for the service. It provides you with a centralized place to work with all the artifacts you create when you use Azure Machine Learning.

We're using `DefaultAzureCredential` to get access to the workspace. This credential should be capable of handling most Azure SDK authentication scenarios.

If `DefaultAzureCredential` does not work for you, see [`azure-identity reference documentation`](/python/api/azure-identity/azure.identity) or [`Set up authentication`](how-to-setup-authentication.md?tabs=sdk) for more available credentials.

```python
# Handle to the workspace
from azure.ai.ml import MLClient

# Authentication package
from azure.identity import DefaultAzureCredential

credential = DefaultAzureCredential()
```

If you prefer to use a browser to sign in and authenticate, you should remove the comments in the following code and use it instead.

```python
# Handle to the workspace
# from azure.ai.ml import MLClient

# Authentication package
# from azure.identity import InteractiveBrowserCredential
# credential = InteractiveBrowserCredential()
```

Next, get a handle to the workspace by providing your Subscription ID, Resource Group name, and workspace name. To find these parameters:

1. Look in the upper-right corner of the Azure Machine Learning studio toolbar for your workspace name.
2. Select your workspace name to show your Resource Group and Subscription ID.
3. Copy the values for Resource Group and Subscription ID into the code.

```python
# Get a handle to the workspace
ml_client = MLClient(
    credential=credential,
    subscription_id="<SUBSCRIPTION_ID>",
    resource_group_name="<RESOURCE_GROUP>",
    workspace_name="<AML_WORKSPACE_NAME>",
)
```

The result of running this script is a workspace handle that you'll use to manage other resources and jobs.

> [!NOTE]
> Creating `MLClient` will not connect the client to the workspace. The client initialization is lazy and will wait for the first time it needs to make a call. In this article, this will happen during compute creation.

### Create a compute resource to run the job

AzureML needs a compute resource to run a job. This resource can be single or multi-node machines with Linux or Windows OS, or a specific compute fabric like Spark.

In the following example script, we provision a Linux [`compute cluster`](./how-to-create-attach-compute-cluster.md?tabs=python). You can see the [`Azure Machine Learning pricing`](https://azure.microsoft.com/pricing/details/machine-learning/) page for the full list of VM sizes and prices. We only need a basic cluster for this example; thus, we'll pick a Standard_DS3_v2 model with 2 vCPU cores and 7 GB RAM to create an AzureML compute.

```python
from azure.ai.ml.entities import AmlCompute

# Name assigned to the compute cluster
cpu_compute_target = "cpu-cluster"

try:
    # let's see if the compute target already exists
    cpu_cluster = ml_client.compute.get(cpu_compute_target)
    print(
        f"You already have a cluster named {cpu_compute_target}, we'll reuse it as is."
    )

except Exception:
    print("Creating a new cpu compute target...")

    # Let's create the Azure ML compute object with the intended parameters
    cpu_cluster = AmlCompute(
        name=cpu_compute_target,
        # Azure ML Compute is the on-demand VM service
        type="amlcompute",
        # VM Family
        size="STANDARD_DS3_V2",
        # Minimum running nodes when there is no job running
        min_instances=0,
        # Nodes in cluster
        max_instances=4,
        # How many seconds will the node running after the job termination
        idle_time_before_scale_down=180,
        # Dedicated or LowPriority. The latter is cheaper but there is a chance of job termination
        tier="Dedicated",
    )

    # Now, we pass the object to MLClient's create_or_update method
    cpu_cluster = ml_client.compute.begin_create_or_update(cpu_cluster).result()

print(
    f"AMLCompute with name {cpu_cluster.name} is created, the compute size is {cpu_cluster.size}"
)
```

### Create a job environment

To run an AzureML job, you'll need an environment. An AzureML [environment](concept-environments.md) encapsulates the dependencies (such as software runtime and libraries) needed to run your machine learning training script on your compute resource. This environment is similar to a Python environment on your local machine.

AzureML allows you to either use a curated (or ready-made) environment or create a custom environment using a Docker image or a Conda configuration. In this article, you'll create a custom environment for your jobs, using a Conda YAML file.

#### Create a custom environment

To create your custom environment, you'll define your Conda dependencies in a YAML file. First, create a directory for storing the file. In this example, we've named the directory `env`.

```python
import os

dependencies_dir = "./env"
os.makedirs(dependencies_dir, exist_ok=True)
```

Then, create the file in the dependencies directory. In this example, we've named the file `conda.yml`.

```python
%%writefile {dependencies_dir}/conda.yml
name: sklearn-env
channels:
  - conda-forge
dependencies:
  - python=3.8
  - pip=21.2.4
  - scikit-learn=0.24.2
  - scipy=1.7.1
  - pip:  
    - mlflow== 1.26.1
    - azureml-mlflow==1.42.0
```

The specification contains some usual packages (such as numpy and pip) that you'll use in your job.

Next, use the YAML file to create and register this custom environment in your workspace. The environment will be packaged into a Docker container at runtime.

```python
from azure.ai.ml.entities import Environment

custom_env_name = "sklearn-env"

job_env = Environment(
    name=custom_env_name,
    description="Custom environment for sklearn image classification",
    conda_file=os.path.join(dependencies_dir, "conda.yml"),
    image="mcr.microsoft.com/azureml/openmpi4.1.0-ubuntu20.04:latest",
)
job_env = ml_client.environments.create_or_update(job_env)

print(
    f"Environment with name {job_env.name} is registered to workspace, the environment version is {job_env.version}"
)
```

For more information on creating and using environments, see [Create and use software environments in Azure Machine Learning](how-to-use-environments.md).

## Configure and submit your training job

In this section, we'll cover how to run a training job, using a training script that we've provided. To begin, you'll build the training job by configuring the command for running the training script. Then, you'll submit the training job to run in AzureML.


### Prepare the training script

In this article, we've provided the training script *train_iris.py*. In practice, you should be able to take any custom training script as is and run it with AzureML without having to modify your code.

> [!NOTE]
> The provided training script does the following:
> - shows how to log some metrics to your AzureML run;
> - downloads and extracts the training data using `iris = datasets.load_iris()`; and
> - trains a model, then saves and registers it.

To use and access your own data, see [how to read and write data in a job](how-to-read-write-data-v2.md) to make data available during training.

To use the training script, first create a directory where you will store the file.

```python
import os

src_dir = "./src"
os.makedirs(src_dir, exist_ok=True)
```

Next, create the script file in the source directory.

```python
%%writefile {src_dir}/train_iris.py
# Modified from https://www.geeksforgeeks.org/multiclass-classification-using-scikit-learn/

import argparse
import os

# importing necessary libraries
import numpy as np

from sklearn import datasets
from sklearn.metrics import confusion_matrix
from sklearn.model_selection import train_test_split

import joblib

import mlflow
import mlflow.sklearn

def main():
    parser = argparse.ArgumentParser()

    parser.add_argument('--kernel', type=str, default='linear',
                        help='Kernel type to be used in the algorithm')
    parser.add_argument('--penalty', type=float, default=1.0,
                        help='Penalty parameter of the error term')

    # Start Logging
    mlflow.start_run()

    # enable autologging
    mlflow.sklearn.autolog()

    args = parser.parse_args()
    mlflow.log_param('Kernel type', str(args.kernel))
    mlflow.log_metric('Penalty', float(args.penalty))

    # loading the iris dataset
    iris = datasets.load_iris()

    # X -> features, y -> label
    X = iris.data
    y = iris.target

    # dividing X, y into train and test data
    X_train, X_test, y_train, y_test = train_test_split(X, y, random_state=0)

    # training a linear SVM classifier
    from sklearn.svm import SVC
    svm_model_linear = SVC(kernel=args.kernel, C=args.penalty)
    svm_model_linear = svm_model_linear.fit(X_train, y_train)
    svm_predictions = svm_model_linear.predict(X_test)

    # model accuracy for X_test
    accuracy = svm_model_linear.score(X_test, y_test)
    print('Accuracy of SVM classifier on test set: {:.2f}'.format(accuracy))
    mlflow.log_metric('Accuracy', float(accuracy))
    # creating a confusion matrix
    cm = confusion_matrix(y_test, svm_predictions)
    print(cm)

    registered_model_name="sklearn-iris-flower-classify-model"

    ##########################
    #<save and register model>
    ##########################
    # Registering the model to the workspace
    print("Registering the model via MLFlow")
    mlflow.sklearn.log_model(
        sk_model=svm_model_linear,
        registered_model_name=registered_model_name,
        artifact_path=registered_model_name
    )

    # # Saving the model to a file
    print("Saving the model via MLFlow")
    mlflow.sklearn.save_model(
        sk_model=svm_model_linear,
        path=os.path.join(registered_model_name, "trained_model"),
    )
    ###########################
    #</save and register model>
    ###########################
    mlflow.end_run()

if __name__ == '__main__':
    main()
```

### Build the training job

Now that you have all the assets required to run your job, it's time to build it using the AzureML Python SDK v2. For this, we'll be creating a `command`.

An AzureML `command` is a resource that specifies all the details needed to execute your training code in the cloud. These details include the inputs and outputs, type of hardware to use, software to install, and how to run your code. The `command` contains information to execute a single command.


#### Configure the command

You'll use the general purpose `command` to run the training script and perform your desired tasks. Create a `Command` object to specify the configuration details of your training job. 

- The inputs for this command include the number of epochs, learning rate, momentum, and output directory.
- For the parameter values:
    - provide the compute cluster `cpu_compute_target = "cpu-cluster"` that you created for running this command;
    - provide the custom environment `sklearn-env` that you created for running the AzureML job;
    - configure the command line action itself—in this case, the command is `python train_iris.py`. You can access the inputs and outputs in the command via the `${{ ... }}` notation; and
    - configure the metadata such as the display name and experiment name; where an experiment is a container for all the iterations one does on a certain project. Note that all the jobs submitted under the same experiment name would be listed next to each other in AzureML studio.

```python
from azure.ai.ml import command
from azure.ai.ml import Input

job = command(
    inputs=dict(kernel="linear", penalty=1.0),
    compute=cpu_compute_target,
    environment=f"{job_env.name}:{job_env.version}",
    code="./src/",
    command="python train_iris.py --kernel ${{inputs.kernel}} --penalty ${{inputs.penalty}}",
    experiment_name="sklearn-iris-flowers",
    display_name="sklearn-classify-iris-flower-images",
)
```

### Submit the job

It's now time to submit the job to run in AzureML. This time you'll use `create_or_update` on `ml_client.jobs`. 

```python
ml_client.jobs.create_or_update(job)
```

Once completed, the job will register a model in your workspace (as a result of training) and output a link for viewing the job in AzureML studio.

> [!WARNING]
> Azure Machine Learning runs training scripts by copying the entire source directory. If you have sensitive data that you don't want to upload, use a [.ignore file](concept-train-machine-learning-model.md#understand-what-happens-when-you-submit-a-training-job) or don't include it in the source directory.

### What happens during job execution
As the job is executed, it goes through the following stages:

- **Preparing**: A docker image is created according to the environment defined. The image is uploaded to the workspace's container registry and cached for later runs. Logs are also streamed to the run history and can be viewed to monitor progress. If a curated environment is specified, the cached image backing that curated environment will be used.

- **Scaling**: The cluster attempts to scale up if the cluster requires more nodes to execute the run than are currently available.

- **Running**: All scripts in the script folder *src* are uploaded to the compute target, data stores are mounted or copied, and the script is executed. Outputs from *stdout* and the *./logs* folder are streamed to the run history and can be used to monitor the run.

## Tune model hyperparameters

Now that you've seen how to do a simple Scikit-learn training run using the SDK, let's see if you can further improve the accuracy of your model. You can tune and optimize our model's hyperparameters using Azure Machine Learning's [`sweep`](/python/api/azure-ai-ml/azure.ai.ml.sweep) capabilities.

To tune the model's hyperparameters, define the parameter space in which to search during training. You'll do this by replacing some of the parameters (`kernel` and `penalty`) passed to the training job with special inputs from the `azure.ml.sweep` package.

```python
from azure.ai.ml.sweep import Choice

# we will reuse the command_job created before. we call it as a function so that we can apply inputs
# we do not apply the 'iris_csv' input again -- we will just use what was already defined earlier
job_for_sweep = job(
    kernel=Choice(values=["linear", "rbf", "poly", "sigmoid"]),
    penalty=Choice(values=[0.5, 1, 1.5]),
)
```

Then, you'll configure sweep on the command job, using some sweep-specific parameters, such as the primary metric to watch and the sampling algorithm to use.

In the following code we use random sampling to try different configuration sets of hyperparameters in an attempt to maximize our primary metric, `Accuracy`.

```python
sweep_job = job_for_sweep.sweep(
    compute="cpu-cluster",
    sampling_algorithm="random",
    primary_metric="Accuracy",
    goal="Maximize",
    max_total_trials=12,
    max_concurrent_trials=4,
)
```

Now, you can submit this job as before. This time, you'll be running a sweep job that sweeps over your train job.

```python
returned_sweep_job = ml_client.create_or_update(sweep_job)

# stream the output and wait until the job is finished
ml_client.jobs.stream(returned_sweep_job.name)

# refresh the latest status of the job after streaming
returned_sweep_job = ml_client.jobs.get(name=returned_sweep_job.name)
```

You can monitor the job by using the studio user interface link that is presented during the job run.


## Find and register the best model

Once all the runs complete, you can find the run that produced the model with the highest accuracy.

```python
from azure.ai.ml.entities import Model

if returned_sweep_job.status == "Completed":

    # First let us get the run which gave us the best result
    best_run = returned_sweep_job.properties["best_child_run_id"]

    # lets get the model from this run
    model = Model(
        # the script stores the model as "sklearn-iris-flower-classify-model"
        path="azureml://jobs/{}/outputs/artifacts/paths/sklearn-iris-flower-classify-model/".format(
            best_run
        ),
        name="run-model-example",
        description="Model created from run.",
        type="custom_model",
    )

else:
    print(
        "Sweep job status: {}. Please wait until it completes".format(
            returned_sweep_job.status
        )
    )
```

You can then register this model.

```python
registered_model = ml_client.models.create_or_update(model=model)
```


## Deploy the model

After you've registered your model, you can deploy it the same way as any other registered model in Azure ML. For more information about deployment, see [Deploy and score a machine learning model with managed online endpoint using Python SDK v2](how-to-deploy-managed-online-endpoint-sdk-v2.md).


## Next steps

In this article, you trained and registered a scikit-learn model, and you learned about deployment options. See these other articles to learn more about Azure Machine Learning.

* [Track run metrics during training](how-to-log-view-metrics.md)
* [Tune hyperparameters](how-to-tune-hyperparameters.md)