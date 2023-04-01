
# How to do hyperparameter tuning in pipeline (v2)

[!INCLUDE [dev v2](../../includes/machine-learning-dev-v2.md)]

In this article, you'll learn how to do hyperparameter tuning in Azure Machine Learning pipeline.

## Prerequisite

1. Understand what is [hyperparameter tuning](how-to-tune-hyperparameters.md) and how to do hyperparameter tuning in Azure Machine Learning use SweepJob.
2. Understand what is a [Azure Machine Learning pipeline](concept-ml-pipelines.md)
3. Build a command component that takes hyperparameter as input.

## How to do hyperparameter tuning in Azure Machine Learning pipeline

This section explains how to do hyperparameter tuning in Azure Machine Learning pipeline using CLI v2 and Python SDK. Both approaches share the same prerequisite: you already have a command component created and the command component takes hyperparameters as inputs. If you don't have a command component yet. Follow below links to create a command component first.

- [AzureML CLI v2](how-to-create-component-pipelines-cli.md)
- [AzureML Python SDK v2](how-to-create-component-pipeline-python.md)

### CLI v2

The example used in this article can be found in [azureml-example repo](https://github.com/Azure/azureml-examples). Navigate to *[azureml-examples/cli/jobs/pipelines-with-components/pipeline_with_hyperparameter_sweep* to check the example.

Assume you already have a command component defined in `train.yaml`. A two-step pipeline job (train and predict) YAML file looks like below.

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/pipelineJob.schema.json
type: pipeline
display_name: pipeline_with_hyperparameter_sweep
description: Tune hyperparameters using TF component
settings:
    default_compute: azureml:cpu-cluster
jobs:
  sweep_step:
    type: sweep
    inputs:
      data: 
        type: uri_file
        path: wasbs://datasets@azuremlexamples.blob.core.windows.net/iris.csv
      degree: 3
      gamma: "scale"
      shrinking: False
      probability: False
      tol: 0.001
      cache_size: 1024
      verbose: False
      max_iter: -1
      decision_function_shape: "ovr"
      break_ties: False
      random_state: 42
    outputs:
      model_output:
      test_data:
    sampling_algorithm: random
    trial: ./train.yml
    search_space:
      c_value:
        type: uniform
        min_value: 0.5
        max_value: 0.9
      kernel:
        type: choice
        values: ["rbf", "linear", "poly"]
      coef0:
        type: uniform
        min_value: 0.1
        max_value: 1
    objective:
      goal: minimize
      primary_metric: training_f1_score
    limits:
      max_total_trials: 5
      max_concurrent_trials: 3
      timeout: 7200

  predict_step:
    type: command
    inputs:
      model: ${{parent.jobs.sweep_step.outputs.model_output}}
      test_data: ${{parent.jobs.sweep_step.outputs.test_data}}
    outputs:
      predict_result:
    component: ./predict.yml

    
```

The `sweep_step` is the step for hyperparameter tuning. Its type needs to be `sweep`.  And `trial` refers to the command component defined in `train.yaml`. From the `search space` field we can see three hyparmeters (`c_value`, `kernel`, and `coef`) are added to the search space. After you submit this pipeline job, Azure Machine Learning will run the trial component multiple times to sweep over hyperparameters based on the search space and terminate policy you defined in `sweep_step`. Check [sweep job YAML schema](reference-yaml-job-sweep.md) for full schema of sweep job.

Below is the trial component definition (train.yml file). 

```yaml
$schema: https://azuremlschemas.azureedge.net/latest/commandComponent.schema.json
type: command

name: train_model
display_name: train_model
version: 1

inputs: 
  data:
    type: uri_folder
  c_value:
    type: number
    default: 1.0
  kernel:
    type: string
    default: rbf
  degree:
    type: integer
    default: 3
  gamma:
    type: string
    default: scale
  coef0: 
    type: number
    default: 0
  shrinking:
    type: boolean
    default: false
  probability:
    type: boolean
    default: false
  tol:
    type: number
    default: 1e-3
  cache_size:
    type: number
    default: 1024
  verbose:
    type: boolean
    default: false
  max_iter:
    type: integer
    default: -1
  decision_function_shape:
    type: string
    default: ovr
  break_ties:
    type: boolean
    default: false
  random_state:
    type: integer
    default: 42

outputs:
  model_output:
    type: mlflow_model
  test_data:
    type: uri_folder
  
code: ./train-src

environment: azureml:AzureML-sklearn-1.0-ubuntu20.04-py38-cpu@latest

command: >-
  python train.py 
  --data ${{inputs.data}}
  --C ${{inputs.c_value}}
  --kernel ${{inputs.kernel}}
  --degree ${{inputs.degree}}
  --gamma ${{inputs.gamma}}
  --coef0 ${{inputs.coef0}}
  --shrinking ${{inputs.shrinking}}
  --probability ${{inputs.probability}}
  --tol ${{inputs.tol}}
  --cache_size ${{inputs.cache_size}}
  --verbose ${{inputs.verbose}}
  --max_iter ${{inputs.max_iter}}
  --decision_function_shape ${{inputs.decision_function_shape}}
  --break_ties ${{inputs.break_ties}}
  --random_state ${{inputs.random_state}}
  --model_output ${{outputs.model_output}}
  --test_data ${{outputs.test_data}}
```

The hyperparameters added to search space in pipeline.yml need to be inputs for the trial component. The source code of the trial component is under `./train-src` folder. In this example, it's a single `train.py` file. This is the code that will be executed in every trial of the sweep job. Make sure you've logged the metrics in the trial component source code with exactly the same name as `primary_metric` value in pipeline.yml file. In this example, we use `mlflow.autolog()`, which is the recommended way to track your ML experiments. See more about mlflow [here](./how-to-use-mlflow-cli-runs.md)  
 
Below code snippet is the source code of trial component. 

```python
# imports
import os
import mlflow
import argparse

import pandas as pd
from pathlib import Path

from sklearn.svm import SVC
from sklearn.model_selection import train_test_split

# define functions
def main(args):
    # enable auto logging
    mlflow.autolog()

    # setup parameters
    params = {
        "C": args.C,
        "kernel": args.kernel,
        "degree": args.degree,
        "gamma": args.gamma,
        "coef0": args.coef0,
        "shrinking": args.shrinking,
        "probability": args.probability,
        "tol": args.tol,
        "cache_size": args.cache_size,
        "class_weight": args.class_weight,
        "verbose": args.verbose,
        "max_iter": args.max_iter,
        "decision_function_shape": args.decision_function_shape,
        "break_ties": args.break_ties,
        "random_state": args.random_state,
    }

    # read in data
    df = pd.read_csv(args.data)

    # process data
    X_train, X_test, y_train, y_test = process_data(df, args.random_state)

    # train model
    model = train_model(params, X_train, X_test, y_train, y_test)
    # Output the model and test data
    # write to local folder first, then copy to output folder

    mlflow.sklearn.save_model(model, "model")

    from distutils.dir_util import copy_tree

    # copy subdirectory example
    from_directory = "model"
    to_directory = args.model_output

    copy_tree(from_directory, to_directory)

    X_test.to_csv(Path(args.test_data) / "X_test.csv", index=False)
    y_test.to_csv(Path(args.test_data) / "y_test.csv", index=False)


def process_data(df, random_state):
    # split dataframe into X and y
    X = df.drop(["species"], axis=1)
    y = df["species"]

    # train/test split
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, random_state=random_state
    )

    # return split data
    return X_train, X_test, y_train, y_test


