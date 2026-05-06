


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

Simplify with `docker-compose.yml`:

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

---

## ☸️ Kubernetes Deployment

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

```bash
kubectl apply -f k8s/two-tier-app-deployment.yml
kubectl apply -f k8s/two-tier-app-svc.yml
```

Access at:  
`http://<NodeIP>:30004`

---

## 🔄 CI/CD Pipeline

- **Jenkins (CI):** Automates build/test → pushes Docker image to DockerHub  
- **ArgoCD (CD):** Watches GitHub repo → syncs Kubernetes manifests → deploys automatically  

Flow:  
**GitHub → Jenkins → DockerHub → ArgoCD → Kubernetes**

---

## 📊 Monitoring & Operations

For production readiness:
- Add Prometheus + Grafana for metrics  
- Use Kubernetes Secrets for sensitive data  
- Enable Logging with ELK stack or Loki  

---

## 🎯 Next Steps

- Extend to a **three‑tier app** with frontend UI  
- Add **CI/CD pipeline scripts** in Jenkinsfile and ArgoCD Application manifests  
- Integrate **Keycloak** for authentication  

---

✨ With this guide, you can **containerize, orchestrate, and automate** a real-world two‑tier application. Perfect for DevOps learners and practitioners building end‑to‑end pipelines.

---

Bilal, do you want me to also prepare a **diagram (architecture flow)** showing:  
GitHub → Jenkins → DockerHub → ArgoCD → Kubernetes? That would make your README visually engaging.
