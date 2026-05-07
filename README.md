

# 🚀 Two‑Tier Flask + MySQL on Docker & Kubernetes (Main Branch)

This branch represents the **complete DevOps pipeline** for a two‑tier application:  
- **Frontend/Backend:** Flask app  
- **Database:** MySQL  

We’ll walk through running it with **Docker**, scaling it with **Kubernetes**, and automating deployments using **Jenkins (CI)**, **GitHub Webhooks**, and **ArgoCD (CD)**.

---

## 📂 Project Structure

```
|-- Dockerfile
|-- README.md
|-- app.py
|-- docker-compose.yml
|-- Jenkinsfile
|-- k8s/
|   |-- mysql-deployment.yml
|   |-- mysql-pv.yml
|   |-- mysql-pvc.yml
|   |-- mysql-svc.yml
|   |-- two-tier-app-deployment.yml
|   |-- two-tier-app-svc.yml
|-- requirements.txt
`-- templates/
    `-- index.html
```

---

## 🛠️ Prerequisites

- Docker  
- kubectl  
- Kubernetes  
- GitHub account for source code  
- DockerHub account for image registry  
- Jenkins server with GitHub integration  
- ArgoCD installed in Kubernetes cluster  

---

## ⚙️ DevOps Workflow

We follow the **GitOps lifecycle**:

- Code → GitHub hosts source code  
- Webhook → Triggers Jenkins pipeline on commit  
- Build → Jenkins builds Docker image and pushes to DockerHub  
- Deploy → ArgoCD syncs manifests from GitHub and deploys to Kubernetes  

---

## 🐳 Docker Setup

Same as branch4: build, run, and test locally. Once verified, push to DockerHub for Kubernetes deployment.

---

## 📦 Jenkins CI Pipeline

The **Jenkinsfile** automates:
1. Clone repo from GitHub  
2. Build Docker image  
3. Run tests (optional)  
4. Push image to DockerHub  
5. Update Kubernetes manifests with new image tag  

Example snippet:
```groovy
pipeline {
  agent any
  stages {
    stage('Build') {
      steps {
        sh 'docker build -t your-username/flaskapp:${BUILD_NUMBER} .'
      }
    }
    stage('Push') {
      steps {
        sh 'docker login -u $DOCKER_USER -p $DOCKER_PASS'
        sh 'docker push your-username/flaskapp:${BUILD_NUMBER}'
      }
    }
    stage('Update Manifests') {
      steps {
        sh 'sed -i "s|image:.*|image: your-username/flaskapp:${BUILD_NUMBER}|g" k8s/two-tier-app-deployment.yml'
        sh 'git commit -am "Update image tag to ${BUILD_NUMBER}"'
        sh 'git push origin main'
      }
    }
  }
}
```

---

## 🔔 GitHub Webhook

- Configure a **GitHub webhook** to trigger Jenkins on every push to `main`.  
- Webhook URL: `http://<jenkins-server>/github-webhook/`  
- Event: `push`  

This ensures every commit automatically starts the CI pipeline.

---

## ☸️ ArgoCD Continuous Deployment

- Install ArgoCD in your Kubernetes cluster.  
- Create an **Application manifest** pointing to your GitHub repo:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: two-tier-app
spec:
  project: default
  source:
    repoURL: 'https://github.com/your-username/your-repo.git'
    targetRevision: main
    path: k8s
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

- ArgoCD will automatically sync manifests whenever Jenkins updates them.

---

## 🔄 CI/CD Pipeline Flow

**GitHub → Webhook → Jenkins → DockerHub → ArgoCD → Kubernetes**

1. Developer pushes code to GitHub  
2. GitHub webhook triggers Jenkins  
3. Jenkins builds & pushes Docker image to DockerHub  
4. Jenkins updates Kubernetes manifests in GitHub  
5. ArgoCD detects changes and deploys automatically  

---

## 🌿 Branch Overview

- **branch1** → Local Flask + MySQL setup  
- **branch2** → Docker + Docker Compose  
- **branch3** → Docker + Compose + Jenkins CI  
- **branch4** → Docker + Compose + Kubernetes  
- **main** → Full pipeline: Docker, Compose, Jenkins CI/CD, GitHub Webhook, Kubernetes, ArgoCD  

---

## 🎯 Next Steps

- Add monitoring with Prometheus + Grafana  
- Secure secrets with Kubernetes Secrets  
- Extend pipeline with automated tests and notifications  

---

✨ This **main branch** completes the series: a full **GitOps pipeline** from code commit to automated deployment. Ideal for DevOps learners building real-world CI/CD workflows.

---

Bilal, this now makes your repo a **complete DevOps learning roadmap**. Would you like me to also prepare a **compact visual diagram** for the main branch showing GitHub → Jenkins → DockerHub → ArgoCD → Kubernetes → Users? That would make the final README even more engaging.
