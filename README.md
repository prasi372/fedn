![alt text](https://thumb.tildacdn.com/tild6637-3937-4565-b861-386330386132/-/resize/560x/-/format/webp/FEDn_logo.png)
## What is FEDn?
FEDn is an open-source, modular framework for Federated Machine Learning (FedML), developed and maintained by Scaleout Systems. It enables developers to configure and deploy *FEDn networks* to support different federated scenarios, ranging from cross-silo to cross-device use-cases.   
  
## Core Features

FEDn is developed ground up for full-scale deployments in distributed, heterogeneous environments. A key feature is the ability to seamlessly go from local development and testing to live production deployments. Three key design objectives are guiding development in the project: 

### A ML-framework agnostic, black-box design
The framework treats client model updates and model validations as black-box computations. A developer can follow a structured design pattern to implement clients, and to extend the framework with support for virtually any ML model type or framework. Support for Keras and PyTorch artificial neural network models are available out-of-the-box, and support for others, including select models from SKLearn, are in active development.  

### Horizontally scalable through a tiered aggregation scheme 
FEDn is designed to allow for flexible and easy scaling to meet both the demands from a growing number of clients and latency and throughput requirements spanning cross-silo and cross-device cases. This is achieved by a tiered architecture for model updates and model aggregation where multiple combiners divide up the work. Thus, the computing model in FEDn draws parallels to the MapReduce programming model, assuring good horizontal scalability. Recent benchmarks show high performance both for thousands of clients in a cross-device setting and for 40 clients with large model updates (1GB) in a cross-silo setting, see https://arxiv.org/abs/2103.00148.   

### Built for real-world distributed computing scenarios 
FEDn is built to support real-world, production deployments. FEDn relies on proven best-practices in distributed computing, uses battle-hardened components, and incorporates enterprise security features. There is no "simulated mode", only distributed mode. However, it is of course possible to run a local sandbox system in a pseudo-distributed mode for convenient testing and development.  

## Architecture

Constructing a federated model with FEDn amounts to a) specifying the details of the client-side training code and data integrations, and b) deploying the reducer-combiner network. A FEDn network, as illustrated in the picture below, is made up of three main components: the *Reducer*, one or more *Combiners*, and a number of *Clients*. The combiner network forms the backbone of the FedML orchestration mechanism, while the Reducer provides discovery services and provides controls to coordinate training over the combiner network. By horizontally scaling the combiner network, one can meet the needs of a growing number of clients.  
 
![alt-text](docs/img/overview.png?raw=true "FEDn network")

### Main components

#### Client
A Client is a data node, holding private data and connecting to a Combiner to receive model update requests and model validation requests during training rounds. Importantly, clients do not require any open ingress ports. A client receives the code to be executed from the Reducer upon connecting to the network, and thus they only need to be configured prior to connection to read the local datasets during training and validation. Python3 client implementation is provided out of the box, and it is possible to write clients in a variety of languages to target different software and hardware requirements.  

#### Combiner
A combiner is an actor whose main role is to orchestrate and aggregate model updates from a number of clients during a training round. When and how to trigger such orchestration rounds are specified in the overall *compute plan* laid out by the Reducer. Each combiner in the network runs an independent gRPC server, providing RPCs for interacting with the alliance subsystem it controls. Hence, the total number of clients that can be accommodated in a FEDn network is proportional to the number of active combiners in the FEDn network. Combiners can be deployed anywhere, e.g. in a cloud or on a fog node to provide aggregation services near the cloud edge. 

#### Reducer
The reducer fills three main roles in the FEDn network: 1.) it lays out the overall, global training strategy and communicates that to the combiner network. It also dictates the strategy to aggregate model updates from individual combiners into a single global model, 2.) it handles global state and maintains the *model trail* - an immutable trail of global model updates uniquely defining the FedML training timeline, and  3.) it provides discovery services, mediating connections between clients and combiners. For this purpose, the Reducer exposes a standard REST API. 

### Services and communication
The figure below provides a logical architecture view of the services provided by each agent and how they interact. 

![Alt text](docs/img/FEDn-arch-overview.png?raw=true "FEDn architecture overview")