def train_model(params, X_train, X_test, y_train, y_test):
    # train model
    model = SVC(**params)
    model = model.fit(X_train, y_train)

    # return model
    return model


def parse_args():
    # setup arg parser
    parser = argparse.ArgumentParser()

    # add arguments
    parser.add_argument("--data", type=str)
    parser.add_argument("--C", type=float, default=1.0)
    parser.add_argument("--kernel", type=str, default="rbf")
    parser.add_argument("--degree", type=int, default=3)
    parser.add_argument("--gamma", type=str, default="scale")
    parser.add_argument("--coef0", type=float, default=0)
    parser.add_argument("--shrinking", type=bool, default=False)
    parser.add_argument("--probability", type=bool, default=False)
    parser.add_argument("--tol", type=float, default=1e-3)
    parser.add_argument("--cache_size", type=float, default=1024)
    parser.add_argument("--class_weight", type=dict, default=None)
    parser.add_argument("--verbose", type=bool, default=False)
    parser.add_argument("--max_iter", type=int, default=-1)
    parser.add_argument("--decision_function_shape", type=str, default="ovr")
    parser.add_argument("--break_ties", type=bool, default=False)
    parser.add_argument("--random_state", type=int, default=42)
    parser.add_argument("--model_output", type=str, help="Path of output model")
    parser.add_argument("--test_data", type=str, help="Path of output model")

    # parse args
    args = parser.parse_args()

    # return args
    return args


