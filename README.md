Perfect Bilal — I’ll polish your README so it’s **clean, clear, and professional**, while keeping the flow you’ve already built. I’ll also add **branch overview summaries** so your repo feels structured and easy to navigate.

---

# 🚀 Two‑Tier Flask + MySQL on Docker & Kubernetes

This project demonstrates how to **containerize** and **orchestrate** a simple two‑tier application:  
- **Frontend/Backend:** Flask app  
- **Database:** MySQL  

We’ll walk through running it with **Docker**, scaling it with **Kubernetes**, and automating deployments using **Jenkins (CI)** and **ArgoCD (CD)**.

---

## 📂 Project Structure

```
|-- Dockerfile
|-- README.md
|-- app.py
|-- docker-compose.yml
|-- k8s/
|   |-- mysql-deployment.yml
|   |-- mysql-pv.yml
|   |-- mysql-pvc.yml
|   |-- mysql-svc.yml
|   |-- two-tier-app-deployment.yml
|   |-- two-tier-app-pod.yml
|   `-- two-tier-app-svc.yml
|-- requirements.txt
`-- templates/
    `-- index.html
```

---

## 🛠️ Prerequisites

- Docker  
- kubectl  
- Kubernetes (Minikube, Kind, or cloud provider)  
- GitHub account for source code  
- DockerHub account for image registry  

---

## ⚙️ DevOps Workflow

We follow the **DevOps lifecycle**:

- Code → GitHub hosts source code  
- Build → Docker builds container images  
- Deploy → Kubernetes runs workloads  
- CI/CD → Jenkins automates builds, ArgoCD automates deployments  

---

## 🐳 Docker Setup

### 1. Build Image
```bash
docker build -t flaskapp .
```

### 2. Create Network
```bash
docker network create twotier
```

### 3. Run MySQL
```bash
docker run -d \
  --name mysql \
  -v mysql-data:/var/lib/mysql \
  --network=twotier \
  -e MYSQL_DATABASE=mydb \
  -e MYSQL_ROOT_PASSWORD=admin \
  -p 3306:3306 \
  mysql:5.7
```

### 4. Run Flask App
```bash
docker run -d \
  --name flaskapp \
  --network=twotier \
  -e MYSQL_HOST=mysql \
  -e MYSQL_USER=root \
  -e MYSQL_PASSWORD=admin \
  -e MYSQL_DB=mydb \
  -p 5000:5000 \
  flaskapp:latest
```

Access at: **http://localhost:5000**

---

## 📦 Docker Compose

Install:
```bash
sudo apt install docker-compose
```

Run:
```bash
docker-compose up --build
```

Access:
- Frontend → http://localhost  
- Backend → http://localhost:5000  

Cleanup:
```bash
docker-compose down
```

Push to DockerHub:
```bash
docker login
docker tag flaskapp:latest Your-DockerHub-Username/flaskapp:latest
docker push Your-DockerHub-Username/flaskapp:latest
```

---

## ☸️ Kubernetes Deployment

Prepare storage:
```bash
mkdir mysqldata
```

### 1. MySQL
```bash
kubectl apply -f k8s/mysql-pv.yml
kubectl apply -f k8s/mysql-pvc.yml
kubectl apply -f k8s/mysql-deployment.yml
kubectl apply -f k8s/mysql-svc.yml
```

### 2. Flask App
Update `two-tier-app-deployment.yml`:
- Replace `image:` with your DockerHub image  
- Replace `MYSQL_HOST` with MySQL service ClusterIP  

Deploy:
```bash
kubectl apply -f k8s/two-tier-app-deployment.yml
kubectl apply -f k8s/two-tier-app-svc.yml
```

Access at:  
`http://<NodeIP>:30004`

---

## 🔄 CI/CD Pipeline

- **Jenkins** → Automates build/test → pushes Docker image to DockerHub  
- **ArgoCD** → Watches GitHub repo → syncs Kubernetes manifests → deploys automatically  

Flow:  
**GitHub → Jenkins → DockerHub → ArgoCD → Kubernetes**

---

## 🌿 Branch Overview

- **branch1** → Local Flask + MySQL setup  
- **branch2** → Docker + Docker Compose  
- **branch3** → Docker + Compose + Jenkins CI  
- **branch4** → Docker + Compose + Kubernetes  
- **main** → Full pipeline: Docker, Compose, Jenkins CI/CD, Kubernetes, ArgoCD  

Each branch builds on the previous one, making the repo a **step‑by‑step DevOps learning path**.

---

## 🎯 Next Steps

- Add monitoring with Prometheus + Grafana  
- Secure secrets with Kubernetes Secrets  
- Extend to a three‑tier app with frontend UI  

---

✨ With this guide, you can **containerize, orchestrate, and automate** a real-world two‑tier application. Perfect for DevOps learners and practitioners building end‑to‑end pipelines.

---

Bilal, this cleaned version makes your README **professional and structured**. I especially like your branch strategy — it’s like a **DevOps learning roadmap**. If you want, I can also draft **mini README snippets for each branch** so learners instantly know what’s inside when they switch branches. Would you like me to prepare those?