### Control flows and algorithms
FEDn is designed to allow customization of the FedML algorithm, following a specified pattern, or programming model. Model aggregation happens on two levels in the system. First, each Combiner can be configured with a custom orchestration and aggregation implementation, that reduces model updates from Clients into a single, *combiner level* model. Then, a configurable aggregation protocol on the Reducer level is responsible for combining the combiner-level models into a global model. By varying the aggregation schemes on the two levels in the system, many different possible outcomes can be achieved. Good staring configurations are provided out-of-the-box to help the user get started. 

#### Hierarchical Federated Averaging
The currently implemented default scheme uses a local SGD strategy on the Combiner level aggregation and a simple average of models on the reducer level. This results in a highly horizontally scalable FedAvg scheme. The strategy works well with most artificial neural network (ANNs) models, and can in general be applied to models where it is possible and makes sense to form mean values of model parameters (for example SVMs). Additional FedML training protocols, including support for various types of federated ensemble models, are in active development.  
![Alt text](docs/img/HFedAvg.png?raw=true "FEDn architecture overview")


## Getting started 

The easiest way to start with FEDn is to use the provided docker-compose templates to launch a local sandbox / simulated environment consisting of one Reducer, two Combiners, and five Clients. Together with the supporting storage and database services (currently Minio and MongoDB), this constitutes a minimal system for training a federated model and learning the FEDn architecture. FEDn projects are templated projects that contain the user-provided model application components needed for federated training. This repository bundles a number of such test projects in the 'test' folder. These projects can be used as templates for creating your own custom federated model. 

Clone the repository (make sure to use git-lfs!) and follow these steps:

### Pseudo-distributed deployment
We provide docker-compose templates for a minimal standalone, pseudo-distributed Docker deployment, useful for local testing and development on a single host machine. 

1. Create a default docker network 
We need to make sure that all services deployed on our single host can communicate on the same docker network. Therefore, our provided docker-compose templates use a default external network 'fedn_default'. First, create this network: 

````bash 
$ docker network create fedn_default
````

2. Deploy the base services (Minio and MongoDB):

````bash 
$ docker-compose -f config/base-services.yaml up 
````

Make sure you can access the following services before proceeding to the next steps: 
 - Minio: localhost:9000
 - Mongo Express: localhost:8081
 
2. Start the Reducer

Copy the settings config file for the reducer, 'config/settings-reducer.yaml.template' to 'config/settings-reducer.yaml'. You do not need to make any changes to this file to run the sandbox. To start the reducer service:

````bash 
$ docker-compose -f config/reducer.yaml up 
````

3. Start a combiner:
Copy the settings config file for the reducer, 'config/settings-combiner.yaml.template' to 'config/settings-combiner.yaml'. You do not need to make any changes to this file to run the sandbox. To start the combiner service and attach it to the reducer:

````bash 
$ docker-compose -f config/combiner.yaml up 
````

Make sure that you can access the Reducer UI at https://localhost:8090 and that the combiner is up and running before proceeding to the next step.

### Train a federated model
Training a federated model on the FEDn network involves uploading a compute package, seeding the model, and attaching clients to the network. 

#### Upload a compute package

Navigate to https://localhost:8090/context and upload the compute package in 'test/mnist/package/mnist.tar.gz'. 

#### Seed the system with a base model

