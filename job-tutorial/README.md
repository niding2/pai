<!--
  Copyright (c) Microsoft Corporation
  All rights reserved.

  MIT License

  Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated
  documentation files (the "Software"), to deal in the Software without restriction, including without limitation
  the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and
  to permit persons to whom the Software is furnished to do so, subject to the following conditions:
  The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

  THE SOFTWARE IS PROVIDED *AS IS*, WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING
  BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND
  NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM,
  DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
  OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
-->


# How to Run a Deep Learning Job

## Introduction

The system supports major deep learning frameworks, including CNTK and TensorFlow, etc. 
It also supports other type of workload through a customized docker image. 
Users need to prepare a config file and submit it for a job submission. 
This guide introduces the details of job submission.


## Prerequisites

This guide assumes the system has already been deployed properly and a docker registry is available to store docker images. 


## Docker image

The system launches a deep learning job in one or more Docker containers. A Docker images is required in advance. 
The system provides a base Docker images with HDFS, CUDA and cuDNN support, based on which users can build their own custom Docker images.

To build a base Docker image, for example [Dockerfile.build.base](Dockerfiles/Dockerfile.build.base), run:
```sh
docker build -f Dockerfiles/Dockerfile.build.base -t aii.build.base:hadoop2.7.2-cuda8.0-cudnn6-devel-ubuntu16.04 Dockerfiles/
```

Then a custom docker image can be built based on it by adding `FROM aii.build.base:hadoop2.7.2-cuda8.0-cudnn6-devel-ubuntu16.04` in the Dockerfile.

As an example, we customize a TensorFlow Docker image using [Dockerfile.run.tensorflow](Dockerfiles/Dockerfile.run.tensorflow):
```sh
docker build -f Dockerfiles/Dockerfile.run.tensorflow -t aii.run.tensorflow Dockerfiles/
```

Next, the built image is pushed to a docker registry for every node in the system to access that image:
```sh
docker tag aii.run.tensorflow localhost:5000/aii.run.tensorflow
docker push localhost:5000/aii.run.tensorflow
```

And the image is ready to serve. Note that above script assume the docker registry is deployed locally. 
Actual script can vary depending on the configuration of Docker registry. 


## Json config file for job submission

A json file describe detailed configuration required for a job submission. The detailed format is shown as below:

```
{
  "jobName":   String,
  "image":     String,
  "dataDir":   String,
  "outputDir": String,
  "codeDir":   String,
  "taskRoles": [
    {
      "name":       String,
      "taskNumber": Integer,
      "cpuNumber":  Integer,
      "memoryMB":   Integer,
      "gpuNumber":  Integer,
      "command":    String
    }
  ],
  "killAllOnCompletedTaskNumber": Integer,
  "retryCount": Integer
}
```

Below please find the detailed explanation for each of the parameters in the config file:

| Field Name                     | Schema                     | Description                              |
| :----------------------------- | :------------------------- | :--------------------------------------- |
| `jobName`                      | String, required           | Name for the job, need to be unique      |
| `image`                        | String, required           | URL pointing to the Docker image for all tasks in the job |
| `dataDir`                      | String, optional, HDFS URI | Data directory existing on HDFS          |
| `outputDir`                    | String, optional, HDFS URI | Output directory existing on HDFS        |
| `codeDir`                      | String, required, HDFS URI | Code directory existing on HDFS          |
| `taskRoles`                    | List, required             | List of `taskRole`, one task role at least |
| `taskRole.name`                | String, required           | Name for the task role, need to be unique with other roles |
| `taskRole.taskNumber`          | Integer, required          | Number for the task role, no less than 1 |
| `taskRole.cpuNumber`           | Integer, required          | CPU number for one task in the task role, no less than 1 |
| `taskRole.memoryMB`            | Integer, required          | Memory for one task in the task role, no less than 100 |
| `taskRole.gpuNumber`           | Integer, required          | GPU number for one task in the task role, no less than 0 |
| `taskRole.command`             | String, required           | Executable command for tasks in the task role, can not be empty |
| `killAllOnCompletedTaskNumber` | Integer, optional          | Number of completed tasks to kill the entire job, no less than 0 |
| `retryCount`                   | Integer, optional          | Job retry count, no less than 0          |


## Runtime environment

Each task in a job runs in one Docker container. 
For a multi-task job, one task might communicate with others.
So a task need to be aware of other tasks' runtime information such as IP, port, etc. 
The system exposes such runtime information as environment variables to each task's Docker container. 
For mutual communication, user can write code in the container to access those runtime environment variables.

Below we show a complete list of environment variables accessible in a Docker container:

