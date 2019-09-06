# tf-yarnᵝ

tf-yarn is a Python library we have built at Criteo for training TensorFlow models on a YARN cluster. An introducing blog post can be found [here](https://medium.com/criteo-labs/train-tensorflow-models-on-yarn-in-just-a-few-lines-of-code-ba0f354f38e3).

It supports running on one worker or on multiple workers with different distribution strategies, it can run on CPUs or GPUs and also runs with the recently added standalone client mode, and this with just a few lines of code.

Its API provides an easy entry point for working with Estimators. Keras is currently supported via the [model_to_estimator](https://www.tensorflow.org/api_docs/python/tf/keras/estimator/model_to_estimator) conversion function, and low-level distributed TensorFlow via standalone client mode API. Please refer to the [examples](https://github.com/criteo/tf-yarn/tree/master/examples) for some code samples.

[MLflow](https://www.mlflow.org/docs/latest/quickstart.html) is supported when MLflow package is installed (`pip install mlflow`)
For more info check out [examples/mlflow_example.py](https://github.com/criteo/tf-yarn/blob/master/examples/mlflow_example.py).

![tf-yarn](https://github.com/criteo/tf-yarn/blob/master/skein.png?raw=true)

## Installation

### Install with Pip

```bash
$ pip install tf-yarn
```

### Install from source

```bash
$ git clone https://github.com/criteo/tf-yarn
$ cd tf-yarn
$ pip install .
```

### Prerequisites

tf-yarn only supports Python ≥3.6.

Make sure to have Tensorflow working with HDFS by setting up all the environment variables as described [here](https://github.com/tensorflow/examples/blob/master/community/en/docs/deploy/hadoop.md).

You can run the `check_hadoop_env` script to check that your setup is OK (it has been installed by tf_yarn):

```
$ check_hadoop_env
# You should see something like
# INFO:tf_yarn.bin.check_hadoop_env:results will be written in /home/.../shared/Dev/tf-yarn/check_hadoop_env.log
# INFO:tf_yarn.bin.check_hadoop_env:check_env: True
# INFO:tf_yarn.bin.check_hadoop_env:write dummy file to hdfs hdfs://root/tmp/a1df7b99-fa47-4a86-b5f3-9bc09019190f/hello_tf_yarn.txt
# INFO:tf_yarn.bin.check_hadoop_env:check_local_hadoop_tensorflow: True
# INFO:root:Launching remote check
# ...
# INFO:tf_yarn.bin.check_hadoop_env:remote_check: True
# INFO:tf_yarn.bin.check_hadoop_env:Hadoop setup: OK
```

## tf-yarn API's

tf-yarn comes with two API's to launch a training — run_on_yarn and standalone_client_mode.

### run_on_yarn

The only abstraction tf-yarn adds on top of the ones already present in
TensorFlow is `experiment_fn`. It is a function returning a triple of one `Estimator` and two specs -- `TrainSpec` and `EvalSpec`.

Here is a stripped down `experiment_fn` from one of the provided [examples][linear_classifier_example] to give you an idea of how it might look:

```python
from tf_yarn import Experiment

def experiment_fn():
  # ...
  estimator = tf.estimator.DNNClassifier(...)
  return Experiment(
    estimator,
    tf.estimator.TrainSpec(train_input_fn, max_steps=...),
    tf.estimator.EvalSpec(eval_input_fn)
 )
```

An experiment can be scheduled on YARN using the run_on_yarn function which takes three required arguments:

- `pyenv_zip_path` which contains the tf-yarn modules and dependencies like TensorFlow to be shipped to the cluster. pyenv_zip_path can be generated easily with a helper function based on the current installed virtual environment;
- `experiment_fn` as described above;
- `task_specs` dictionary specifying how much resources to allocate for each of the distributed TensorFlow task type.

The example uses the [Wine Quality][wine-quality] dataset from UCI ML repository. With just under 5000 training instances available, there is no need for multi-node training, meaning that a chief complemented by an evaluator would manage just fine. Note that each task will be executed in its own YARN container.

```python
from tf_yarn import TaskSpec, run_on_yarn
from tf_yarn import packaging

pyenv_zip_path = packaging.upload_env_to_hdfs()
run_on_yarn(
    pyenv_zip_path,
    experiment_fn,
    task_specs={
        "chief": TaskSpec(memory="2 GiB", vcores=4),
        "evaluator": TaskSpec(memory="2 GiB", vcores=1),
        "tensorboard": TaskSpec(memory="2 GiB", vcores=1)
    }
)
```

The final bit is to forward the `winequality.py` module to the YARN containers,
in order for the tasks to be able to import them:

```python
run_on_yarn(
    ...,
    files={
        os.path.basename(winequality.__file__): winequality.__file__,
    }
)
```

Under the hood, the experiment function is shipped to each container, evaluated and then passed to the `train_and_evaluate` function.

```python
experiment = experiment_fn()
tf.estimator.train_and_evaluate(
  experiment.estimator,
  experiment.train_spec,
  experiment.eval_spec
)
```

[linear_classifier_example]: https://github.com/criteo/tf-yarn/blob/master/examples/linear_classifier_example.py
[wine-quality]: https://archive.ics.uci.edu/ml/datasets/Wine+Quality

### standalone_client_mode

[Standalone client mode](https://github.com/tensorflow/tensorflow/blob/r1.13/tensorflow/contrib/distribute/README.md?#standalone-client-mode) keeps most of the previous concepts. Instead of calling `train_and_evaluate` on each worker, one just spawns the TensorFlow server on each worker and then locally runs `train_and_evaluate` on the client. TensorFlow will take care of sending the graph to each worker. This removes the burden of having to ship manually the experiment function to the containers.

Here is the previous example in Standalone client mode:

```python
from tensorflow.contrib.distribute import DistributeConfig, ParameterServerStrategy
from tf_yarn import standalone_client_mode, TaskSpec

with standalone_client_mode(
     task_specs={
       "worker": TaskSpec(memory=4 * 2**10, vcores=4, instances=2),
       "ps": TaskSpec(memory=4 * 2**10, vcores=4, instances=1)}
) as cluster_spec:
    distrib_config = DistributeConfig(
      train_distribute=ParameterServerStrategy(),
      remote_cluster=cluster_spec
    )
    estimator = tf.estimator.DNNClassifier(
      ...
      config=tf.estimator.RunConfig(
        experimental_distribute=distrib_config
      )
    )

    tf.estimator.train_and_evaluate(
      estimator,
      tf.estimator.TrainSpec(...),
      tf.estimator.EvalSpec(...))
```

`standalone_client_mode`  takes care of creating the `ClusterSpec` as described before. We activate `ParameterServerStrategy` in the `RunConfig` and then call `train_and_evaluate`.

In addition to training estimators, Standalone client mode also gives access to TensorFlow’s low-level API. Have a look at the [examples](https://github.com/criteo/tf-yarn/blob/master/examples/session_run_example.py) for more information.

## Distributed TensorFlow

The following is a brief summary of the core distributed TensorFlow concepts relevant to training [estimators](https://www.tensorflow.org/api_docs/python/tf/estimator/train_and_evaluate) with the ParameterServerStrategy, as it is the distribution strategy activated by default when training Estimators on multiple nodes.

Distributed TensorFlow operates in terms of tasks.
A task has a type which defines its purpose in the distributed TensorFlow cluster:
- `worker` tasks headed by the `chief` doing model training
- `chief` task additionally handling checkpoints, saving/restoring the model, etc.
- `ps`  tasks (aka parameter servers) storing the model itself. These tasks typically do not compute anything.
Their sole purpose is serving the model variables
- `evaluator` task periodically evaluating the model from the saved checkpoint

The types of tasks can depend on the distribution strategy, for example, ps tasks are only used by ParameterServerStrategy.
The following picture presents an example of a cluster setup with 2 workers, 1 chief, 1 ps and 1 evaluator.

```
+-----------+              +---------+   +----------+   +----------+
| evaluator |        +-----+ chief:0 |   | worker:0 |   | worker:1 |
+-----+-----+        |     +----^----+   +-----^----+   +-----^----+
      ^              |          |            |              |
      |              v          |            |              |
      |        +-----+---+      |            |              |
      |        | model   |   +--v---+        |              |
      +--------+ exports |   | ps:0 <--------+--------------+
               +---------+   +------+
```

The cluster is defined by a ClusterSpec, a mapping from task types to their associated network addresses. For instance, for the above example, it looks like that:

```
{
  "chief": ["chief.example.com:2125"],
  "worker": ["worker0.example.com:6784",
             "worker1.example.com:6475"],
  "ps": ["ps0.example.com:7419"],
  "evaluator": ["evaluator.example.com:8347"]
}
```
Starting a task in the cluster requires a ClusterSpec. This means that the spec should be fully known before starting any of the tasks.

Once the cluster is known, we need to export the ClusterSpec through the [TF_CONFIG](https://cloud.google.com/ml-engine/docs/tensorflow/distributed-training-details) environment variable and start the TensorFlow server on each container.

Then we can run the [train-and-evaluate](https://www.tensorflow.org/api_docs/python/tf/estimator/train_and_evaluate) function on each container.
We just launch the same function as in local training mode, TensorFlow will automatically detect that we have set up a ClusterSpec and start a distributed learning.

You can find more information about distributed Tensorflow [here](https://github.com/tensorflow/examples/blob/master/community/en/docs/deploy/distributed.md) and about distributed training Estimators [here](https://www.tensorflow.org/api_docs/python/tf/estimator/train_and_evaluate).

## Training with multiple workers

Activating the previous example in tf-yarn is just changing the cluster_spec by adding the additional `worker` and `ps` instances: 

```python
run_on_yarn(
    ...,
    task_specs={
        "chief": TaskSpec(memory="2 GiB", vcores=4),
        "worker": TaskSpec(memory="2 GiB", vcores=4, instances=2),
        "ps": TaskSpec(memory="2 GiB", vcores=8),
        "evaluator": TaskSpec(memory="2 GiB", vcores=1),
        "tensorboard": TaskSpec(memory="2 GiB", vcores=1)
    }
)
```

## Configuring the Python interpreter and packages

tf-yarn needs to ship an isolated virtual environment to the containers.

You can use the packaging module to generate a package on hdfs based on your current installed Virtual Environment.
(You should have installed the dependencies from `requirements.txt` first `pip install -r requirements.txt`)
This works if you use Anaconda and also with [Virtual Environments](https://docs.python.org/3/tutorial/venv.html).

By default the generated package is a [pex][pex] package. The packaging module will generate the pex package, upload it to hdfs and you can start tf_yarn by providing the hdfs path.

```python
pyenv_zip_path, env_name = packaging.upload_env_to_hdfs()
run_on_yarn(
    pyenv_zip_path=pyenv_zip_path
)
```

If you hosting evironment is Anaconda `upload_env_to_hdfs` the packaging module will use [conda-pack][conda-pack] to create the package.

You can also directly use the command line tools provided by [conda-pack][conda-pack] and [pex][pex] to generate the packages.

For pex you can run this command in the root directory to create the package (it includes all requirements from setup.py)
```
pex . -o myarchive.pex
```

You can then run tf-yarn with your generated package:

```python
run_on_yarn(
    pyenv_zip_path="myarchive.pex"
)
```

[conda-pack]: https://conda.github.io/conda-pack/
[pex]: https://pex.readthedocs.io/en/stable/

## Running on GPU

YARN does not have first-class support for GPU resources. A common workaround is
to use [node labels][node-labels] where CPU-only nodes are unlabelled, while
the GPU ones have a label. Furthermore, in this setting GPU nodes are
typically bound to a separate queue which is different from the default one.

Currently, tf-yarn assumes that the GPU label is ``"gpu"``. There are no
assumptions on the name of the queue with GPU nodes, however, for the sake of
example we wil use the name ``"ml-gpu"``.

The default behaviour of `run_on_yarn` is to run on CPU-only nodes. In order
to run on the GPU ones:

1. Set the `queue` argument.
2. Set `TaskSpec.label` to `NodeLabel.GPU` for relevant task types.
   A good rule of a thumb is to run compute heavy `"chief"` and `"worker"`
   tasks on GPU, while keeping `"ps"` and `"evaluator"` on CPU.
3. Generate two python environements: one with Tensorflow for CPUs and one
   with Tensorflow for GPUs. You need to provide a custom path in archive_on_hdfs
   as the default one is already use by the CPU pyenv_zip_path

```python
import getpass
from tf_yarn import NodeLabel
from tf_yarn import packaging

pyenv_zip_path_cpu, _ = packaging.upload_env_to_hdfs()
pyenv_zip_path_gpu, _ = packaging.upload_env_to_hdfs(
    archive_on_hdfs=f"{packaging.get_default_fs()}/user/{getpass.getuser()}/envs/tf_yarn_gpu_env.pex",
    additional_packages={"tensorflow-gpu", "2.0.0a0"},
    ignored_packages={"tensorflow"}
)
run_on_yarn(
    {NodeLabel.CPU: pyenv_zip_path_cpu, NodeLabel.GPU: pyenv_zip_path_gpu}
    experiment_fn,
    task_specs={
        "chief": TaskSpec(memory="2 GiB", vcores=4, label=NodeLabel.GPU),
        "evaluator": TaskSpec(memory="1 GiB", vcores=1),
        "tensorboard": TaskSpec(memory="1 GiB", vcores=1)
    },
    queue="ml-gpu"
)
```

[node-labels]: https://hadoop.apache.org/docs/stable/hadoop-yarn/hadoop-yarn-site/NodeLabel.html

## Accessing HDFS in the presence of [federation][federation]

`skein` the library underlying `tf_yarn` automatically acquires a delegation token
for ``fs.defaultFS`` on security-enabled clusters. This should be enough for most
use-cases. However, if your experiment needs to access data on namenodes other than
the default one, you have to explicitly list them in the `file_systems` argument
to `run_on_yarn`. This would instruct `skein` to acquire a delegation token for
these namenodes in addition to ``fs.defaultFS``:

```python
run_on_yarn(
    ...,
    file_systems=["hdfs://preprod"]
)
```

Depending on the cluster configuration, you might need to point libhdfs to a
different configuration folder. For instance:

```python
run_on_yarn(
    ...,
    env={"HADOOP_CONF_DIR": "/etc/hadoop/conf.all"}
)
```

[federation]: https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/Federation.html

## Tensorboard

You can use Tensorboard with TF Yarn.
Tensorboard is automatically spawned when using a default task_specs. Thus running as a separate container on YARN.
If you use a custom task_specs, you must add explicitly a Tensorboard task to your configuration.

```python
run_on_yarn(
    ...,
    task_specs={
        "chief": TaskSpec(memory="2 GiB", vcores=4),
        "worker": TaskSpec(memory="2 GiB", vcores=4, instances=8),
        "ps": TaskSpec(memory="2 GiB", vcores=8),
        "evaluator": TaskSpec(memory="2 GiB", vcores=1),
        "tensorboard": TaskSpec(memory="2 GiB", vcores=1, instances=1, termination_timeout_seconds=30)
    }
)
```

Both instances and termination_timeout_seconds are optional parameters.
* instances: controls the number of Tensorboard instances to spawn. Defaults to 1
* termination_timeout_seconds: controls how many seconds each tensorboard instance must stay alive after the end of the run. Defaults to 30 seconds

The full access URL of each tensorboard instance is advertised as a _url_event_ starting with "Tensorboard is listening at...".
Typically, you will see it appearing on the standard output of a _run_on_yarn_ call.

### Environment variables
The following optional environment variables can be passed to the tensorboard task:
* TF_BOARD_MODEL_DIR: to configure a model directory. Note that the experiment model dir, if specified, has higher priority. Defaults: None
* TF_BOARD_EXTRA_ARGS: appends command line arguments to the mandatory ones (--logdir and --port): defaults: None 
