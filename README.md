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
Used the provided mnist_cnn.pt model, but it's accuracy is quite low. A [better model](https://paperswithcode.com/sota/image-classification-on-mnist?metric=Accuracy) can be used.

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
* Used multi-stage build and buildkit.
* Used HEALTHCHECK in Dockerfile, to check every 5 mins if the server works. HEALTHCHECK code can be improved.

### 4. Deployment

* Used minikube to deploy docker container on kubernetes.
* Assigned resources in the deployment yaml file.
* ```ReadinessProbe``` or ```LivenessProbe``` can be used to keep a check on deployment lifecycle and can also help ```Loadbalancer``` in redirecting traffic to a healthy pod.

## Running the project

For building the image, I have created a shell script.
```BASH
build_docker.sh
```
```BASH
#!/bin/bash

IMAGE_NAME=instill/mnist:1.1.0

#for using the minikube cluster's docker engine.
echo "Pointing your shell to minikube's docker-daemon...."
eval $(minikube docker-env)

echo "Building Docker Image...."
DOCKER_BUILDKIT=1 docker build -t `echo $IMAGE_NAME` . &
bp_id=$!

wait $bp_id

echo "All done!!"
```
Keep the image name same in ```deploy.yaml```.

The deployment config file, ```deploy.yaml```.
```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: instill
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      bb: web
  template:
    metadata:
      labels:
        bb: web
    spec:
      containers:
      - name: mnist
        image: instill/mnist:1.1.0
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
      

---
apiVersion: v1
kind: Service
metadata:
  name: entrypoint
  namespace: default
spec:
  type: NodePort
  selector:
    bb: web
  ports:
  - port: 8051
    targetPort: 8051
    nodePort: 30001
```
I have defined container config and service config in the same file. 
The service ```entrypoint``` is of type NodePort for exposing the port 30001 to external TCP traffic for accepting requests.

Now, deploying the container on kubernetes.
```BASH
deploy_conatiner.sh
```
```BASH
#!/bin/bash

#using code substitution for getting public ip of cluster
CLUSTER_IP=$(echo $(minikube ip))

#deploying the docker container
minikube kubectl -- apply -f deploy.yaml &
process_id=$!

wait $process_id

echo "Checking the status of pods in 10s...."
sleep 10


minikube kubectl -- get pods
sleep 10

#using code substitution for getting the NodePort which accepts external traffic
NODE_PORT=$(echo $(minikube kubectl -- describe service entrypoint | grep NodePort| grep -o "\S*[0-9]"))


echo "Should be serving at http://`echo $CLUSTER_IP`:`echo $NODE_PORT`/mnist/ in some time."
echo "To use swagger UI go to http://`echo $CLUSTER_IP`:`echo $NODE_PORT`/docs "

echo "Example : 
curl -X 'POST' 
  'http://192.168.49.2:30001/mnist/' 
  -H 'accept: application/json' 
  -H 'Content-Type: application/json' 
  -d '{
  "'"base64"'": "'"<string>"'"
}' "
```
I have also created a single shell script for building image and deploying conatiner.
```BASH
#!/bin/bash

echo "STAGE 1: Building Image"

./build_docker.sh &
process1_id=$!

wait $process1_id

echo "STAGE 2: Deploying container"

./deploy_container.sh &
process2_id=$!

wait $process2_id
```

Before testing the api, allow the nodeport to accept traffic through firewall.
```BASH
sudo ufw allow 30001
```
