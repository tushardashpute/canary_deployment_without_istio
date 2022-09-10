# Canary Deployments with NGINX Ingress Controller
## Overview
This repository contains all resources that are required to test the canary feature of NGINX Ingress Controller. 

## Requirements
* Kubernetes cluster 

```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install my-ingress-nginx ingress-nginx/ingress-nginx --version 4.2.5
```

## Getting Started

### Canary Test Scenario
##### Prepare Manifests  
First of all, change the host definition in the ingress manifests ***deploy/prod-ingress.yaml*** and ***deploy/canary-ingress.yaml*** from canary-demo.example.com to your URL
  
##### Deploy production release  
Roll-out the stable version 1.0.0 to the cluster
```bash
  kubectl apply -f ./deploy/prod-namespace.yaml
  kubectl apply -f ./deploy/prod-deployment.yaml,./deploy/prod-service.yaml,./deploy/prod-ingress.yaml  
  sleep 2
  kubectl get deploy,svc,ing -n demo-prod
  
```
  
##### Run tests  
Execute the following commands to send n=1000 requests to the endpoint
```bash
#!/bin/bash
for i in {1..1000}
do
   curl http://<your_url>/version
done

$ curl -s "http://<your_URL>/metrics" | jq '.calls'
```
If everything is working as expected, the curl command should return "1000".
  
##### Reset request counter  
Send GET requests to /reset endpoint to set the request counter to zero
```bash
$ curl "http://<your_URL>/reset"
```
  
##### Canary deployment  
Push the new software version 1.0.1 as a canary deployment to the cluster
```bash
  kubectl apply -f ./deploy/canary-namespace.yaml
  kubectl apply -f ./deploy/canary-deployment.yaml,./deploy/canary-service.yaml,./deploy/canary-ingress.yaml
  sleep 2
  kubectl get deploy,svc,ing -n demo-canary
```
  
##### Perform tests  
Again, start sending traffic to the endpoint
```bash
#!/bin/bash
for i in {1..1000}
do
   curl http://<your_url>/version
done
```
  
##### Verify the weight split  
Do a port forward to each of the pods to check the request count
```bash
$ kubectl -n demo-prod port-forward <pod-name> 8080:8080
$ curl -s http://localhost:8080/metrics | jq '.calls'
$ kubectl -n demo-prod port-forward <pod-name> 8081:8080
$ curl -s http://localhost:8081/metrics | jq '.calls'
```
Unless the weight has been changed to a different value, you should see approximately 800 requests being served by the production deployment and the remainig 200 by the canary. 

### Delete
Remove all resource from the cluster 
```bash
$ make clean-up
```
