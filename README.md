# DevOps Assignment

This repository contains the steps and resources to set up a local Kubernetes cluster using **Minikube** on an **Ubuntu** system. 
---

## Setup 
- Operating System: AWS EC2 instance - **Ubuntu** (x86-64)
- A user account with **sudo** privileges
- An active internet connection

---

## ⚙️ Task 1: Kubernetes Cluster Setup

This section outlines the process to install Docker and Minikube on your Ubuntu system to Create a local 3 node Kubernetes cluster.


## Installation Process

All commands should be run in the terminal.

```bash
# Update the system packages
sudo apt-get update

# Install required packages
sudo apt-get install -y apt-transport-https ca-certificates curl

# Install Docker
sudo apt-get install -y docker.io

```
## 🧰 Setup minikube

1. Download and install the latest Minikube release
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb
```
2. Start Minikube
```bash
minikube start --driver=docker --force -n 3
```
3. Check minikube status
```
minikube status
```
<img width="910" height="300" alt="image" src="https://github.com/user-attachments/assets/eda00d5b-f129-4a68-a514-61c7886da557" />

4. Create an alias for kubectl via Minikube
```
alias kubectl="minikube kubectl --"
```
5. Check node status
```bash
kubectl get nodes
```
<img width="1478" height="208" alt="image" src="https://github.com/user-attachments/assets/8af6923f-5e00-4b7f-883d-50cceded9cf6" />



---
## ⚙️ Task 2: Nginx Ingress Setup

Install an Nginx ingress controller using Helm to expedite the process and Configure a basic ingress for the application, routing traffic through a 
single path.

1. Install helm
```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```
2. Check installation
```bash
helm version
```
![Image](https://github.com/user-attachments/assets/25ca8812-5bc0-4df4-b28a-43845f51dceb)

3. To install an NGINX Ingress controller using Helm, add the nginx-stable repository to helm, then run helm repo update . After we have added the repository we can deploy using the chart nginx-stable/nginx-ingress.
```bash
helm repo add nginx-stable https://helm.nginx.com/stable
helm repo update
```
4. Install nginx ingress
```bash
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --values values.yaml
```
values.yaml file
```bash
controller:
  kind: DaemonSet
  hostPort:
    enabled: true
  ingressClass: nginx
```
5. Check NGINX Ingress controller is running.
```bash
kubectl get all -n ingress-nginx
```
<img width="1478" height="268" alt="image" src="https://github.com/user-attachments/assets/0c774f36-e1ce-4482-b4f1-2d71db45404b" />

---
## ⚙️ Task 3: Sample Application Deployment

Deploy a "Hello World" application using a pre-built Docker image. Create a Kubernetes manifest for deployment, ensuring the application is 
accessible via the ingress.

1. Create a yaml file to Deploy hello world application and create service 
```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
  namespace: hello-world
  labels:
    app: hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-world
        image: hashicorp/http-echo
        args:
          - "-text=Hello World Assignment"
        ports:
        - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: hello-world
  namespace: hello-world
spec:
  selector:
    app: hello-world
  ports:
  - protocol: TCP
    port: 80
    targetPort: 5678
```

2. Create a namespace and apply the yml file
```bash
kubectl create namespace hello-world
kubectl apply -f helloWorld.yaml
```

3. Create hello-world-ingress.yaml
```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-world-ingress
  namespace: hello-world
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: hello.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-world
            port:
              number: 80
```

4. Apply hello-world-ingress.yaml
```bash
kubectl apply -f hello-world-ingress.yaml
```
5. Check the Status 
```bash
kubectl get all -n hello-world
```
<img width="1185" height="317" alt="image" src="https://github.com/user-attachments/assets/1e846b88-5b64-4ae9-9631-81adbab16dc6" />

```bash
curl http://hello.local
```
![Image](https://github.com/user-attachments/assets/940510a5-dd85-41e9-9369-b30134c7bef9)

---
## ⚙️ Task 4: Security with TLS/SSL
Implement TLS termination at the Nginx ingress using a self-signed certificate. 

1. Generate Self-Signed TLS Certificate and Key with OpenSSL
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout hello-world.key \
  -out hello-world.crt \
  -subj "/CN=hello.local/O=hello.local"
```
This creates:
hello-world.crt → the certificate
hello-world.key → the private key

2. Create a Kubernetes TLS Secret
```bash

kubectl create secret tls hello-world-tls \
  --cert=hello-world.crt \
  --key=hello-world.key \
  -n hello-world
```
This creates a secret named hello-world-tls in the hello-world namespace.

3. Update the Ingress YAML for TLS
```bash
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-world-ingress
  namespace: hello-world
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - hello.local
      secretName: hello-world-tls
  rules:
    - host: hello.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: hello-world
                port:
                  number: 80
```

Test TLS
```bash
curl https://hello.local --insecure
```
<img width="819" height="67" alt="image" src="https://github.com/user-attachments/assets/27c729d8-58c0-4dd4-a36a-2d3fa14e6703" />

----

## ⚙️ Troubleshooting

IMPORTANT ***setup networking to allow inbound traffic

1. Encountering this error because Minikube does not support running the Docker driver as root
<img width="1302" height="199" alt="image" src="https://github.com/user-attachments/assets/cff27ee9-1d28-4f1d-8ba3-4bdb7443f6b4" />

```bash
use option "--driver=docker --force"
```

2. Minikube detected that kubectl wasn't available locally. It downloaded the matching kubectl binary to work with the current Kubernetes cluster version.
<img width="993" height="136" alt="image" src="https://github.com/user-attachments/assets/5ab60215-75b3-428a-98c6-ccc21d297e68" />

3. Helm repo could not be resolved 
<img width="756" height="55" alt="image" src="https://github.com/user-attachments/assets/fc7fac36-25c2-4883-83f9-dfb25c127c45" />

```bash
 helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```

4. error in resolving DNS
<img width="674" height="54" alt="image" src="https://github.com/user-attachments/assets/7b1fc8dd-6139-4d4e-a7af-31165b639f99" />

5. Challanges faced with YAML Syntax Issues,The IngressClass in the YAML does not match the controller and pods runtime issues debugged by checking logs and configuration.
