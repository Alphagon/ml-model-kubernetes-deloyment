# Docker and Kubernetes for ML

## Overview
This documentation outlines the setup process to containerize a FastAPI application one built here and
deploy it on a Kubernetes cluster, making the service scalable and robust.

## Setup Process

### 1. Pushing  the image to Docker Hub
First create an account in docker hub. And install the docker client on your machine.
- Login to docker - `docker login`
- Tag the docker image which you want to push - `docker tag <image tag> <username>/<repository_name>:<tag>`
- Pushing the image to Docker hub - `docker push <username>/<repository_name>:<tag>`
- To remove the tagged image from local or cloud machine if not required - `docker rmi <username>/<repository_name>`


My image is published to following location in Dockerhub 
`alphagon/fastapi-sentiment:latest`

### 2. Creating service.yaml and configuration.yaml
Login to Google Cloud Services and open the cloud terminal 

Ensure you are using the correct GCP project and zone by running the following commands in the terminal:
- My project name is `theta-function-i8` - `gcloud config set project theta-function-i8`
- My Zone is `asia-south1-a` - `gcloud config set compute/zone asia-south1-a`

Create a Google Kubernetes cluster - `gcloud container clusters create fastapi-sentiment --zone asia-south1-a --num-nodes=3`

This command will take time to create the cluster. You can check the status of the cluster using the following command - 
`gcloud container clusters describe fastapi-sentiment --zone asia-south1-a`. 
Look for the status field in the output. It can show different states such as `PROVISIONING`, `RUNNING`, `REPAIRING`, or `DELETING`.
Wait till the status is converted from `PROVISIONING` to `RUNNING`.

Next get cluster credentials for kubectl to access - `gcloud container clusters get-credentials fastapi-sentiment --zone asia-south1-a`


Create yaml file for configutaion and services

```
cat <<EOF > configuration.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: fastapi-sentiment
  namespace: default
  labels:
    app: fastapi-sentiment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: fastapi-sentiment
  template:
    metadata:
      labels:
        app: fastapi-sentiment
    spec:
      containers:
      - name: fastapi-sentiment
        image: alphagon/fastapi-sentiment:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: fastapi-sentiment-hpa
  namespace: default
  labels:
    app: fastapi-sentiment
spec:
  scaleTargetRef:
    kind: Deployment
    name: fastapi-sentiment
    apiVersion: apps/v1
  minReplicas: 1
  maxReplicas: 5
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 80
EOF
```

```
cat <<EOF > service.yaml
apiVersion: v1
kind: Service
metadata:
  name: fastapi-sentiment-service
  namespace: default
  labels:
    app: fastapi-sentiment
  annotations:
    cloud.google.com/neg: '{"ingress":true}'
  finalizers:
  - service.kubernetes.io/load-balancer-cleanup
spec:
  type: LoadBalancer
  selector:
    app: fastapi-sentiment
  ports:
  - protocol: TCP
    port: 8000
    targetPort: 8000
    nodePort: 30428
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  allocateLoadBalancerNodePorts: true
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  sessionAffinity: None
EOF
```
### 3. Applying the configuraion and Service, and accessing the app from external IP
Apply configuration and service yaml files using kubectl
- `kubectl apply -f configuration.yaml`
- `kubectl apply -f service.yaml`

Verify deployment
- `kubectl get pods`
- `kubectl get svc`

Access the the app using the external IP generated from the above command 
`<external-IP>:8000/docks`

or using curl command

```
curl -X 'POST' \
  '<external-IP>:8000/predict' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -d '{
  "text": "The first episode is interesting and you get the feeling that this might be quite good! People get stuck in a village for mysterious reasons and get slaughtered by creepy monsters."
  }'
```

### 4. Deleting the cluster
Delete the cluster once it is not required - `gcloud container clusters delete fastapi-sentiment --zone asia-south1-a`

Once deleted check again if there are any cluster running - `gcloud container clusters list`