# run script
if __name__ == "__main__":
    # parse args
    args = parse_args()

    # run main function
    main(args)

```

### Python SDK

The Python SDK example can be found in [azureml-example repo](https://github.com/Azure/azureml-examples). Navigate to *azureml-examples/sdk/jobs/pipelines/1c_pipeline_with_hyperparameter_sweep* to check the example.

In Azure Machine Learning Python SDK v2, you can enable hyperparameter tuning for any command component by calling `.sweep()` method.

Below code snippet shows how to enable sweep for `train_model`.

```python
train_component_func = load_component(source="./train.yml")
score_component_func = load_component(source="./predict.yml")

# define a pipeline
@pipeline()
def pipeline_with_hyperparameter_sweep():
    """Tune hyperparameters using sample components."""
    train_model = train_component_func(
        data=Input(
            type="uri_file",
            path="wasbs://datasets@azuremlexamples.blob.core.windows.net/iris.csv",
        ),
        c_value=Uniform(min_value=0.5, max_value=0.9),
        kernel=Choice(["rbf", "linear", "poly"]),
        coef0=Uniform(min_value=0.1, max_value=1),
        degree=3,
        gamma="scale",
        shrinking=False,
        probability=False,
        tol=0.001,
        cache_size=1024,
        verbose=False,
        max_iter=-1,
        decision_function_shape="ovr",
        break_ties=False,
        random_state=42,
    )
    sweep_step = train_model.sweep(
        primary_metric="training_f1_score",
        goal="minimize",
        sampling_algorithm="random",
        compute="cpu-cluster",
    )
    sweep_step.set_limits(max_total_trials=20, max_concurrent_trials=10, timeout=7200)

    score_data = score_component_func(
        model=sweep_step.outputs.model_output, test_data=sweep_step.outputs.test_data
    )


pipeline_job = pipeline_with_hyperparameter_sweep()

# set pipeline level compute
pipeline_job.settings.default_compute = "cpu-cluster"
```

 We first load `train_component_func` defined in `train.yml` file. When creating `train_model`, we add `c_value`, `kernel` and `coef0` into search space(line 15-17). Line 30-35 defines the primary metric, sampling algorithm etc.

## Check pipeline job with sweep step in Studio

After you submit a pipeline job, the SDK or CLI widget will give you a web URL link to Studio UI. The link will guide you to the pipeline graph view by default.

To check details of the sweep step, double click the sweep step and navigate to the **child job** tab in the panel on the right.

:::image type="content" source="./media/how-to-use-sweep-in-pipeline/pipeline-view.png" alt-text="Screenshot of the pipeline with child job and the train_model node highlighted." lightbox= "./media/how-to-use-sweep-in-pipeline/pipeline-view.png":::

This will link you to the sweep job page as seen in the below screenshot. Navigate to **child job** tab, here you can see the metrics of all child jobs and list of all child jobs.

:::image type="content" source="./media/how-to-use-sweep-in-pipeline/sweep-job.png" alt-text="Screenshot of the job page on the child jobs tab." lightbox= "./media/how-to-use-sweep-in-pipeline/sweep-job.png":::

If a child jobs failed, select the name of that child job to enter detail page of that specific child job (see screenshot below). The useful debug information is under **Outputs + Logs**.

:::image type="content" source="./media/how-to-use-sweep-in-pipeline/child-run.png" alt-text="Screenshot of the output + logs tab of a child run." lightbox= "./media/how-to-use-sweep-in-pipeline/child-run.png":::

## Sample notebooks

- [Build pipeline with sweep node](https://github.com/Azure/azureml-examples/blob/main/sdk/python/jobs/pipelines/1c_pipeline_with_hyperparameter_sweep/pipeline_with_hyperparameter_sweep.ipynb)
- [Run hyperparameter sweep on a command job](https://github.com/Azure/azureml-examples/blob/main/sdk/python/jobs/single-step/lightgbm/iris/lightgbm-iris-sweep.ipynb)

## Next steps

- [Track an experiment](how-to-log-view-metrics.md)
- [Deploy a trained model](how-to-deploy-online-endpoints.md)
