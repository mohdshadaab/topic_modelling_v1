## Table of Content  <!-- omit in toc -->

 - [Installation and Setup](#Installation-and-Setup)
 - [Understanding the Architecture](#Understanding-the-Architecture)
 - [Running the project](#Running-the-project)

## Installation and Setup
### Installation
 *  [Docker](https://docs.docker.com/engine/install/ubuntu/#installation-methods)
 *  [MiniKube](https://v1-18.docs.kubernetes.io/docs/tasks/tools/install-minikube/#installing-minikube)
 
 Set Docker as default driver
 ```BASH
 minikube config set driver docker
 ```
 
 Start the cluster
 ```BASH
 minikube start
 ```

## Understanding the Architecture

### 1. Model
Used the provided mnist_cnn.pt model, but it's accuracy is quite low. A [better model](https://paperswithcode.com/sota/image-classification-on-mnist?metric=Accuracy) could have been used.

### 2. Backend Server

* Used FASTApi and Uvicorn.
* Could have used [Falcon](https://falconframework.org/) for even better performance.
* Defined a function ```stringToImage(base64: str)```, for decoding base64 image.
* FASTApi server is auto-tuned for your current server (and number of CPU cores).
* Security: we will have to define a security scheme using OAuth2 with password(and hashing), bearer with JWT tokens for authentication.
* There can be performance bottleneck when integrating with a database, so we have to be very careful about the queries.
* Test for network optimization after exposing external endpoints.

### 3. Container

* Used docker to containerise backend server.
* Multi-stage build using buildkit.
* 
