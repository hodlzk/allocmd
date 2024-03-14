# Deploy a Worker Node with allocmd

![Docker!](https://img.shields.io/badge/Docker-2CA5E0?style=for-the-badge&logo=docker&logoColor=white)
![Python!](https://img.shields.io/badge/Python-FFD43B?style=for-the-badge&logo=python&logoColor=blue)
![Apache License](https://img.shields.io/badge/Apache%20License-D22128?style=for-the-badge&logo=Apache&logoColor=white)

A worker node is a machine-intelligent application registered on the Allora chain that provides inference/prediction on a particular topic it's subscribed to and gets rewarded by the Allora chain validators based on the inference quality.

After providing more details about the structure of a worker node, this guide will walk through how to build a worker node and then deploy it to a remote environment. We do so using a tool created for the convenience of data scientists, `allocmd`. However, one can choose to build and deploy a worker node without this tool, which we discuss in the next section <a href="https://docs.allora.network/docs/build-a-worker-node-from-scratch" target="_blank">here</a>.

## Components of a Worker Node

A Worker Node in the Allora Protocol consists of the following components:

[block:image]
{
  "images": [
    {
      "image": [
        "https://files.readme.io/c081b88-model-diagram-final-v32x.png",
        null,
        ""
      ],
      "align": "center"
    }
  ]
}
[/block]


### 1. Allora Inference Base Image (AIB)

This Docker image serves as the base image for your Dockerfile and includes the fundamental node logic. It handles the combination of every part that makes the whole of the worker node and the accommodation for the additional functionality written in the node function.

### 2. Node Function

This component is the crux of your specific application, enhancing the `allora-inference-base` docker image. The node function stored on IPFS aims to handle all the custom functionalities needed to provide the inferences. While the AIB handles the communication between the worker node and the Allora chain, the node function provides the actual inference. The structure is such that the AIB downloads the function from IPFS given the attached CID and runs the function, which will then run your custom `main.py`. This is where the magic comes in: each worker will use a different, custom `main.py` to provide the inference requested. It is up to the worker to customize  `main.py`. A few examples: the process can be completely run inside that `main.py`, or it can hit a separate service (e.g. a separate backend with a REST API endpoint), it could query a particular database... the possibilities are endless. You can find a more comprehensive example <a href="https://docs.allora.network/docs/build-a-worker-node-from-scratch#2-node-function-python-script" target="_blank">here</a>.

### 3. Head Node

This independent node publishes the Allora chain request and inference topics to be subscribed to by the worker nodes, which provides inference data in response, thereby channeled to the chain. When a worker node is initialized, it starts with an environment variable called `BOOT_NODES`, which helps handle the connection and communications between worker nodes and the head node.

With these components, the worker and head node are registered on the Allora chain, which makes requests to the nodes. The inference request is made to the head from the chain, which in turn publishes the request to be subscribed to by the worker, which will call the functions that invoke your custom logic that handles the actual inference. The request-response is a bidirectional flow from the Allora chain to the head node to the worker node to the node function, and then to the model server.

## Building a Worker Node with the allocmd CLI

The `allocmd` is a CLI tool that handles worker nodes' seamless creation and deployment. With this tool, you do not need to write the worker node from scratch, the CLI tool will help you bootstrap all the needed components to get worker nodes working. All you have to do is to update the `config.yaml` file with your custom parameters, update the provided`main.py`to communicate with your inference server, run the deploy command, and your worker should be up and running.

To build a worker node with `allocmd`, you will need to follow the following steps:

### 1. Install `allocmd` CLI

You will begin with installing the tool on your machine. 

```shell
pip install allocmd
```

> you can run `allocmd --help` to get general help or `allocmd [command] --help` to get help relating to a particular command.

### 2. Initialize the worker for development

The next step is initializing the CLI to bootstrap all the needed components to get your worker running. The following command will handle the initialization process. It will create all the files in the appropriate directories and generate identities for your node to be used for local development.

```shell
allocmd init --name <preffered name> --topic <topic id> --env dev
```

Before running this command you will have to [pick the topic Id ](https://docs.allora.network/docs/existing-allora-appchain-topics)you wish to generate inference for after which you can run this command with the topic Id. The command will auto-create some files, the most important of which is the `dev-docker-compose.yaml`file which is an already complete docker-compose that you can run immediately to see your worker and head nodes running perfectly on your local machine. You can edit the files as you wish. for instance the `main.py` is meant for you to call your inference server, hence you will have to edit the sample code with actual URLs and logic as you prefer.

When you run the docker-compose (`docker-compose -f dev-docker-compose.yaml up --build`), maybe after you have written and tested your logic in `main.py`, you then should be seeing the logs from the nodes, and you should be able to make a request to your head node and see it get a response from the worker node. Note that in production, you won't be the one to make the inference request, as the Allora chain will do this at the cadence provided by the topic creator.

You can test your node by running the following curl command:

```
curl --location 'http://localhost:6000/api/v1/functions/execute' --header 'Accept: application/json, text/plain, */*' --header 'Content-Type: application/json;charset=UTF-8' --data '{
    "function_id": "bafybeigpiwl3o73zvvl6dxdqu7zqcub5mhg65jiky2xqb4rdhfmikswzqm",
    "method": "allora-inference-function.wasm",
    "parameters": null,
    "topic": "<TOPIC_ID>",
    "config": {
        "env_vars": [
            {                              
                "name": "BLS_REQUEST_PATH",
                "value": "/api"
            },
            {                              
                "name": "ALLORA_ARG_PARAMS",
                "value": "<argument>"
            }
        ],
        "number_of_nodes": -1,
        "timeout" : 2
    }
}' | jq
```

The `<TOPIC_ID>` needs to be [an existing topic on the chain](https://docs.allora.network/docs/existing-allora-appchain-topics). The `<argument>` is what the topic is expecting to receive to perform the inference (as an indication to test, you can use the `DefaultArg`  value from the topic on-chain, e.g. for ETH prediction topic, it should be `"ETH"`).

### 3. Initialize the worker for production

Your worker node is now ready to be deployed, the `main.py` has been modified, all env variables passed, and the worker node is running locally and you are now ready to deploy your worker to run in the production environment. The following command will handle the generation of the `prod-docker-compose.yaml` file which contains all the keys and parameters needed for your worker to function perfectly in production.

```shell
allocmd init --env prod
```

By running this command, `prod-docker-compose.yaml` will be generated with appropriate keys and parameters. You can now run the docker-compose file or deploy the whole codebase in your preferred cloud instance. At this stage, your worker should be responding to inference request from the Allora Chain.
