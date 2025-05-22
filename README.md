### EKS-Style Secure Deployment Using Minikube


* This project simulates a secure microservices deployment in a local Kubernetes cluster using Minikube, mimicking AWS EKS production patterns. The implementation focuses on infrastructure, security, observability, and incident response.


#### Architecture diagram

![Untitled Diagram drawio (1)](https://github.com/user-attachments/assets/7b573fe3-2ce5-4dd3-ac97-3e0295feb03d)

<img src="https://github.com/user-attachments/assets/7b573fe3-2ce5-4dd3-ac97-3e0295feb03d" width="700"> 



 #### Requirements:
* Install:
   1. Minikube
   2. Docker
   3. kubectl
   4. helm
   5. Kyverno



#### Start Minikube cluster:
```
minikube start --cpus=2 --memory=4096 --addons=ingress,metrics-server

```


### Part 1: Microservice Stack with Kustomize

 Microservice Stack with Kustomize

* This repository contains a minimal microservice architecture deployed with Kustomize.

## Services
- **Gateway**: `nginxdemos/hello` exposed via Ingress
- **Auth Service**: `kennethreitz/httpbin` (internal only)
- **Data Service**: `hashicorp/http-echo` (internal only)

## Features
- Namespaced deployments (`app`, `system`)
- Liveness and readiness probes
- Resource requests and limits
- Ingress access to gateway only

### Prerequisites
- Kubernetes cluster (e.g. Minikube)
- `kubectl`
- `kustomize`
- Ingress controller (e.g., NGINX)

## Deployment

```
kubectl apply -k overlays/dev/
```


## Local Ingress Access (Optional)
* Add the following to your `/etc/hosts`:
```
127.0.0.1 gateway.local
```

*  create  a ```minikube tunnel``` to access the Ingress 
Then access:
```
http://gateway.local
```
![MINGW64__c_Users_Rajib_EKS-Style-secure 17-05-2025 21_00_06](https://github.com/user-attachments/assets/710e1e84-f618-42b9-b69c-c732eb1553d5)



![Hello World - Google Chrome 17-05-2025 20_59_11](https://github.com/user-attachments/assets/f169643e-26d7-4939-82c1-582e55e3b72a)



-----------------------------------------
####  Part 2: IAM Simulation with MinIO

* Deployed MinIO as a mock S3 service.

####  The setup includes:

* MinIO deployment with credentials stored in a Kubernetes Secret

* ServiceAccount for the data-service with access to MinIO credentials

* Test pods to verify access control

#### Prerequisites
Kubernetes cluster(minikube)

Deployment Steps
1. Create the MinIO Secret
```
kubectl apply -f minio-secret.yaml
```
2. Deploy MinIO
```
kubectl apply -f minio-deployment.yaml
kubectl apply -f minio-service.yaml
```

3. Create ServiceAccount for data-service

```
kubectl apply -f data-sa.yaml
```
4. Update data-service Deployment
```
kubectl apply -f data-service-deployment.yaml
```

* Enforce Access Controls : enforced that only data-service can talk to MinIO using a NetworkPolicy, This allows only pods with label app: data-service to reach MinIO.



```
kubectl apply -f  allow-to-data-service.yaml
```

#### Test data-service access
1. Deploy the test pod:

```
kubectl apply -f test-data-pod.yaml
```
2. Access the pod:

```
kubectl exec -n app -it test-data -- bash
```

3. Inside the pod, create a bucket:

```
export AWS_ACCESS_KEY_ID=minioadmin
export AWS_SECRET_ACCESS_KEY=minioadmin123
aws --endpoint-url http://minio:9000 s3 mb s3://mybucket
```




4.Confirm the bucket was created
Still inside the pod, run:



```
aws --endpoint-url http://minio:9000 s3 ls
```

![MINGW64__c_Users_Rajib_EKS-Style-secure 17-05-2025 21_23_51](https://github.com/user-attachments/assets/ac20fc59-451f-4556-92d3-751bd1fab07a)

#### Verify access denial for unauthorized services
1. Deploy the test pod:

```
kubectl apply -f test-auth-pod.yaml
```
2. Attempt to access MinIO:

```
kubectl exec -n app -it test-auth -- bash
aws --endpoint-url http://minio:9000 s3 ls
```
![MINGW64__c_Users_Rajib_EKS-Style-secure 17-05-2025 21_28_08](https://github.com/user-attachments/assets/328ad9ea-6851-42b8-85bd-851cf01d00ac)



* This  fail as expected.
----------------------------
#### Part 3: Security Incident Simulation: Authorization Header Leak

* This repository documents a simulated security incident where the auth-service in a Kubernetes cluster was leaking Authorization headers to logs and communicating with unauthorized external endpoints. The solution involves:

* Identifying the logging leak

* Implementing network policies to restrict communication

* Setting up preventive controls using Kyverno  

Step 1: Simulate and Confirm the Logging Leak

   1. Port-forward the auth-service:

```
kubectl port-forward svc/auth-service 8080:80 -n app
```
  2. Send a test request with Authorization header:

```
curl -H "Authorization: Bearer secret-token" http://localhost:8080/headers
```
  3. Check for leaked headers in logs:

```
kubectl logs deploy/auth-service -n app
```

![MINGW64__c_Users_Rajib_EKS-Style-secure 17-05-2025 22_01_25](https://github.com/user-attachments/assets/eec971da-58c5-4f15-b8e2-bf248d7905c6)



### Block /headers via Ingress

#### using an NGINX ingress controller, so we’ll modify your Ingress resource YAML for gateway to block access to /headers (and optionally /get) using the nginx.ingress.kubernetes.io/server-snippet annotation.

Access Through Ingress in gateway-ingress.yaml

```
kubectl apply -f gateway-ingress.yaml
```

#### Add /etc/hosts entry (Windows)

Open Notepad as Admin

File > Open: C:\Windows\System32\drivers\etc\hosts

Add:
```
127.0.0.1 local.test
```

#### Test Ingress (not port-forward)
Now run:
```
curl http://local.test/headers
```

output: ```403 Forbidden```

![MINGW64__c_Users_Rajib_EKS-Style-secure 17-05-2025 22_41_05](https://github.com/user-attachments/assets/acca7e02-d8d7-45a1-b556-f799a92e19fd)



Step 2: Implement Network Controls
   1. Apply a deny-all-egress policy:

```
kubectl apply -f deny-egress.yaml
```
   2. Allow specific communication to data-service:

```
kubectl apply -f allow-to-data-service.yaml

```




#### Network Policy files:

* deny-egress.yaml

* allow-to-data-service.yaml

#### Allow auth-service to only communicate with data-service
* run kubectl curl test  
```
kubectl run curl-test -n app --image=alpine/curl --restart=Never -- sh -c "curl http://data-service.app.svc.cluster.local:5678
```

![MINGW64__c_Users_Rajib_EKS-Style-secure 17-05-2025 22_53_4222](https://github.com/user-attachments/assets/09f3693c-5149-440a-9758-9069d03d2b1c)


Step 3: Prevent Future Leaks with Kyverno
   1. Install Kyverno (if not present):

```
kubectl create -f https://github.com/kyverno/kyverno/releases/latest/download/install.yaml

```
  2. Apply the prevention policy:

```
kubectl apply -f block-authorization-logging.yaml
```


Policy file:

```block-authorization-logging.yaml```


![MINGW64__c_Users_Rajib_EKS-Style-secure 17-05-2025 23_50_33](https://github.com/user-attachments/assets/0fcd1505-bccf-4cb2-b196-3b1230d79893)



---------------------------------

Part 4: Observability

* This repository is  the setup of a comprehensive observability stack for Kubernetes using Prometheus, Grafana, and AlertManager. 

* Cluster-wide metrics collection

* Visualization dashboards

* Alerting for abnormal conditions

#### Installation
* Prerequisites
* Kubernetes cluster(minikube)

* Helm installed

* kubectl configured

1. Install Prometheus & Grafana Stack
```
# Add the Helm repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update


# Install the monitoring stack
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace

```
This installs:

*  Prometheus for metrics collection

* Grafana for visualization

* AlertManager for notifications

* Various exporters for Kubernetes metrics

#### Accessing the Dashboards
1. Grafana Access
Port-forward the Grafana service:

```
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80
```

![MINGW64__c_Users_Rajib_EKS-Style-secure 18-05-2025 09_37_04](https://github.com/user-attachments/assets/befb86c5-7b34-4184-962b-bf6497813c2d)


2.Open in browser: http://localhost:3000

3. Login credentials:

    1. Username: admin

     2.  Password: (retrieve with command below)

```
kubectl get secret monitoring-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode
```

### Prebuilt Dashboards
* Recommended dashboard IDs to import:

* Kubernetes Pod Monitoring: 6417

* Cluster Overview: 315

* HTTP metrics (if you expose them via Prometheus): Custom or use dashboards for NGINX/Ingress

  ![Kubernetes _ Compute Resources _ Pod - Dashboards - Grafana - Google Chrome 18-05-2025 09_39_42](https://github.com/user-attachments/assets/1c2e44c0-2e2e-4b45-9402-397683430e0a)


#### Set up an alert (in Prometheus or Grafana) for abnormal restarts or failed probes.

* Apply the alert rule:

```
kubectl apply -f restart-alert.yaml
```

* You’ll see it inside Prometheus Alerts UI (access via port-forward: svc/monitoring-kube-prometheus-stack-prometheus, port 9090)
  
![Prometheus Time Series Collection and Processing Server - Google Chrome 18-05-2025 07_38_28](https://github.com/user-attachments/assets/f3b74eee-48bf-43c3-8513-19a04ffcb725)



---------------------------------

 #### Part 5: Failure Simulation 
 
 #### Prerequisites
 * Kubernetes cluster with monitoring stack installed

* kubectl configured

* Grafana accessible (port-forwarded to localhost:3000)

  1. Simulate Pod Failure
```
# List running pods in your application namespace
kubectl get pods -n app
```
```
# Select and delete a pod to simulate failure
kubectl delete pod <auth-service-pod-name> -n app
```
2. Observe Automatic Recovery
```
# Watch the pod lifecycle events in real-time
kubectl get pods -n app -w

```
Expected recovery flow:
* Pod status changes to Terminating

* New pod gets scheduled (ContainerCreating)

* New pod reaches Running status

![MINGW64__c_Users_Rajib_EKS-Style-secure 18-05-2025 08_08_02](https://github.com/user-attachments/assets/abb79ec3-0cea-46ae-a1e6-53f9a25a0fdb)

  

#### Observe in Grafana
* Open Grafana:
*  Check for Metrics:
* Go to Dashboards → Browse → Kubernetes / Compute Resources / Pod (or any pod metrics dashboard).
* Pod CPU / Memory: Look for spikes/drops during deletion.

![Kubernetes _ Compute Resources _ Pod - Dashboards - Grafana - Google Chrome 19-05-2025 20_08_28](https://github.com/user-attachments/assets/8617be36-f76a-40e3-b394-289a651331f3)

  
