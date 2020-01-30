# Kubernetes deployment strategies

This is not an officially supported Google product.

Copyright 2019 Google LLC

A detailed overview of the deployment patterns is presented
[here](https://cloud.google.com/solutions/implementing-deployment-and-testing-strategies-on-gke).

## Getting started
Export environment variables:
```
export PROJECT=$(gcloud config get-value project)
```
Create a GKE cluster with Istio enabled:
```
gcloud beta container clusters create "test-cluster" \
   --zone "us-central1-a" \
   --scopes "https://www.googleapis.com/auth/cloud-platform" \
   --addons HorizontalPodAutoscaling,HttpLoadBalancing,Istio \
   --istio-config auth=MTLS_PERMISSIVE
```
Get running cluster credentials:
```
gcloud container clusters get-credentials "test-cluster"  \
   --zone "us-central1-a" \
   --project $PROJECT
```
Clone repository:
```
git clone https://github.com/GoogleCloudPlatform/gke-deployment-testing-strategies
cd kubernetes-deployment-patterns
```

## Recreate
Create deployment with the current version of the application:
```
kubectl apply -f recreate/deployment-old.yaml
```
Expose the deployment using Kubernetes service:
```
kubectl apply -f recreate/service.yaml
```
Check if deployment and service created successfully:
```
kubectl rollout status deploy app -w
kubectl get svc/app -w
```
Wait for external IP to be allocated before proceeding. Press CTRL-C to end the watch loop.

On a new terminal, get the service IP and send requests to the current deployment:
```
while(true); \
do \
curl "http://$(kubectl get svc app -o jsonpath="{.status.loadBalancer.ingress[0].ip}"):8080/version"; echo; \
done
```
Create deployment with the new version of the application:
```
kubectl apply -f recreate/deployment-new.yaml
```
Monitor the response changing on the terminal where curl command was executed.

Cleanup:
```
kubectl delete -f recreate/ --ignore-not-found
```

## Rolling Update
Create deployment with the current version of the application:
```
kubectl apply -f rollingupdate/deployment-old.yaml
```
Expose the deployment using Kubernetes service:
```
kubectl apply -f rollingupdate/service.yaml
```
Check if deployment and service created successfully: 
```
kubectl rollout status deploy app -w
kubectl get svc/app -w
```
Wait for external IP to be allocated before proceeding. Press CTRL-C to end the watch loop.

On a new terminal, get the service IP and send requests to the current deployment:
```
while(true); \
do \
curl "http://$(kubectl get svc app -o jsonpath="{.status.loadBalancer.ingress[0].ip}"):8080/version"; echo; \
done
```
Create deployment with the new version of the application:
```
kubectl apply -f rollingupdate/deployment-new.yaml
```
Monitor the response changing on the terminal where curl command was executed.

Cleanup:
```
kubectl delete -f rollingupdate/ --ignore-not-found
```

## Blue/Green
Create deployment with the current version of the application:
```
kubectl apply -f bluegreen/deployment-old.yaml
```
Expose the deployment using Kubernetes service:
```
kubectl apply -f bluegreen/service-old.yaml
```
Check if deployment and service created successfully:
```
kubectl rollout status deploy app-01 -w
kubectl get svc/app -w
```
Wait for external IP to be allocated before proceeding. Press CTRL-C to end the watch loop.

On a new terminal, get the service IP and send requests to the current deployment:
```
while(true); \
do \
curl "http://$(kubectl get svc app -o jsonpath="{.status.loadBalancer.ingress[0].ip}"):8080/version"; echo; \
done
```
Create deployment with the new version of the application:
```
kubectl apply -f bluegreen/deployment-old.yaml
```
Check if the new deployment created successfully:
```
kubectl rollout status deploy app-02 -w
```
Update the service to point to the new version:
```
kubectl apply -f bluegreen/service-new.yaml
```
Monitor the response changing on the terminal where curl command was executed.

Cleanup:
```
kubectl delete -f bluegreen/ --ignore-not-found
```

## Canary
Create deployment with the current version of the application:
```
kubectl apply -f canary/deployment-old.yaml
```
Expose the deployment using Istio ingress gateway:
```
kubectl apply -f canary/gateway.yaml -f canary/virtualservice.yaml
```
Check if deployment created:
```
kubectl rollout status deploy app-01 -w
```
On a new terminal, get Istio ingress gateway IP and send requests:
```
while(true); \
do \
curl "http://$(kubectl get service istio-ingressgateway -n istio-system -o jsonpath="{.status.loadBalancer.ingress[0].ip}")/version"; echo; \
done
```
Create deployment with the new version of the application:
```
kubectl apply -f canary/deployment-new.yaml
```
Check if the new deployment created successfully:
```
kubectl rollout status deploy app-02 -w
```
Enforce traffic split rules (80-20) between the two versions:
```
kubectl apply -f canary/destinationrule.yaml -f canary/virtualservice-split.yaml
```
Monitor the response changing on the terminal where curl command was executed.

Cleanup:
```
kubectl delete -f canary/ --ignore-not-found
```

## Shadow
Create deployment with the current version of the application:
```
kubectl apply -f shadow/deployment-old.yaml
```
Expose the deployment using Istio ingress gateway:
```
kubectl apply -f shadow/gateway.yaml -f shadow/virtualservice.yaml
```
Check if deployment created successfully:
```
kubectl rollout status deploy app-01 -w
```
On a new terminal, get Istio ingress gateway IP and send requests:
```
while(true); \
do \
curl "http://$(kubectl get service istio-ingressgateway -n istio-system -o jsonpath="{.status.loadBalancer.ingress[0].ip}")/version"; echo; \
done
```
Create deployment with the new version of the application:
```
kubectl apply -f shadow/deployment-new.yaml
```
Set up mirroring:
```
kubectl apply -f shadow/virtualservice-mirror.yaml
```
Check if the new deployment created successfully:
```
kubectl rollout status deploy app-02 -w
```
Check pods logs:
```
kubectl logs -f --tail=3 deployment/app-01
kubectl logs -f --tail=3 deployment/app-02
```
Cleanup:
```
kubectl delete -f shadow/ --ignore-not-found
```

## A/B Testing
Create deployment with the current version of the application:
```
kubectl apply -f ab/deployment-old.yaml
```
Expose the deployment using Istio ingress gateway:
```
kubectl apply -f ab/gateway.yaml -f ab/virtualservice.yaml
```
Check if deployment created successfully:
```
kubectl rollout status deploy app-01 -w
```
Get Istio ingress gateway IP and send request:
```
curl "http://$(kubectl get service istio-ingressgateway -n istio-system -o jsonpath="{.status.loadBalancer.ingress[0].ip}")/version"
```
Create deployment with the new version of the application:
```
kubectl apply -f ab/deployment-new.yaml
```
Check if the new deployment created successfully:
```
kubectl rollout status deploy app-02 -w
```
Split traffic based on "end-user" header:
```
kubectl apply -f ab/destinationrule.yaml -f ab/virtualservice-split.yaml
```
Send request with end-user as "dummyUser":
```
curl -H "end-user:dummyUser" "http://$(kubectl get service istio-ingressgateway -n istio-system -o jsonpath="{.status.loadBalancer.ingress[0].ip}")/version"
```
Cleanup:
```
kubectl delete -f ab/ --ignore-not-found
```