| Environment Variable Name          | Description                              |
| :--------------------------------- | :--------------------------------------- |
| AII_JOB_NAME                       | `jobName` in config file                 |
| AII_DATA_DIR                       | `dataDir` in config file                 |
| AII_OUTPUT_DIR                     | `outputDir`in config file                |
| AII_CODE_DIR                       | `codeDir` in config file                 |
| AII_TASK_ROLE_NAME                 | `taskRole.name` of current task role     |
| AII_TASK_ROLE_NUM                  | `taskRole.number` of current task role   |
| AII_TASK_CPU_NUM                   | `taskRole.cpuNumber` of current task     |
| AII_TASK_MEM_MB                    | `taskRole.memoryMB` of current task      |
| AII_TASK_GPU_NUM                   | `taskRole.gpuNumber` of current task     |
| AII_TASK_ROLE_INDEX                | Index of current task in the task role, starting from 0 |
| AII_TASK_ROLE_NO                   | Index of current task role in config file, starting from 0 |
| AII_TASKS_NUM                      | Total tasks' number in config file       |
| AII_TASK_ROLES_NUM                 | Total task roles' number in config file  |
| AII_KILL_ALL_ON_COMPLETED_TASK_NUM | `killAllOnCompletedTaskNumber` in config file |
| AII_CURRENT_CONTAINER_IP           | Allocated ip for current docker container |
| AII_CURRENT_CONTAINER_PORT         | Allocated port for current docker container |
| AII_TASK_ROLE\_`$i`\_HOST_LIST     | Host list for `AII_TASK_ROLE_NO == $i`, comma separated `ip:port` string |


## An example deep learning job

A distributed TensorFlow job is listed below as an example:

```
{
  "jobName": "tensorflow-distributed-jobguid",
  // customized tensorflow docker image with hdfs, cuda and cudnn support
  "image": "localhost:5000/aii.run.tensorflow",
  // this example uses cifar10 dataset, which is available from
  // http://www.cs.toronto.edu/~kriz/cifar.html
  "dataDir": "hdfs://path/tensorflow-distributed-jobguid/data",
  "outputDir": "hdfs://path/tensorflow-distributed-jobguid/output",
  // this example uses code from tensorflow benchmark https://git.io/vF4wT
  "codeDir": "hdfs://path/tensorflow-distributed-jobguid/code",
  "taskRoles": [
    {
      "name": "ps_server",
      // use 2 ps servers in this job
      "taskNumber": 2,
      "cpuNumber": 2,
      "memoryMB": 8192,
      "gpuNumber": 0,
      // run tf_cnn_benchmarks.py in code directory
      // please refer to https://www.tensorflow.org/performance/performance_models#executing_the_script for arguments' detail
      // if there's no `scipy` in the docker image, need to install it first
      "command": "pip install scipy && python tf_cnn_benchmarks.py --local_parameter_device=cpu --num_gpus=4 --batch_size=32 --model=resnet20 --variable_update=parameter_server --data_dir=$AII_DATA_DIR --data_name=cifar10 --train_dir=$AII_OUTPUT_DIR --ps_hosts=$AII_TASK_ROLE_0_HOST_LIST --worker_hosts=$AII_TASK_ROLE_1_HOST_LIST --job_name=ps --task_index=$AII_TASK_ROLE_INDEX"
    },
    {
      "name": "worker",
      // use 2 workers in this job
      "taskNumber": 2,
      "cpuNumber": 2,
      "memoryMB": 16384,
      "gpuNumber": 4,
      "command": "pip install scipy && python tf_cnn_benchmarks.py --local_parameter_device=cpu --num_gpus=4 --batch_size=32 --model=resnet20 --variable_update=parameter_server --data_dir=$AII_DATA_DIR --data_name=cifar10 --train_dir=$AII_OUTPUT_DIR --ps_hosts=$AII_TASK_ROLE_0_HOST_LIST --worker_hosts=$AII_TASK_ROLE_1_HOST_LIST --job_name=worker --task_index=$AII_TASK_ROLE_INDEX"
    }
  ],
  // kill all 4 tasks when 2 worker tasks completed
  "killAllOnCompletedTaskNumber": 2,
  "retryCount": 0
}
```


## Job submission

1. Put the code and data on HDFS

    Use `aii-fs` to upload your code and data to HDFS on the system, for example
    ```sh
    aii-fs -cp -r /local/data/dir hdfs://path/tensorflow-distributed-jobguid/data
    ```
    please refer to [aii-fs/README.md](../aii-fs/README.md#usage) for more details.

2. Prepare a job config file

    Prepare the [config file](#json-config-file-for-job-submission) for your job.

3. Submit the job through web portal

    Open web portal in a browser, click "Submit Job" and upload your config file.