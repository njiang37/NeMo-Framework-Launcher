# BigNLP-Scripts

Scripts and code to provide end-to-end data preparation and training for
Megatron-LM.

## Table of contents
  - [Model Overview](#model-overview)
  - [Installation](#installation)
  - [General Configuration](#general-configuration)
  - [Data Preparation](#data-preparation)
  - [GPT-3 Training](#gpt-3-training)
  - [Model Evaluation](#model-evaluation)
  - [Deploying the BigNLP model](#deploying-the-bignlp-model)
    - [Model inference deployment process](#model-inference-deployment-process)
    - [1. Prepare environment](#1-prepare-environment)
      - [1.1 Slurm cluster](#21-slurm-cluster)
      - [1.2 BCP cluster](#12-bcp-cluster)
    - [2. Provide model and inference configuration](#2-provide-model-and-inference-configuration)
      - [2.1 Predefined configuration for selected models](#21-predefined-configuration-for-selected-models)
      - [2.2. Optimal configuration search](#22-optimal-configuration-search)
        - [2.2.1 Random weights checkpoint benchmark](#221-random-weights-checkpoint-benchmark)
        - [2.2.2. Trained checkpoint benchmark](#222-trained-checkpoint-benchmark)
      - [2.3. Review deployment search results](#23-review-deployment-search-results)
    - [3. Prepare NVIDIA Triton Model Repository and run accuracy / performance tests](#3-prepare-nvidia-triton-model-repository-and-run-accuracy-performance-tests)
    - [4. Run NVIDIA Triton Server with selected Model Repository](#4-run-nvidia-triton-server-with-selected-model-repository)
    - [5. Text generation scripts](#5-text-generation-scripts)
  - [Performance](#performance)
    - [Benchmarking](#benchmarking)
    - [Results](#results)
      - [Training Accuracy Results](#training-accuracy-results)
      - [Training Performance Results](#training-performance-results)
      - [Inference performance](#inference-performance)
        - [5B model](#5b-model)
          - [5B: Chatbot question for answering](#5b-chatbot-for-question-answering)
          - [5B: Translation and style transfer](#5b-translation-and-style-transfer)
          - [Summary for 5B results](#summary-for-5b-results)
        - [20B model](#20b-model)
          - [20B: Chatbot question for answering](#20b-chatbot-for-question-answering)
          - [20B: Translation and style transfer](#20b-translation-and-style-transfer)
          - [Summary for 20B results](#summary-for-20b-results)
        - [Model size and performance](#model-size-and-performance)
          - [Online scenario](#online-scenario)
          - [Offline scenario](#offline-scenario)

## Installation:
To be able to call the necessary scripts from the login node on a cluster, some
packages must be installed using the requirements.txt file:
        cd bignlp-scripts
        pip3 install --user -r requirements.txt

## General Configuration
The first parameter that must be set is the bignlp_path parameter inside the
conf/config.yaml file, which must point to the absolute path where this
bignlp-scripts repository is stored in the file system. For BCP, this hydra
scripts repository can be stored in a workspace mounted to /workspace-scripts in 
the container and bignlp_path can be set to "/workspace-scripts/bignlp-scripts".

## Model Overview

NeMo Megatron is a new version in the NeMo framework that allows developers to effectively train and scale language models to billions of parameters. With NeMo Megatron, you can train different variants of GPT-3 models and scale them to multiple nodes on superpods. This deep learning software stack is  optimized for NVIDIA DGX A100 SuperPODs using NVIDIA InfiniBand to provide efficient on-premises compute for training and inferring  complex workloads.

Early access to NeMo Megatron is limited to enterprises that want to train and deploy GPT-3 style models on NVIDIA A100 SuperPOD to perform zero-shot tasks such as answering deep domain questions, translating languages, comprehending and summarizing complex documents. 


GPT-3 architecture

<img src="img/model_overview.png"/>

Figure1: The model includes 24 transformer layers, a hidden size of 4096, and 32 attention heads. The sequence length is 2048, and the optimizer is Adam. This model uses tensor parallelism of 2.


Main layers would be parallelized:
* ColumnParallelLinear
* RowParallelLinear
* ParallelMLP
* ParallelSelfAttention


## Feature matrix

| Feature                         | Training               | Inference                                                                                                                                                         |
| ------------------------------- | ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Data Parallelism                | Yes                    |                                                                                                                                                                   |
| Tensor Parallelism              | Yes                    | Yes                                                                                                                                                               |
| Pipeline Parallelism            | No                     | Yes (for Megatron checkpoints)                                                                                                                                    |
| Gradient checkpointing          | Yes                    |                                                                                                                                                                   |
| Partial gradient checkpointing  | Yes                    |                                                                                                                                                                   |
| FP32/TF32                       | Yes                    | Yes (FP16 enabled by default)                                                                                                                                     |
| AMP/FP16                        | Yes (Model Size <= 5B) | Yes                                                                                                                                                               |
| BF16                            | Yes (Model Size > 5B)  | No                                                                                                                                                                |
| Multi GPU                       | Yes                    | Yes                                                                                                                                                               |
| Multi Node                      | Yes                    | Yes                                                                                                                                                               |
| Inference deployment            | N/A                    | [NVIDIA Triton supported](https://github.com/triton-inference-server/backend#where-can-i-find-all-the-backends-that-are-available-for-triton), Faster Transformer |
| SW stack support                | SLURM/BCM/PCP          | SLURM/BCM/PCP                                                                                                                                                     |
| Distributed data pre processing | Yes (Piles only)       |                                                                                                                                                                   |
| NVfuser                         | Yes (FP16)             |                                                                                                                                                                   |



## Setup

### Support matrix

| Software          | EA                 |
| ----------------- | ------------------ |
| NVIDIA Triton     | 2.15.0             |
| FasterTransformer | V4                 |
| PyTorch           | 1.10.0a0+0aef44c   |
| NeMo              | 1.5.0              |
| PyTorch Lightning | 1.5.0              |
| Hydra             | 1.1.1              |
| CUDA              | NVIDIA CUDA 11.4.2 |
| cuBLAS            | 11.6.5.2           |
| cuDNN             | 8.2.4.15           |
| NCCL              | 2.11.4             |
| Container OS      | Ubuntu 20.04       |
| rdma-core         | 36.0               |
| GDRcopy           | 2.3                |
| HPC-X             | 2.9.0              |
| BCM               | 1.0.0              |


## Quick Start Guide
### Training BigNLP Models

#### 1. Prepare the environment

The whole solution uses a set of Docker containers executed on at the Slurm
cluster using the [pyxis](https://github.com/NVIDIA/pyxis) plug-in. The training container also includes conversion scripts and NVIDIA Triton Model Navigator. The inference container comprises the NVIDIA Triton Inference Server with the FasterTransformer backend installed.

The bignlp-scripts codebase is included as part of the training container. To copy it to a local directory in the cluster, it needs to be extracted from the container. To copy the code to a directory named /path/to/local/dir the following command can be executed.

```
srun -p [partition] -N 1 --container-mounts=/path/to/local/dir:/workspace/mount_dir --container-image=[container_tag] bash -c "cp -r /opt/bignlp/bignlp-scripts /workspace/mount_dir/"
```

Install the BigNLP scripts dependencies on the head node of your cluster:

```
pip install -r requirements.txt
```
You can use virtualenv to prevent polluting your head node environment for
other Python projects. If your Slurm configuration lacks pip, then you can
install pip using use [get_pip.py](https://github.com/pypa/get-pip) with just `python3`.

##### 1.1. General Configuration

The first parameter that must be set is the `bignlp_path` parameter inside the `conf/config.yaml` file. This parameter must point to the absolute path where the 
`bignlp-scripts` repository is stored in the file system.
Every other path or directory in all the config files can either be an absolute
or a relative path. Every path starting with the “/” symbol will be considered
an absolute path, and everything else will be treated as a relative path, and
the path indicated in the `bignlp_path` parameter of the `conf/config.yaml` file
will be appended to the beginning of each relative path. 
The `bignlp_path` parameter will automatically be mounted to the container at the
same path as in the local file system. Any additional directories that should
be mounted must be specified using the `container_mounts` parameter. All the
paths will be mounted to the same path inside and outside the container.
The `data_dir` parameter can also be modified to point to where the dataset will
be loaded from or saved. Please note that if a directory outside of the
`bignlp_path` parameter is used, the directory must also be mounted using the
`container_mounts` parameter.

`Main.py` is the main file that needs to be executed to run both the data
preparation, training, and evaluation pipelines. Each of these pipelines has a
parameter in the `conf/config.yaml` file that decides whether to run that
pipeline or not. To run these pipelines execute:

```
python3 main.py
```

The entire repository uses `hydra/omegaconf` to handle job configuration using
YAML files, so look at the documentation for those projects to learn more.

#### 2. Data Preparation
We provide utilities to download and prepare
[The Pile](https://pile.eleuther.ai/)
dataset, which is formed by 22 smaller datasets. The dataset is already blended
by using the mix described in their [paper](https://arxiv.org/pdf/2101.00027.pdf).
It is recommended to store this repository and the datasets in a file system
shared by all the nodes (gpfs).

The configuration used for data preparation must be specified in the
`conf/config.yaml` file and `run_data_preparation` must be set to `True` to run it.
The `data_preparation` parameter specifies which file to use for data preparation
configuration purposes. The default value is set to `download_pile`, which can be
found in `conf/data_preparation/download_pile.yaml`. The parameters can be
modified to perform the different tasks and to decide where to store the
datasets, vocab, and merge files.

The Pile dataset consists of 30 shards, and downloading, extracting and
preprocessing each file takes approximately 1 hour assuming a 30MB/s download
speed. The data preparation can be parallelized by using up to 30 nodes to download all 30 files in parallel. To download a reduced portion of the dataset to run tests, the
`file_numbers` parameter can be updated to download only one of the shards by
changing “0-29” to “0”.

Set configuration for a Slurm cluster in the YAML file:

```yaml
slurm:                    # example config for enterprise cluster
    account: null         # slurm account
    partition: ???        # slurm partition
    time_limit: “2:00:00” # slurm time parameter
    Nodes: 30             # number of nodes to use per task
```

Set the configuration for the data preparation job in the YAML file:

```yaml
download_the_pile: True        # Whether to download the pile dataset from the internet.
file_numbers: "0-29"           # The pile dataset consists of 30 files (0-29), choose which ones to download.
preprocess_data: True          # True to preprocess the data using the megatron code, False otherwise.
download_vocab_url: "https://huggingface.co/gpt2/resolve/main/vocab.json"    # URL to download the vocab from.
download_merges_url: "https://huggingface.co/gpt2/resolve/main/merges.txt"   # URL to download the merges from.
vocab_save_dir: ${bignlp_path}/data_preparation/bpe       # Location to save the vocab file upon downloading
merges_save_dir: ${bignlp_path}/data_preparation/bpe       # Location to save the merge file upon downloading
log_dir: ${bignlp_path}/data_preparation/logs  # Location to save the logs
```

Example:

To run only the data preparation pipeline and run the training, evaluation or
inference pipelines, set the `conf/config.yaml` file to:
```yaml
run_data_preparation: True
run_training: False
run_evaluation: False
```

And then run:
```
python3 main.py
```

### Training with Predefined configs

We provide three configurations of three different model sizes: 126M, 5B and 20B parameters. These configurations include carefully selected hyper-parameters, which should be used as a guideline for any custom model configurations. All these configurations are provided in the `conf/training/`
directory. The desired configuration can be chosen by selecting the training
and the `training_config` parameters in the `conf/config.yaml` file.

**126M configuration:**

The 126M model uses 8 nodes with 8 GPUs per node by default, and fp16 data type
for training. The model includes 12 transformer layers, a hidden size of 768,
    and 12 attention heads. The sequence length is 2048, and the optimizer is
    Adam. This model does not use any model parallelism. For the details on all
    the parameters, see the 126m.yaml config file.

To train a 126M GPT-3 model, modify the conf/config.yaml file to set:
```
- training: 126m
training_config: 126m
run_training: True
```

And run:
```
python3 main.py
```


**5B configuration:**

The 5B model uses 20 nodes with 8 GPUs per node by default, and fp16 data type for training, and can be trained in about one week. The model includes 24 transformer layers, a hidden size of 4096, and 32 attention heads. The sequence length is 2048, and the optimizer is Adam. This model uses tensor parallelism of 2. For the details on all the parameters, see the 5b.yaml config file.

To train a 5B GPT-3 model, modify the `conf/config.yaml` file to set:
```
- training: 5b
training_config: 5b
run_training: True
```

And run:
```
python3 main.py
```


**20B configuration:**

The 20B model uses 80 nodes with 8 GPUs per node by default, and bf16 data type for training, and can be trained in about one week. The model includes 44 transformer layers, a hidden size of 6144, and 48 attention heads. The sequence length is 2048, and the optimizer is Adam. This model uses tensor parallelism of 8. For the details on all the parameters, see the 20b.yaml config file.

To train a 20B GPT-3 model, modify the conf/config.yaml file to set:
```
- training: 20B
training_config: 20B
run_training: True
```

And run:
```
python3 main.py
```

### Training with Custom configs

The training config files can be modified, or other files can be created to be used for training. They should follow the same structure and guidelines as the existing model configurations.

As a guideline, any model of 5B parameters or less should use fp16 as a data
type, whereas any model larger than 5B parameters should use bfloat16 (bf16) as
a data type.


### Bring Your Own Dataset

If you want to train the GPT-3 model on your own dataset (which is already
filtered and cleaned), you must first convert the dataset files to jsonl files.
Then, you can run the data preprocessing pipeline without needing to download
The Pile by modifying the configuration in
`conf/data_preparation/download_pile.yaml`. You should set `download_the_pile` to
False, and keep `preprocess_data` as True. When running the data preparation
pipeline, the jsonl files must be stored in the directory indicated in the
`data_save_dir` parameter. The result will be a preprocessed dataset, stored in
the same directory, and ready to be used for training. To train the dataset on
your dataset, the training config file must be modified with the desired blend
of training datasets, by changing the blend in the `model.data.data_prefix`
parameter.


### GPT-3 Training

We provide an easy-to-use yet powerful pipeline to perform distributed training
of GPT-3 models across multiple nodes and GPUs. We also provide
well-established recipes for different sizes of GPT-3 models, where the
throughput has been maximized, and the convergence properties of the
model have been tested and confirmed.

The configuration used for the training pipeline must be specified in the 
`conf/config.yaml` file, specifying the training parameter, specifying which file
to use for training purposes. The `run_training` parameter must be set to True to
run the training pipeline. The default value is set to 5b, which can be found
in `conf/training/5b.yaml`. The parameters can be modified to adjust the
hyperparameters of the training runs.

Set configuration for a Slurm cluster in the YAML file:

```yaml
Slurm:  # Refer to SLURM documentation for details on each sbatch parameter   
    partition: ???
    account: null
    time_limit: "7-00:00:00"
    nodes: 20
    exclusive: True
    mem: 0
    overcommit: True
    ntasks_per_node: 8
    gpus_per_task: 1
    dependency: "singleton"
    job_name: "bignlp-gpt3:5b"
```

Example:
To run only the training pipeline and not the data preparation, evaluation or inference pipelines, set the conf/config.yaml file to:
```yaml
run_data_preparation: False
run_training: True
run_evaluation: False
```
And then run:
```
python3 main.py
```

### Resuming Training from fewer nodes
To be able to resume a training run with a different number of nodes is to keep
the global batch size unchanged. This ensures that each training step will be
almost the same, regardless of the number of nodes. The global batch size (GBS)
can be calculated as:

```
GBS = (MBS * num_gpus * accumulate_grad_batches) / tensor_parallelism
```

Where MBS is the micro batch size. For instance, the default GBS for the 5B
model is 1280; the MBS is 2; the number of GPUs is 20\*8 = 160; the
accumulate\_grad\_batches is set to 8; and the\ tensor\_parallelism value is set to 2.
The GBS can be calculated like this:

```
1280 = (2 * 160 * 8) / 2
```

To modify the number of nodes to be used, the user should modify the value of
`accumulate\_grad\_batches` in the inverse way. For instance, if the number of
nodes gets cut in half (20 → 10), then the `accumulate\_grad\_batches` should be
doubled (8 → 16).


### Model Evaluation

We also provide a simple tool to help evaluate the trained checkpoints. You can
evaluate the capabilities of the GPT-3 model on the following ZeroShot
downstream evaluation tasks: `lambada`, `boolq`, `race`, `piqa`, `hellaswag`, `winogrande`,
`wikitext2`, and `wikitext103`.

The configuration used for the evaluation needs to be specified in the
`conf/config.yaml` file, specifying the `evaluation` parameter, which specifies the
file to use for evaluation purposes. The `run_evaluation` parameter must be set
to `True` to run the evaluation pipeline. The default value is set to
`evaluate_all`, which can be found in `conf/evaluation/evaluate_all.yaml`. The
parameters can be modified to adapt different evaluation tasks and checkpoints
in evaluation runs.

Set configuration for a Slurm cluster in the YAML file:
```yaml
Slurm:  # Refer to SLURM documentation for details on each sbatch parameter:
    partition: ???
    account: null
    time_limit: "4:00:00"
    nodes: 1
    exclusive: True
    mem: 0
    overcommit: True
    ntasks_per_node: ${evaluation.model.tensor_model_parallel_size}
    gpus_per_task: null
    dependency: "singleton"
    job_name: "bignlp-gpt3:evaluation”
```

To specify the configuration for what tasks to run for evaluation, use the `run.tasks` parameter:
```yaml
run:
    name: eval_tasks
    tasks: all_tasks  # lambada, boolq, race, piqa, hellaswag, winogrande, wikitext2, wikitext103 OR all_tasks
    output_path: ${bignlp_path}/eval_scripts/logs
```

To specify which model checkpoint to load and its definition, use the model parameter:

```yaml
model:
    type: nemo-gpt3
    checkpoint_path: ${bignlp_path}/train_scripts/logs/5b/checkpoints/megatron_gpt.nemo # path to .nemo file
    tensor_model_parallel_size: 2   # Same tensor parallelism as used for training
    eval_batch_size: 1  # Batch size to use during evaluation
    vocab_file: ${bignlp_path}/data_preparation/bpe/vocab.json
    merge_file: ${bignlp_path}/data_preparation/bpe/merges.txt
```

**Example:**

To run only the evaluation pipeline and not the data preparation, training or
inference pipelines set the `conf/config.yaml` file to:
```yaml
run_data_preparation: False
run_training: False
run_evaluation: True
```

then run:
```
python3 main.py
```


## Deploying the BigNLP model


This section describes the deployment of the BigNLP model on the NVIDIA Triton
Inference Server with FasterTransformer Backend on both single and multiple
node environments.  NVIDIA Triton Inference Server supports many inference
scenarios, of which two most important are:
* Offline inference  scenario - with a goal to maximize throughput regardless
  of the latency, usually achieved with increasing batch size and using server
  static batching feature.
* Online inference scenario - with a goal to maximize throughput within a given
  latency budget, usually achieved with small batch sizes and increasing
  concurrency requests to the server, using dynamic batching feature.

[NVIDIA Triton Model Navigator](https://github.com/triton-inference-server/model_navigator)
helps with conversion and setting up a deployment environment to do inference
for models from BigNLP training scripts. Use scripts to convert models to a new
format, then use NVIDIA Triton Inference Server to process inference requests.

The inference scripts execute at a Slurm or BCP cluster in several steps:
* Megatron/NeMo checkpoint conversion to FasterTransformer format.
* Preparation of model repository for NVIDIA Triton Inference Server.
* Profiling and selecting the best inference model and NVIDIA
  Triton Inference Server configuration.
* Accuracy verification.
* Profiling of deployed models.

The inference container is pulled from a Docker registry. You must ensure that
your cluster configuration allows access to your registry. NVIDIA provides the
container with all components necessary for inference at the
[NGC Docker registry](https://ngc.nvidia.com/catalog/containers).
Inference scripts use the [pyxis slurm plug-in](https://github.com/NVIDIA/pyxis)
to pull and run the container in a Slurm node.


The navigator script converts a checkpoint from a training format to the
[FasterTransformer](https://github.com/triton-inference-server/fastertransformer_backend)
format. The NVIDIA Triton Model Navigator looks for a trained
checkpoint in the workspace passed as an argument and creates a navigator
workspace with all output files, which can be used for production inference
deployment.

The NVIDIA Triton Model Navigator script generates many NVIDIA Triton model
repositories and manages them to conduct optimization of configuration
parameters. This optimizes GPU memory and makes inference a lot faster. NVIDIA
Triton Inference Server’s optimization tool Model Analyzer helps to find the
best configuration, taking into account constraints defined in the navigator’s
configuration. It is possible to set constraints for latency, number of GPUs
and [NVIDIA DGX A100](https://www.nvidia.com/en-us/data-center/dgx-a100/)
machines. All generated models are profiled to report latency and throughput.
Once the model is optimized, you can deploy it to your inference infrastructure
and use it in production.


### Model inference deployment process

<img src="img/inference_deployment_flow.png"/>


### 1. Prepare environment

The whole solution uses a set of Docker containers executed at Slurm or BCP cluster.
The training container also includes conversion
scripts and NVIDIA Triton Model Navigator. The inference container is just the
NVIDIA Triton Inference Server with the FasterTransformer backend installed.
Install the BigNLP scripts dependencies on the:
  - head node of your Slurm cluster
  - your workstation if running them on BCP cluster

```
pip install -r requirements.txt
```

You can use `virtualenv` to prevent polluting your
head node environment for other Python projects. If your Slurm configuration
lacks pip, then you can use [get\_pip.py](https://github.com/pypa/get-pip)
with just `python3`.

You must set your configuration for a cluster in YAML file.

#### 1.1 Slurm cluster

Sample Slurm cluster configuration file:

```yaml
cluster:                # example config for enterprise cluster
  type: pyxis           # type of job executor to be used
  account: null         # slurm account
  partition: "batch"    # slurm partition
  srun_args: ["--mpi", "pmix"] # additional slurm arguments list
  enable_gpus_allocation: true
env:
  job_name_prefix: "bignlp-"
  training_container_image: nvcr.io/ea-bignlp/bignlp-training:21.12-py3-base
  inference_container_image: nvcr.io/ea-bignlp/bignlp-inference:21.12-py3-base
```

The `cluster` section configures Slurm cluster parameters. The `srun_args`
should contain [--mpi](https://slurm.schedmd.com/mpi_guide.html) parameter
valid for your cluster. `enable_gpus_allocation` parameters controls
sbatch/srun `--gpus[-XXX]` parameters and should be disabled on cluster where
allocation of GPUs is not supported.

The `env` section sets development environment parameters:
 * `job_name_prefix`: Prefix which will be prepended to the name of each queued job.
 * `training_container_image`: NGC training container for BigNLP.
 * `inference_container_image`: NGC inference container for BigNLP.

#### 1.2 BCP cluster

Sample BCP cluster configuration file:
```yaml
cluster:                # example config for enterprise cluster
  type: base_command    # type of job executor to be used
  instance_with_gpu: dgxa100.40g.8.norm
  instance_without_gpu: dgxa100.40g.1.norm 
env:
  job_name_prefix: "bignlp-"
  training_container_image: nvcr.io/ea-bignlp/bignlp-training:21.12-py3-base
  inference_container_image: nvcr.io/ea-bignlp/bignlp-inference:21.12-py3-base
```

The `cluster` section set BCP parameters:
 * `instance_with_gpu`: Instance to be used when Job to be submitted will require GPUs
 * `instance_without_gpu`: Instance to be used when Job to be submitted will not require GPUs

The `env` section sets development environment parameters:
 * `job_name_prefix`: Prefix which will be prepended to the name of each queued job.
 * `training_container_image`: NGC training container for BigNLP.
 * `inference_container_image`: NGC inference container for BigNLP.

When using BCP cluster [workspaces](https://docs.nvidia.com/base-command-platform/user-guide/index.html#managing-workspaces) 
are used to share with Jobs executed on computation node
input data (checkpoints and datasets) and result files (Triton Model Repositories, result files, etc).
Sample structure of workspace:

```
/5b-pile-all-optimize-checkpoint  # directory with Megatron checkpoints
/5b.nemo                          # or Nemo checkpoint file
/lambada                          # datset of accuracy testing
/infer_workspace-20211201_000000  # workspace with results created on each execution of Inference Scripts
```
  
During the execution of Inference Scripts, it is checked if paths to input and output files are placed inside the directory
where the NGC workspace is mounted. The exception is for Model Navigator and cluster config files - 
they are not needed to be shared with Job container or are copied on the workspace by scripts.
Also, the user needs to define the Inference Scripts workspace inside the NGC workspace. 
Example Inference Script call: 

```sh
./infer_scripts/prepare_model_repository \
    --workspace-path <path_where_ngc_workspace_is_mounted>/infer_workspace-$(date +%Y%m%d_%H%M%S) \  # Inference Script must be 
    --cluster-config-path ./conf/inference/cluster_bcp.yaml \  # not needed to be shared with job
    --navigator-config-path ./conf/inference/medium_mbs_128-pp_1-tp_8-io_60_20.yaml \  # will be copied to wokspace by Inference Script
    # other input/output files path should be inside mounted workspace
    --model-path <path_where_ngc_workspace_is_mounted>/5b-pile-all-optimize-checkpoint \
    --triton-model-repository-path <path_where_ngc_workspace_is_mounted>/my_mode_repo_megatron_5b_pp_1_tp_8_io_60_20 \
    ...
```

### 2. Provide model and inference configuration

#### 2.1 Predefined configuration for selected models

The repository contains the conf/inference folder with predefined NVIDIA Triton
Model Navigator configurations saved in YAML files. Those configurations are
prepared for 5B, 20B, 175B and 530B GPT3 models for two input/output
configurations 200/200 and 60/20. The configurations cover inference with
several GPUs at one node.  The files are present in the
`conf/inference/optimal_configurations` folder.

The configuration changes for different input sequence lengths and output
sequence lengths used in inference tasks. An application like chatbot can work
with an input of 60 tokens and an output of 20 tokens. Scenarios like text
translation require much longer lengths closer to 200 for input tokens and 200
for output tokens. The RAM usage for a bigger batch size with longer sequence
lengths increases significantly, so optimal configurations set different
maximum batch size values for different sequence lengths. The predefined
configuration files can be used with the `prepare_model_repository.py` script
described later. The files are marked with a number of parameters in model
architecture like 5B, which means 5 billion parameters.

Input sequence lengths 60 and output 20:
* **5B GPT3**: `5b_io_60_20.yaml`
* **20B GPT3**: `20b_io_60_20.yaml`
* **175B GPT3**: `175b_io_60_20.yaml`
* **530B GPT3**: `530b_io_60_20.yaml`

Input sequence lengths 200 and output 200:
* **5B GPT3**: `5b_io_200_200.yaml`
* **20B GPT3**: `20b_io_200_200.yaml`
* **175B GPT3**: `175b_io_200_200.yaml`
* **530B GPT3**: `530b_io_200_200.yaml`

The configuration folder also contains configuration for random
FasterTransformer checkpoints. It is possible to start FasterTransformer
inference without weight files because the engine just initializes them to
random values. This model can’t deliver any valid accuracy, but it is possible
to benchmark inference constraints like latency before the expensive training
of a large model is finished. The folder `conf/inference/model_specs` contains a
folder with predefined random model configuration, which cover range of example
GPT3 configurations, where each folder is marked with a number of model
parameters:
* **5B**: `5b.ft`
* **20B**: `20b.ft`
* **89B**: `89b.ft`
* **175B**: `175b.ft`
* **310B**: `310b.ft`
* **530B**: `530b.ft`


#### 2.2. Optimal configuration search

##### 2.2.1 Random weights checkpoint benchmark

NVIDIA Triton Model Navigator can benchmark inference before training is
finished and verify inference constraints ahead of time; for example maximum
latency budget or number of GPUs, thus cost of inference. For performance
reasons, if you already know model size and parameters, you can use the
FasterTransformer NVIDIA Triton backend to generate a checkpoint with random
weights inside the NVIDIA Triton Inference Server.

The first step in the benchmark script generates a random checkpoint based on
your configuration. The second step configures model repositories. The third
step starts a set of NVIDIA Triton Inference Servers and executes the
performance measurements for each.

The inputs:
* Random model configuration - For example, `conf/inference/model_specs/5b.ft`
* Docker image with training and profiling scripts.
* Docker image with NVIDIA Triton and FasterTransformer backend.
* Performance profile configuration YAML file.

The outputs:
* Performance report.
* Performance results.
* Optimal configurations.
* NVIDIA Triton model stores with a placeholder for the trained model checkpoint.

You can benchmark a model using
`infer_scripts/profile_model_with_random_weights.py` script:

```
python3 ./infer_scripts/profile_model_with_random_weights.py \
    --cluster-config-path <Your cluster config>.yaml \
    --navigator-config-path ./conf/inference/profile_offline.yaml \
    --model-path conf/inference/model_specs/5b.ft \
    --model-name ft_5B \
    --tensor-parallel-sizes 1 \
    --pipeline-parallel-sizes 1 \
    --input-output-lengths 60,20 \
    --max-batch-sizes 1 \
    --max-latency-ms 100000
```

The parameters:
* `cluster-config-path`: Cluster configuration YAML file.
* `navigator-config-path`: Navigator configuration YAML;
   for example,`./conf/inference/profile_offline.yaml`
* `model-path`: This model path contains a YAML file with
   random checkpoint configuration.
* `model-name`: Your model name for NVIDIA Triton repository.
* `tensor-parallel-sizes`: Tensor parallel factor; for example, `1 2 4 8`
* `pipeline-parallel-sizes`: Pipeline parallel factor; for example, `1 2 3 4`
* `input-output-lengths`: Analyzed input and output lengths in format of
   `<input_len>,<output_len>[ <input_len>,<output_len> …]`;
   for example, `60,20 200,200`
* `max-batch-sizes`: Maximum batch sizes used for optimization;
   for example, `1 2 4 8 16 256`
* `max-latency-ms`: Maximum p99 latency valid for your scenario.
* `top-n-configs`: Number of optimal configurations to save.

The parameters `tensor-parallel-sizes`, `pipeline-parallel-sizes`,
`input-output-lengths`, and `max-batch-sizes` are used to generate combinations of
possible configurations for FasterTransformer and performance measurement
scripts. The profile script compares throughput normalized to 1 GPU of all
generated configurations and prints N-best configurations taking into account a
maximum latency constraint. If you request very small maximum latency, then the
script won’t be able to find any valid configurations.

The repository contains two profile configurations for Model Navigator:
* `conf/inference/profile_offline.yaml` - Configuration for offline scenario
   focusing on changing batch sizes but not user request concurrency.
* `conf/inference/profile_online.yaml` - Configuration for online scenario
   focusing on changing user request concurrency.


The random model configuration for the model-path parameter is in YAML file:
```yaml
decoder_layers: 105  # Number of decoder layers
head_num: 128        # Number of heads in layer
size_per_head: 160   # Size per head
inter_size: 81920    # It can be: inter_size = size_per_head * head_num * 4
tensor_para_size: 8  # Default tensor parallel configuration (ignored)
vocab_size: 51200    # Vocabulary size based on vocabulary file
start_id: 50256      # id of start token in vocabulary
end_id: 50256        # id of end token in vocabulary
```

The output files are saved in the `current_folder/infer_workspace_<YYYYmmdd_HHMMSS>`.
The N best configurations are printed to the terminal.
The `infer_workspace_<YYYYmmdd_HHMMSS>` folder contains CSV file with all
measurements combined:

```
navigator_workspace/analyzer/results/metrics-model-inference.csv
```

The best configuration is selected based on the throughput normalized for one
GPU. It is possible to deploy the same model at a number of GPUs, so the cost
of model deployment is not constant for all configurations. The script
normalizes this cost by dividing throughput of a model instance by the number
of GPUs used for computation.

##### 2.2.2. Trained checkpoint benchmark

As an alternative to generating checkpoints randomly, you can use a trained
checkpoint to look for optimal configuration; however, for larger models that
might take a significant amount of time and might not be feasible.

The inputs:
* Megatron/NeMo trained checkpoint.
* Docker image with training and profiling scripts.
* Docker image with NVIDIA Triton and FasterTransformer backend.
* Performance profile configuration YAML file.

The outputs:
* Performance report.
* Performance results.
* Optimal configurations.
* NVIDIA Triton model stores with trained FasterTransformer model checkpoint.

Model repository preparation for the NVIDIA Triton Inference Server:

```
python3 ./infer_scripts/profile_model.py \
    --cluster-config-path <Your cluster config>.yaml \
    --navigator-config-path ./conf/inference/profile_offline.yaml \
    --model-path <Your path to training checkpoint> \
    --model-name model_name \
    --tensor-parallel-sizes 1 \
    --pipeline-parallel-sizes 1 \
    --input-output-lengths 60,20 \
    --max-batch-sizes 1 \
    --max-latency-ms 100000
```

The parameters:
* `cluster-config-path`: Cluster configuration YAML file.
* `navigator-config-path`: Navigator configuration YAML;
   for example,`./conf/inference/profile_offline.yaml`
* `model-path`: This model path contains a trained Megatron/NeMo checkpoint.
   A NeMo checkpoint must be passed as a file with .nemo extension,
   but a Megatron checkpoint must be passed as a folder.
* `model-name`: Your model name for NVIDIA Triton repository.
* `tensor-parallel-sizes`: Tensor parallel factor; for example, `1 2 4 8`
* `pipeline-parallel-sizes`: Pipeline parallel factor; for example, `1 2 3 4`
* `input-output-lengths`: Analyzed input and output lengths in format of
   `<input_len>,<output_len>[ <input_len>,<output_len> …]`;
   for example, `60,20 200,200`
* `max-batch-sizes`: Maximum batch sizes used for optimization;
   for example, `1 2 4 8 16 256`
* `max-latency-ms`: Maximum p99 latency valid for your scenario.
* `top-n-configs`: Number of optimal configurations to save.

Megatron checkpoint must have embedded vocabulary in PyTorch checkpoint file
or vocabulary file stored in `<model-path>/vocab.json`. Vocabulary embedding can
be performed with `./infer_scripts/embed_vocab_in_megatron_checkpoint.py` script.

The parameters `tensor-parallel-sizes`, `pipeline-parallel-sizes`,
`input-output-lengths`, and `max-batch-sizes` are used to generate combinations of
possible configurations for FasterTransformer and performance measurement
scripts. The profile script compares throughput normalized to 1 GPU of all
generated configurations and prints N-best configurations taking into account a
maximum latency constraint. If you request very small maximum latency, then the
script won’t be able to find any valid configurations.

#### 2.3. Review deployment search results

The `profile_model_with_random_weights.py` and
`profile_model.py` scripts create a folder
`infer_workspace_<YYYYmmdd_HHMMSS>` with a timestamp at the end.

It contains the following folders:
* `model_name-ft_gpu_counts_8-converted.ft`: Folders with converted
   FasterTransformer checkpoints.
* `logs`: Logs.
* `model_repo_model_name-io_60_20-half_1-pp_1-tp_8-mbs_256`:
   NVIDIA Triton model repository for input sequence length 60
   and output length 20 for pipeline parallel 2 and tensor parallel 8
   and maximum batch size 256.
*  `model_repo_model_name-io_60_20-half_1-pp_1-tp_8-mbs_256`:
   NVIDIA Triton model repository for input sequence length 60
   and output length 20 for pipeline parallel 1 and tensor parallel 8 and
   maximum batch size 256.
* `navigator_workspace`: Folder to NVIDIA Triton Model Navigator configurations.
* `cluster_workspace`: Folder with cluster logs and submission scripts.

Both profile scripts print a list of the best models with the name
of the NVIDIA Triton model repository with the best results and performance metrics.

Results from `profile_model.py` and `profile_model_with_random_weights.py`
scripts are saved for review under:
`./infer_workspace-<YYYYmmdd_HHMMSS>/navigator_workspace/analyzer/results/metrics-model-inference.csv`

The CSV file contains several columns:
* `Model` - NVIDIA Triton model name.
* `Batch` - Batch size.
* `Concurrency` - User request concurrency.
* `Model Config Path` - Path to model configuration.
* `Backend Parameters` - Measurement and backend parameters (PP - pipeline
  parallel, TP - tensor parallel, and half - FP16 used for some computations),
  `max_input` - maximum sequence input length, `max_sec` - maximum sequence input
  length plus maximum sequence output length.
* `Preferred Batch Sizes` - List of preferred batch sizes used in NVIDIA Triton configuration.
* `Satisfies Constraints` - “Yes” if a model satisfies the p99 latency constraint, set as the max-latency-ms parameter.
* `Throughput (inder/sec)` - Throughput not normalized for the number of GPUs but just measured for one model instance.
* `p95 Latency(ms)`.
* `p99 Latency(ms)`.

Best configurations are mentioned from the top,
To review configurations, check the directory with all generated configs:
`infer_workspace-<YYYYmmdd_HHMMSS>/navigator_workspace/top_configs`

NVIDIA Triton model repositories contain symbolic links to folders with weights.
You should copy final folder with model to expand links into files.

```
  cp -rL <NVIDIA Triton store from script> <destination>
```


### 3. Prepare NVIDIA Triton Model Repository and run accuracy / performance tests
Having the best config and trained checkpoint. A trained model checkpoint is
required as this is final model deployment and verification. For large models,
loading a checkpoint from storage can take a significant amount of time.

The inputs:
* Trained model checkpoint.
* Docker image with NVIDIA Triton and FasterTransformer backend.
* Lambada dataset.
* Model vocabulary.
* Model merges file.

The English data for accuracy experiments can be downloaded from open resources.

The Lambada dataset can be downloaded from GITHUB:

```
wget https://raw.githubusercontent.com/cybertronai/bflm/master/lambada_test.jsonl
```

The vocabulary and merge files can be downloaded from the Huggingface project:

```
wget https://s3.amazonaws.com/models.huggingface.co/bert/gpt2-vocab.json
wget https://s3.amazonaws.com/models.huggingface.co/bert/gpt2-merges.txt
```

It’s recommended that you put all files in one folder used for accuracy
verification of your model.

The outputs:
* NVIDIA Triton Model Repository with a converted model in FasterTransformer format.
* Accuracy measurement report.
* Performance measurement report.

The accuracy report is stored in the current directory in the file `lambada_metrics.csv`.
You can verify your model running in NVIDIA Triton by using the Lambada dataset:

```
python3 ./infer_scripts/prepare_model_repository.py \
    --cluster-config-path <Your cluster config>.yaml \
    --navigator-config-path ./conf/inference/small_mbs_256-pp_1-tp_1-io_60_20.yaml \
    --model-path <Your path to training checkpoint> \
    --model-name model_name \
    --dataset-dir <Your lambada folder> \
    --model-repository-path <Your output path for NVIDIA Triton model repository> \
    --accuracy-tests \
    --performance-tests
```

Parameters:
* `cluster-config-path`: Cluster configuration YAML file.
* `navigator-config-path`: Navigator configuration to set up NVIDIA Triton.
* `model-path`: This model path contains a trained Megatron/NeMo checkpoint.
   A NeMo checkpoint must be passed as a file with .nemo extension,
   but a Megatron checkpoint must be passed as a folder.
* `model-name`: Model name.
* `dataset-dir`: Folder with downloaded lambada dataset, merges and vocabulary files.
* `model-repository-path`: Path to result NVIDIA Triton Model Repository.
* `accuracy-tests`: Run accuracy tests.
* `performance-tests`: Run performance offline and online tests.

Megatron checkpoint must have embedded vocabulary in PyTorch checkpoint file
or vocabulary file stored in `<model-path>/vocab.json`. Vocabulary embedding can
be performed with `./infer_scripts/embed_vocab_in_megatron_checkpoint.py` script.

The parameter `navigator-config-path` contains the Navigator configuration to
convert a model, set up a NVIDIA Triton, and parameters to perform performance
tests. You must set some basic parameters to have a working model to verify
accuracy. You can use a predefined configuration for this task, which sets
basic values for a tiny model:

```
./conf/inference/small_mbs_256-pp_1-tp_1-io_60_20.yaml
```

You must check your model size and look for optimal configuration to run
accuracy for your model. The larger models must be run with many GPUs and nodes
to work. The predefined configurations for some GPT3 architectures and
inference tasks are described in the _Predefined configurations_ section above.

### 4. Run NVIDIA Triton Server with selected Model Repository

The inputs:
* NVIDIA Triton model repository with FasterTransformer checkpoint
   ready for inference at production.
* Docker image with NVIDIA Triton and FasterTransformer backend.

The outputs:
* Running NVIDIA Triton model instance serving model in cluster.

To run the NVIDIA Triton Model Navigator, do the following:
```
python3 ./infer_scripts/run_tritonserver.py \
    --cluster-config-path <Your cluster config>.yaml \
    --model-repository-path <Your output path for NVIDIA Triton model repository>
```

The parameters:
* `cluster-config-path`: Cluster configuration YAML file.
* `model-repository-path`: NVIDIA Triton model repository path from folder
   generated by `prepare_model_repository.py` script.

The NVIDIA Triton model repository created in scripts above contains symbolic
links. You need to expand links for `run_tritonserver.py` to
be able to access files when they are mounted in job containers.

The script saves NVIDIA Triton logs so you can verify what happens when
FasterTransformer loads a checkpoint. The command above starts the server, so
that users can test it with other tools created later. You can use this
script to demo inference. The job does not stop on its own, if you don't stop it
manually, it will stop when the time limit is reached on the cluster.

FasterTransformer backend ignores missing files for weights and uses random
tensors in such a scenario. You should make sure that your NVIDIA Triton
instance is serving requests with real weights by inspecting logs.


If you notice warning about missing files, you should double check your model:

```
[WARNING] file /triton-model-repository/model_name/1/1-gpu/model.wpe.bin cannot be opened, loading model fails!
[WARNING] file /triton-model-repository/model_name/1/1-gpu/model.wte.bin cannot be opened, loading model fails!
[WARNING] file /triton-model-repository/model_name/1/1-gpu/model.final_layernorm.bias.bin cannot be opened, loading model fails!
[WARNING] file /triton-model-repository/model_name/1/1-gpu/model.final_layernorm.weight.bin cannot be opened, loading model fails!
```

### 5. Text generation scripts

#### 5.1 Setup

You must start BigNLP training container with interactive session at your cluster.
You can do it with `srun` at slurm:

```
srun --partition=<SLURM PARTITION> \
    --container-workdir /bignlp_workdir \
    --container-image <TRAINING CONTAINER DOCKER IMAGE> \
    --container-mounts <FOLDER WITH BIGNLP SCRIPTS>:/bignlp_workdir \
    --pty bash
```

You must ensure that a vocabulary (`vocab.json`) and merge (`merges.txt`) files are accessible at a compute
node so you can pass the folder with those files as parameter for 
scripts described below.

You need working instance of Triton Inference Server with loaded
FasterTransformer model converted from real checkpoint. You can use
`run_tritonserver.py` script described above to start an inference machine.

#### 5.2 Basic text generation

The simple implementation of text input script was prepared
as Python command line client script `infer_scripts/chatbot.py`.
You can run it to send a simple request:
```
python3  infer_scripts/chatbot.py \
    --url <TRITON CLUSTER NODE>:<PORT> \
    --protocol <PROTOCOL> \
    --datasets-dir <FOLDER WITH MERGES AND VOCABULARY> \
    --model-name <MODEL NAME> \
    --output-len <REQUESTED OUTPUT LEN> \
    --query "<TEXT QUERY>"
```

Parameters:
* `url`: Triton URL. It is printed by `run_tritonserver.py` script.
* `protocol`: Communication protocol (for example GRPC, HTTP).
* `dataset-dir`: Folder with downloaded merges and vocabulary files.
* `model-name`: Model name.
* `output-len`: Token sequence output length.
* `query`: Text sent to model as a query.


The script will print out FasterTransformer output:
```
$ python3  infer_scripts/chatbot.py --url triton-node:8001 --protocol grpc \
    --datasets-dir /bignlp_workdir/data/ \
    --model-name 20B_mega_real --output-len 40 \
    --query "A car is"
 a vehicle that can be driven by one person.

The word "car" comes from the French word for chariot, which was used to describe the first cars in the late 19th century. The first
$
```

You can change `output-len` to generate longer sequences, but a quality of output 
from a small checkpoint degrades significantly when length is increased.

#### 5.3 Longer text generation

The script `author.py` was created to generate longer texts. It passes
an output from a previous inference to model again and asks FasterTransformer to generate more text.
The issue with this approach, is that a context of previous requests is lost quite fast and a model
forgets, what it outputted before.


```
python3  infer_scripts/author.py \
    --url <TRITON CLUSTER NODE>:<PORT> \
    --protocol <PROTOCOL> \
    --datasets-dir <FOLDER WITH MERGES AND VOCABULARY> \
    --model-name <MODEL NAME> \
    --output-len <REQUESTED OUTPUT LEN> \
    --query "<TEXT QUERY>"
```

Parameters:
* `url`: Triton URL. It is printed by `run_tritonserver.py` script
* `protocol`: Communication protocol (for example grpc, http)
* `dataset-dir`: Folder with downloaded dataset, merges and vocabulary files.
* `model-name`: Model name.
* `output-len`: Token sequence output length.
* `query`: Text sent to model as a query.

You can pass the text _AI is like a new steam engine_ to `author.py` to generate few paragraphs of text:

```
$ python3  infer_scripts/author.py --url triton-node:8001 --protocol grpc \
    --datasets-dir /bignlp_workdir/data/ \
    --model-name 20B_mega_real \
    --output-len 40
    --query "AI is like a new steam engine."
 It’s not just about the technology, it’s also about how we can use AI to solve problems that are important for society and our economy.

The first thing I want to do is talk a little bit about what we mean by artificial intelligence (AI).

What is Artificial Intelligence?

Artificial intelligence is defined as “the ability of machines to perform tasks that normally require human intelligence.” This definition is broad and can be applied in many different ways, but it does not necessarily mean that the machine will actually think like a person. For example, a computer program may have been trained to recognize images of cats or dogs by analyzing millions of pictures. The program has learned how to identify these animals based on their features, such as ears, eyes^CKeyboard handler detected with signal
$
```
You can interrupt text generation by using `Ctrl+C`.

The `author.py` script uses output from previous query to generate more text.
The table below shows examples of input and output used for text generated above.

| Input len | Input text | Output len | Output text |
| --------- | ---------- | ---------- | ----------- |
| 8 | b'AI is like a new steam engine.' | 40 | b' It\\u2019s not just about the technology, it\\u2019s also about how we can use AI to solve problems that are important for society and our economy.\\n\\nThe first thing I want' |
| 40 | b'It\\u2019s not just about the technology, it\\u2019s also about how we can use AI to solve problems that are important for society and our economy.\\n\\nThe first thing I want' | 40 | b' to do is talk a little bit about what we mean by artificial intelligence (AI).\\n\\nWhat is Artificial Intelligence?\\n\\nArtificial intelligence is defined as \\u201cthe ability of machines to perform' |
| 40 | b'to do is talk a little bit about what we mean by artificial intelligence (AI).\\n\\nWhat is Artificial Intelligence?\\n\\nArtificial intelligence is defined as \\u201cthe ability of machines to perform' | 40 | b' tasks that normally require human intelligence.\\u201d This definition is broad and can be applied in many different ways, but it does not necessarily mean that the machine will actually think like a person. For example' |
| 41 | b'tasks that normally require human intelligence.\\u201d This definition is broad and can be applied in many different ways, but it does not necessarily mean that the machine will actually think like a person. For example' | 40 | b', a computer program may have been trained to recognize images of cats or dogs by analyzing millions of pictures. The program has learned how to identify these animals based on their features, such as ears, eyes' |

#### 5.4 Dialogue text generation

The `dialogue.py` script was created to showcase text generation for a simple
support chatbot dialogue scenario:

```
python3  infer_scripts/dialogue.py \
    --url <TRITON CLUSTER NODE>:<PORT> \
    --protocol <PROTOCOL> \
    --datasets-dir <FOLDER WITH MERGES AND VOCABULARY> \
    --model-name <MODEL NAME> \
    --output-len <REQUESTED OUTPUT LEN> \
    --customer "<TEXT CONTEXT FOR CUSTOMER ROLE>"
    --support "<TEXT CONTEXT FOR SUPPORT ROLE>"
```

Parameters:
* `url`: Triton URL. It is printed by `run_tritonserver.py` script
* `protocol`: Communication protocol (for example grpc, http)
* `dataset-dir`: Folder with downloaded dataset, merges and vocabulary files.
* `model-name`: Model name.
* `output-len`: Token sequence output length.
* `customer`: Text used to generate prompt for a customer role.
* `support`: Text used to generate prompt for a support role.

A model needs prompt to be able to generate text useful for chatbot application.
You must tell a machine, that it is working in a support team in your company and
answering questions from customers.

```
$ python3 infer_scripts/dialogue.py --url triton-node:8001 --protocol grpc \
    --datasets-dir /bignlp_workdir/data/ \
    --model-name 20B_mega_real \
    --output-len 40
    --customer "NVIDIA customer:"
    --support "NVIDIA machine learning expert:"
NVIDIA customer:What is machine learning?
NVIDIA machine learning expert: It's a way to make computers do things that they couldn't before.
NVIDIA customer: (END to FINISH): What I need to start experiments with machine learning?
NVIDIA machine learning expert: We can help you get started. We have a free trial of our GPU-accelerated deep learning platform, and we'll be happy to show you how it works.
NVIDIA customer: (END to FINISH): Can AI recognize cats?
NVIDIA machine learning expert: Sure! Let's try that!
NVIDIA customer: (END to FINISH): Can AI generate text?
NVIDIA machine learning expert: Yes, it can. It will take a few minutes to train the model.
NVIDIA customer: (END to FINISH): Is AI dangerous?
NVIDIA machine learning expert: No, not at all! We're just trying to help you make better decisions.
NVIDIA customer: (END to FINISH): END
$

```

The `dialogue.py` script reads an input from console, so you can just type
and press Enter to send a question to a model. It is extended with a prompt to
provide context. When you type _What is machine learning?_ then script sends text query:

```
NVIDIA customer: "What is machine learning?"
NVIDIA machine learning expert:

```

The model generates answer much longer than necessary:

```


"It\'s a way to make computers do things that they couldn\'t before."


------
\jamesblonde
I'm not sure what the point of this article was. It seems
```


A model can recognize from this prompt that quote characters should mark an expected
answer. It generates an answer but with more text because it was asked for 40 output
tokens here. Not all training datasets are good with teaching a model, when text
generation should stop, but they contain many dialogues from books and
support forums. The `author.py` script uses quote characters to wrap input
from user and later uses them to extract answer from a text generated by
a model. You must consider similar output cleaning and post processing
routines to use this solution.

The table below shows examples of input and output used for text generated above.
| Input len | Input text | Output len | Output text |
| --------- | ---------- | ---------- | ----------- |
| 15 | b'NVIDIA customer: "What is machine learning?" NVIDIA machine learning expert: ' | 40 | b'\\n"It\'s a way to make computers do things that they couldn\'t before."\\n\\n------\\njamesblonde\\nI\'m not sure what the point of this article was. It seems' |
| 41 | b'NVIDIA machine learning expert: "It\'s a way to make computers do things that they couldn\'t before." NVIDIA customer: "What I need to start experiments with machine learning?" NVIDIA machine learning expert: ' | 40 | b'\\n"We can help you get started. We have a free trial of our GPU-accelerated deep learning platform, and we\'ll be happy to show you how it works."\\n\\nThe' |
| 56 | b'NVIDIA machine learning expert: "We can help you get started. We have a free trial of our GPU-accelerated deep learning platform, and we\'ll be happy to show you how it works." NVIDIA customer: "Can AI recognize cats?" NVIDIA machine learning expert: ' | 40 | b'\\n"Sure! Let\'s try that!"\\n\\nA:\\n\\nI think the best way is to use a library like Googlec or tens.net has is is moreing with a a' |
| 28 | b'NVIDIA machine learning expert: "Sure! Let\'s try that!" NVIDIA customer: "Can AI generate text?" NVIDIA machine learning expert: ' | 40 | b'\\n"Yes, it can. It will take a few minutes to train the model." NVIDIA customer: "Great! I\'ll wait here for you to finish training your model."\\n\\nA:' |
| 36 | b'NVIDIA machine learning expert: "Yes, it can. It will take a few minutes to train the model." NVIDIA customer: "Is AI dangerous?" NVIDIA machine learning expert: ' | 40 | b'\\n"No, not at all! We\'re just trying to help you make better decisions."\\n\\nA:\\n\\nI think this is an interesting question and I\'m going to try my hand' |



## Performance

### Results
#### Training Accuracy Results
Training accuracy: NVIDIA SuperPOD (20 x 8 x A100 80GB for 5B model)
We evaluated the 126M parameter and 5B parameter models on 8 different language tasks. The results can be found in the table below. All the tasks are provided as part of the evaluation harness, so the user can evaluate any .nemo checkpoint file on all these tasks.

|Task              |Metric            | 126M             | 5B               |
| ---------------- | ---------------- | ---------------- | ---------------- |
|Lambada           |Accuracy          | 38.70%           | 68.93%           |
|                  |PPL               | 25.8             | 4.22             |
|Boolq             |Accuracy          | 56.94%           | 65.29%           |
|Race              |Accuracy          | 28.71%           | 38.66%           |
|                  |Accuracy Norm     | 34.74%           | 41.62%           |
|Piqa              |Accuracy          | 61.21%           | 73.88%           |
|                  |Accuracy Norm     | 61.97%           | 75.40%           |
|Hellaswag         |Accuracy          | 28.48%           | 46.45%           |
|                  |Accuracy Norm     | 29.54%           | 60.85%           |
|Winogrande        |Accuracy          | 50.43%           | 60.77%           |
|Wikitext2         |Word PPL          | 31.35            | 12.36            |
|                  |Byte PPL          | 1.9              | 1.6              |
|                  |Bits per Byte PPL | 0.64             | 0.47             |
|Wikitext103       |Word PPL          | 31.35            | 12.36            |
|                  |Byte PPL          | 1.9              | 1.6              |
|                  |Bits per Byte PPL | 0.64             | 0.47             |

Training the 5B GPT-3 model to convergence takes 6.5 days, and the loss curve can be seen in the figure below:

<img src="img/5B_GPT_3_loss_final.svg"/>

The table below shows the converged training loss, the throughput, and the total time to train for the 5B GPT-3 model, using a given number of GPUs and a given Global Batch Size (GBS).

| \#GPUs | GBS  | Seq Length | \#Tokens | Loss  | Throughput (Tokens/sec) | Time to Train |
| ----- | ---- | ---------- | ------- | ----- | ----------------------- | ------------- |
| 160   | 1280 | 2048       | 300B    | 1.685 | 610,795                 | 156           |


#### Training Performance Results
Training performance: NVIDIA SuperPOD (20 x 8 x A100 80GB for 5B model)

We measured the throughput of training a 5B parameter GPT-3 model on a SuperPOD using a different number of nodes, and we achieved near-linear scaling. For example, when scaling from 1 node to 20 nodes, we achieve 18.83x speedup. The table and chart below show the performance results.

|      |                                 |        |        |        | Nodes  |        |        |
| ---- | ------------------------------- | ------ | ------ | ------ | ------ | ------ | ------ |
|      |                                 | 1      | 4      | 8      | 10     | 16     | 20     |
|      | Tokens per Second               | 32440  | 128450 | 252968 | 314572 | 495452 | 610795 |
| 5B   | Perfect Linear Scaling (Tokens) | 32440  | 129761 | 259522 | 324403 | 519045 | 648806 |
|      | Speed-up                        | 1x     | 3.96x  | 7.8x   | 9.7x   | 15.27x | 18.83x |

<img src="img/5B_GPT_3_throughput.svg"/>


#### Inference performance

The most important factor for NLP model performance is the size of a model. You
can use a smaller model to get faster inference but it will likely degrade your
accuracy.

If you know your model size, then there are two parameters you can vary to find
the best throughput and keep inside a latency budget:
* Number of GPUs used for one instance of your model.
* Batch size used during processing requests.

The same model can be executed with different amounts of GPUs and nodes so the
basic throughput values don't reflect cost of inference like for one GPU model.
A throughput normalized to one GPU is used as a proxy for cost of inference in
graphs and tables below.


The FasterTransformer hardware configuration is described by two parameters:
* Tensor parallel (TP) size - number of GPUs used at each node for computation.
* Pipeline parallel (PP) size - number of nodes used for one instance of model.
The number of GPUs used for computation is determined by multiplying those two
numbers. Only easily divisible parts of the whole DGX machine were considered
during tests so it will be easy to deploy a model in a cluster.

The table below contains a summary of used configurations.

| TP | PP | #GPUs | #Nodes | Max GPU RAM \[GB\] |
| -- | -- | ----- | ------ | ------------------ |
| 1  | 1  | 1     | 1      | 80                 |
| 2  | 1  | 2     | 1      | 160                |
| 4  | 1  | 4     | 1      | 320                |
| 8  | 1  | 8     | 1      | 640                |
| 8  | 2  | 16    | 2      | 1280               |
| 8  | 3  | 24    | 3      | 1920               |
| 8  | 4  | 32    | 4      | 2560               |


##### 5B model

The 5B model can fit into a single A100 80GB GPU. Still FasterTranformer can
run 5B model using tensor parallel splitting of model between multiple GPUs and
pipeline parallel, when different transformer layers are distributed across
many nodes it gives the possibility to utilize different tradeoffs (e.g.
latency vs throughput). You can also consider using several DGX nodes in
SuperPOD as one instance of the FasterTransformer model. You should also
consider an inference task for your application. Some inference tasks require
longer token sequence lengths  for input and for output.

##### 5B Chatbot for question answering

Let’s consider a scenario with a chatbot for question answering. It can be
implemented with FasterTransformer, when sequence length for input tokens is 60
and output length is 20. Two graphs below show how latency and throughput vary,
when a certain number of GPUs is used for inference for batch size=1 and for
batch size=256.


<img src="img/5B_GPT_3_batch_size_1_input_len_60_output_len_20.svg"/>

<img src="img/5B_GPT_3_batch_size_256_input_len_60_output_len_20.svg"/>


If latency achievable at 1-GPU configuration fits within latency budget, then
the best performance can be derived from the graph below, which shows how
latency and throughput change for different batch sizes used for computations.


<img src="img/5B_GPT_3_of_GPU_1_input_len_60_output_len_20.svg"/>

A chatbot with a latency budget within 380 ms can work for batch size=64 and 1
GPU used for computation.


##### 5B: Translation and style transfer

A translation or style transfer inference task requires input length 200 and
output length 200.

<img src="img/5B_GPT_3_batch_size_1_input_len_200_output_len_200.svg"/>

<img src="img/5B_GPT_3_batch_size_256_input_len_200_output_len_200.svg"/>

The graph for 1 GPU with many batch sizes shows what batch size can fit into a
certain latency budget.


<img src="img/5B_GPT_3_of_GPU_1_input_len_200_output_len_200.svg"/>

The graph clearly shows that the translation or style transfer inference task
with latency budget 2000 milliseconds can be deployed using 1 GPU and batch
size = 16.

##### Summary for 5B results

The table below contains performance measurements from all graphs for the 5B
model running in FasterTransformer at NVIDIA DGX A100 80GB.


<details>

<summary>
5B model: Latency and throughput for different number of GPUs and batch sizes.
</summary>

| GPUs | Latency p99                | Normalized throughput to 1 GPU | Latency p99 | Normalized throughput to 1 GPU | Latency p99                  | Normalized throughput to 1 GPU | Latency p99 | Normalized throughput to 1 GPU |
| ---- | -------------------------- | ------------------------------ | ----------- | ------------------------------ | ---------------------------- | ------------------------------ | ----------- | ------------------------------ |
|      | Input len 60 output len 60 |                                |             |                                | Input len 200 output len 200 |                                |             |                                |
|      | BS=256                     |                                | BS=1        |                                | BS=256                       |                                | BS=1        |                                |
| 1    | 1143                       | 224                            | 172         | 5.81                           | 9048                         | 28.3                           | 1623        | 0.616                          |
| 2    | 799                        | 160                            | 126         | 3.95                           | 6018                         | 21.3                           | 1219        | 0.410                          |
| 4    | 529                        | 121                            | 94          | 2.66                           | 3939                         | 16.2                           | 923         | 0.271                          |
| 8    | 436                        | 73                             | 115         | 1.08                           | 3154                         | 10.1                           | 998         | 0.125                          |
| 16   | 327                        | 49                             | 101         | 0.62                           | 2776                         | 5.8                            | 977         | 0.064                          |
| 24   | 273                        | 39                             | 100         | 0.42                           | 2484                         | 4.3                            | 950         | 0.044                          |
| 32   | 284                        | 28                             | 95          | 0.33                           | 2517                         | 3.2                            | 897         | 0.035                          |

</details>

##### 20B model

To improve accuracy a larger model can be used.

##### 20B: Chatbot for question answering

<img src="img/20B_GPT_3_batch_size_1_input_len_60_output_len_20.svg"/>

<img src="img/20B_GPT_3_batch_size_256_input_len_60_output_len_20.svg"/>

<img src="img/20B_GPT_3_of_GPU_1_input_len_60_output_len_20.svg"/>



##### 20B: Translation and style transfer


<img src="img/20B_GPT_3_batch_size_1_input_len_200_output_len_200.svg"/>

<img src="img/20B_GPT_3_batch_size_256_input_len_200_output_len_200.svg"/>

<img src="img/20B_GPT_3_of_GPU_4_input_len_200_output_len_200.svg"/>


##### Summary for 20B results

The table below contains performance measurements from all graphs for the 20B
model running in FasterTransformer at NVIDIA DGX A100 80GB.

<details>

<summary>
20B model: Latency and throughput for different number of GPUs and batch sizes.
</summary>

| GPUs | Latency p99                | Normalized throughput to 1 GPU | Latency p99 | Normalized throughput to 1 GPU | Latency p99                  | Normalized throughput to 1 GPU | Latency p99 | Normalized throughput to 1 GPU |
| ---- | -------------------------- | ------------------------------ | ----------- | ------------------------------ | ---------------------------- | ------------------------------ | ----------- | ------------------------------ |
|      | Input len 60 output len 60 |                                |             |                                | Input len 200 output len 200 |                                |             |                                |
|      | BS=256                     |                                | BS=1        |                                | BS=64,128,256                |                                | BS=1        |                                |
| 1    | 4146                       | 62                             | 560         | 1.78                           | 10772                        | 5.9                            | 5650        | 0.177                          |
| 2    | 2429                       | 53                             | 359         | 1.39                           | 10544                        | 6.1                            | 3548        | 0.141                          |
| 4    | 1592                       | 40                             | 251         | 1.00                           | 10453                        | 6.1                            | 2486        | 0.101                          |
| 8    | 1169                       | 27                             | 230         | 0.54                           | 7909                         | 4.0                            | 2147        | 0.058                          |
| 16   | 923                        | 17                             | 218         | 0.29                           | 7380                         | 2.2                            | 2131        | 0.029                          |
| 24   | 758                        | 14                             | 218         | 0.19                           | 6511                         | 1.6                            | 2123        | 0.020                          |
| 32   | 742                        | 11                             | 224         | 0.14                           | 6200                         | 1.3                            | 2124        | 0.015                          |

</details>

##### Model size and performance
###### Online scenario

An online scenario focuses on the minimization of latency. Large checkpoints
were generated with randomly initialized weights.


<img src="img/Chatbot_Q_A_batch_size_1_input_len_60_output_len_20.svg"/>


<img src="img/Translation_or_style_transfer_batch_size_1_input_len_200_output_len_200.svg"/>

The performance measurements were obtained at NVIDIA DGX A100 80GB nodes.



<details>

<summary>
Performance for different model sizes in online scenario
</summary>

|                         | Len input 60 output 20 |                   |            |                             |                                |                | Len input 200 output 200 |                   |            |                             |                                |                |
| ----------------------- | ---------------------- | ----------------- | ---------- | --------------------------- | ------------------------------ | -------------- | ------------------------ | ----------------- | ---------- | --------------------------- | ------------------------------ | -------------- |
| Parameters number \[B\] | Latency\[ms\]          | Infer/sec per GPU | Batch size | Tensor parallel (GPUs used) | Pipeline parallel (nodes used) | Number of GPUs | Latency\[ms\]            | Infer/sec per GPU | Batch size | Tensor parallel (GPUs used) | Pipeline parallel (nodes used) | Number of GPUs |
| 5B                      | 93                     | 2.68              | 1          | 4                           | 1                              | 4              | 923                      | 0.271             | 1          | 4                           | 1                              | 4              |
| 13B                     | 189                    | 1.32              | 1          | 4                           | 1                              | 4              | 1893                     | 0.132             | 1          | 4                           | 1                              | 4              |
| 20B                     | 251                    | 0.50              | 1          | 8                           | 1                              | 8              | 2230                     | 0.056             | 1          | 8                           | 1                              | 8              |
| 89B                     | 464                    | 0.27              | 1          | 8                           | 1                              | 8              | 4585                     | 0.027             | 1          | 8                           | 1                              | 8              |
| 175B                    | 923                    | 0.14              | 1          | 8                           | 1                              | 8              | 8873                     | 0.014             | 1          | 8                           | 1                              | 8              |
| 310B                    | 1354                   | 0.09              | 1          | 8                           | 1                              | 8              | 13391                    | 0.005             | 1          | 8                           | 2                              | 16             |
| 530B                    | 2118                   | 0.03              | 1          | 8                           | 2                              | 16             | 20936                    | 0.003             | 1          | 8                           | 2                              | 16             |

</details>

###### Offline scenario

The offline scenario focuses on maximum throughput. The two graphs below show
latency and throughput for two tasks. The first one is chatbot questions
answering and a second one is translation or style transfer.


<img src="img/Chatbot_Q_A_batch_size_256_input_len_60_output_len_20.svg"/>


<img src="img/Translation_or_Style_Transfer_batch_size_max_input_len_200_output_len_200.svg"/>

The chatbot scenario can be executed with batch size equal to 256 for all model
sizes so it is possible to utilize computing resources in GPUs.


<details>

<summary>
Performance for different model sizes in offline scenario
</summary>

|                         | Len input 60 output 20 |                   |            |                 |                                |                | Len input 200 output 200 |                   |            |                             |                                |                |
| ----------------------- | ---------------------- | ----------------- | ---------- | --------------- | ------------------------------ | -------------- | ------------------------ | ----------------- | ---------- | --------------------------- | ------------------------------ | -------------- |
| Parameters number \[B\] | Latency\[ms\]          | Infer/sec per GPU | Batch size | Tensor parallel | Pipeline parallel (nodes used) | Number of GPUs | Latency\[ms\]            | Infer/sec per GPU | Batch size | Tensor parallel (GPUs used) | Pipeline parallel (nodes used) | Number of GPUs |
| 5B                      | 1143                   | 224.0             | 256        | 1               | 1                              | 1              | 9047                     | 28.297            | 256        | 1                           | 1                              | 1              |
| 13B                     | 2756                   | 92.9              | 256        | 1               | 1                              | 1              | 13390                    | 9.559             | 256        | 2                           | 1                              | 2              |
| 20B                     | 4145                   | 61.8              | 256        | 1               | 1                              | 1              | 10453                    | 6.123             | 256        | 4                           | 1                              | 4              |
| 89B                     | 2889                   | 22.2              | 256        | 4               | 1                              | 4              | 17815                    | 1.796             | 256        | 8                           | 1                              | 8              |
| 175B                    | 2033                   | 15.7              | 256        | 8               | 1                              | 8              | 16181                    | 0.494             | 64         | 8                           | 1                              | 8              |
| 310B                    | 6768                   | 2.4               | 256        | 8               | 2                              | 16             | 13686                    | 0.018             | 2          | 8                           | 1                              | 8              |
| 530B                    | 8660                   | 1.8               | 256        | 8               | 2                              | 16             | 20936                    | 0.003             | 1          | 8                           | 2                              | 16             |

</details>

## Changelog

**November 2021**
* Early access release