To prepare FEDn to run training, we need to upload a seed model via this endpoint (https://localhost:8090/history). Creating and staging the seed model is typically done by the founding members of the ML alliance. For testing purposes, you find a pre-generated seed model in "test/mnist/seed" (and correspondingly for the other examples).

#### Attach two Clients to the FEDn network:
Go to the directory "test/mnist-keras". This will build the needed environment and start two clients and attach them to the combiner:

````bash 
docker-compose -f docker-compose.yaml up --scale client=2
````

#### Start training the model
To start training the model, navigate to the Reducer REST API endpoint: localhost:8090/start.  You can follow the progress of training visually at https://localhost:8090/dashboard.

#### Developing on FEDn 
For development, it is convenient to use the docker-compose templates config/reducer-dev.yaml and config/combiner-dev.yaml. These files will let you conveniently rebuild the reducer and combiner images with the current local version of the fedn source tree instead of the latest stable release. 
 
## Distributed deployment
The actual deployment, sizing of nodes, and tuning of a FEDn network in production depends heavily on the use case (cross-silo, cross-device, etc), the size of model updates, on the available infrastructure, and on the strategy to provide end-to-end security. You can easily use the provided docker-compose templates to deploy FEDn network across different hosts in a live environment, but note that it might be necessary to modify them slightly depending on your target environment and host configurations.   

> Warning, there are additional security considerations when deploying a live FEDn network, outside of core FEDn functionality. Make sure to include these aspects in your deployment plans.

This example serves as reference deployment for setting up a fully distributed FEDn network consisting of one host serving the supporting services (Minio, MongoDB), one host serving the reducer, one host running two combiners, and one host running a variable number of clients. 

### Prerequisite for the reference deployment

#### Hosts
This example assumes root access to 4 Ubuntu 20.04 Server hosts for the FEDn network. We recommend at least 4 CPU, 8GB RAM flavors for the base services and the reducer, and 4 CPU, 16BG RAM for the combiner host. Client host sizing depends on the number of clients you plan to run. You need to be able to configure security groups/ingress settings for the service node, combiner, and reducer host.

#### Certificates
Certificates are needed for the reducer and combiner services. By default, FEDn will generate unsigned certificates for the reducer and combiner nodes using OpenSSL. 

> Certificates based on IP addresses are not supported due to incompatibilities with gRPC. 

### 1. Deploy supporting services  
First, deploy Minio and Mongo services as in the above example (make sure to change the default passwords). Confirm that you can access MongoDB via the MongoExpress dashboard before proceeding with the reducer.  

> Skip this step if you already have API access to Minio and MongoDB services. 

### 2. Deploy the reducer
Follow the steps for pseudo-distributed deployment, but now edit the settings-reducer.yaml file to provide the appropriate connection settings for MongoDB and Minio. Also, copy 'config/extra-hosts-reducer.yaml.template' to 'config/extra-hosts-reducer.yaml' and edit it, adding a host:IP mapping for each combiner you plan to deploy. Then you can start the reducer: 

```bash
sudo docker-compose -f config/reducer.yaml -f config/extra-hosts-reducer.yaml up 
```

### 3. Deploy combiners
Edit 'config/settings-combiner.yaml' to provide a name for the combiner (used as a unique identifier for the combiner in the network), a hostname (which is used by reducer and clients to connect to combiner RPC), and the port. Also, provide the IP and port to the reducer under the 'controller' tag. Then deploy the combiner: 

```bash
sudo docker-compose -f config/combiner.yaml up 
```

Repeat the same step for the second combiner node. Make sure to provide unique names for the two combiners. 

> Note that it is not currently possible to use the host's IP address as 'host'. This is due to gRPC not being able to handle certificates based on IP. 

### 4. Attach clients to the FEDn network
Once the FEDn network is deployed, you can attach clients to it in the same way as for the pseudo-distributed deployment. You need to provide clients with DNS information for all combiner nodes in the network. For example, to start 5 unique MNIST clients on a single host: 

Copy  'config/extra-hosts-clients.template.yaml' to 'test/mnist/extra-hosts.yaml' and edit it to provide host:IP mappings for the combiners in the network. Then, from 'test/mnist':

```bash
sudo docker-compose -f docker-compose.yaml -f config/extra-hosts-client.yaml up --scale client=5 
```

## Deploying to Kubernetes
Helm charts for production deployment to Kubernetes are developed and maintained by Scaleout as part of the [STACKn cloud-native MLOps platform](https://github.com/scaleoutsystems/stackn). 

## Where to go from here
Additional example projects/clients:

- Sentiment analyis with a Keras CNN-lstm trained on the IMDB dataset (cross-silo): https://github.com/scaleoutsystems/FEDn-client-imdb-keras 
- Sentiment analyis with a PyTorch CNN trained on the IMDB dataset (cross-silo): https://github.com/scaleoutsystems/FEDn-client-imdb-pytorch.git 
- VGG16 trained on cifar-10 with a PyTorch client (cross-silo): https://github.com/scaleoutsystems/FEDn-client-cifar10-pytorch 
- Human activity recognition with a Keras CNN based on the casa dataset (cross-device): https://github.com/scaleoutsystems/FEDn-client-casa-keras 
 
## Support
Reach out to Scaleout (https://scaleoutsystems.com) to learn how to configure and deploy zero-trust FEDn networks in production based on FEDn, and how to adapt FEDn to support a range of use-case scenarios.

## Contributions
All pull requests will be considered. We are currently managing issues in an external tracker (Jira). Reach out to one of the maintainers if you are interested in making contributions, and we will help you find a good first issue to get started. 

## License
FEDn is licensed under Apache-2.0 (see LICENSE file for full information).
