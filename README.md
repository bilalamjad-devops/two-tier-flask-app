# How to setup two-tier application deployment on kubernetes cluster
## First setup kubernetes kubeadm cluster
Use this repository to setup kubeadm https://github.com/LondheShubham153/kubestarter/blob/main/kubeadm_installation.md

## SetUp
- First clone the code to your machine
```bash
git clone https://github.com/LondheShubham153/two-tier-flask-app.git
```

Step 3: ☸️ Deploy


- MySQL

Prepare Persistent Storage:

```bash
mkdir mysqldata  # Required for PV
cd k8s
kubectl apply -f mysql-pv.yml
kubectl apply -f mysql-pvc.yml
```


```bash
kubectl apply -f mysql-deployment.yml
kubectl apply -f mysql-svc.yml
```


- Flask App:


Save MySQL ClusterIP in two-tier-deployment.yaml 

```bash
kubectl get svc -o wide  # Check Service name and ClusterIP
```

```bash
vi two-tier-app-deployment.yml
1. change image: YOUR-DOCKERHUB-USERNAME/flaskapp:latest # Paste your dockerhub name
2. change value: "10.98.19.211" # Paste your mysql-service ClusterIP
```

```bash

kubectl apply -f two-tier-app-deployment.yml
```
```bash
kubectl apply -f two-tier-app-svc.yml
```


Verify the app at: http://<EC2_PUBLIC_IP>:30004

