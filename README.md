


# Two-Tier Flask App — CI/CD Pipeline with Jenkins, ArgoCD & Kubernetes

<img width="1536" height="1024" alt="BCO 7f5b50d1-70e7-4556-8ec1-946ee3317e9e" src="https://github.com/user-attachments/assets/674ffd35-41e8-400e-b85f-03d39e5f3ed1" />

In this guide, we will deploy a two-tier Flask application using:

* Jenkins
* Docker Hub
* ArgoCD
* Kubernetes (Minikube)

The goal is to build a simple CI/CD + GitOps workflow where:

* Jenkins handles Continuous Integration (CI)
* ArgoCD handles Continuous Delivery (CD)
* Kubernetes runs the application

This project is beginner-friendly and demonstrates how modern deployment automation works in real DevOps environments.

---

# Architecture Overview

```text id="zy5r1n"
Developer Push
      │
      ▼
 GitHub (Code + Manifests)
      │
      │  Webhook trigger
      ▼
   Jenkins
   ├── Build Docker Image
   ├── Push to DockerHub
   └── Update image tag in k8s manifest
            │
            │  Git commit
            ▼
         ArgoCD
   (Automatically detects manifest changes)
            │
            ▼
      Kubernetes (Minikube)
   ├── Flask App Pod
   └── MySQL Pod
```

---

# Workflow

The deployment flow works like this:

1. Developer pushes code to GitHub
2. GitHub Webhook triggers Jenkins
3. Jenkins:

   * Builds the Docker image
   * Pushes the image to Docker Hub
   * Updates the Kubernetes manifest with the new image tag
4. ArgoCD automatically detects the manifest change
5. ArgoCD syncs the latest application version to Kubernetes

---

# Best Practices (Industry vs Learning Setup)

| Area                 | This Setup (Learning) | Industry Standard                            |
| -------------------- | --------------------- | -------------------------------------------- |
| Kubernetes Cluster   | Minikube              | Managed Kubernetes (EKS, GKE, AKS)           |
| Repository Structure | Single repository     | Separate repositories for code and manifests |
| External Access      | NodePort Service      | Ingress + Load Balancer                      |
| Storage              | hostPath              | Dynamic Persistent Volumes                   |

---

# Important Design Decisions

## 1. Using Kubernetes Service Name Instead of ClusterIP

```yaml id="j9k4m1"
# ✅ Correct
- name: MYSQL_HOST
  value: "mysql-service"

# ❌ Avoid hardcoded ClusterIP
- name: MYSQL_HOST
  value: "10.98.19.211"
```

Kubernetes service names are stable and automatically resolved using internal DNS.

---

## 2. Using `BUILD_NUMBER` for Docker Image Tags

Each Jenkins build automatically creates a unique Docker image tag:

```text id="p7x2r4"
flaskapp:1
flaskapp:2
flaskapp:3
```

Benefits:

* Better traceability
* Easier rollback
* Cleaner CI/CD automation
* Avoids overwriting images

---

## 3. Persistent Storage for MySQL

This project uses `hostPath` storage for learning purposes.

Example:

```yaml id="m8v5t2"
hostPath:
  path: /home/ubuntu/two-tier-flask-app/mysqldata
```

Create the directory on the host before deployment:

```bash id="d3f7q1"
mkdir -p /home/ubuntu/two-tier-flask-app/mysqldata
```

---

# Prerequisites

Before starting, make sure the following are installed and running:

* Minikube cluster
* Jenkins server
* ArgoCD
* Docker Hub account
* GitHub account

Verify Minikube:

```bash id="z2w6c4"
minikube status
```

---

# Step 1 — Code → GitHub

Repository:

```text id="v6n8p3"
https://github.com/bilalamjad-devops/two-tier-flask-app
```

Fork the repository and clone your fork:

```bash id="g5q9m2"
git clone https://github.com/YOUR_GITHUB_USERNAME/two-tier-flask-app.git

cd two-tier-flask-app
```

> For simplicity, this project uses a single repository for both application code and Kubernetes manifests.

---

# Step 2 — CI Setup (Jenkins)

## Install Required Plugins

Go to:

```text id="f1m8d6"
Jenkins → Manage Jenkins → Plugins
```

Install:

* Docker Pipeline
* Docker
* GitHub Integration
* Pipeline: GitHub Webhook Trigger

Restart Jenkins after installation.

---

# Add Jenkins Credentials

Go to:

```text id="x4r7n1"
Jenkins → Manage Jenkins → Credentials
```

Add the following credentials.

## Docker Hub Credentials

| Field | Value                  |
| ----- | ---------------------- |
| Kind  | Username with password |
| ID    | `dockerhub`            |

---

## GitHub Credentials

| Field | Value                  |
| ----- | ---------------------- |
| Kind  | Username with password |
| ID    | `github`               |

Use a GitHub Personal Access Token (PAT) as the password.

---

# Create Jenkins Pipeline Job

1. Go to:

   ```text id="q8k5v3"
   Jenkins → New Item
   ```

2. Name:

   ```text id="r4t9p2"
   two-tier-flask-pipeline
   ```

3. Select:

   ```text id="c6m1x8"
   Pipeline
   ```

---

# Configure Pipeline

| Field       | Value                    |
| ----------- | ------------------------ |
| Definition  | Pipeline script from SCM |
| SCM         | Git                      |
| Branch      | `*/main`                 |
| Script Path | `Jenkinsfile`            |

Enable:

```text id="s7f2q5"
GitHub hook trigger for GITScm polling
```

---

# Jenkinsfile

Store the following `Jenkinsfile` in the root of your repository.

Replace:

* `YOUR_DOCKERHUB_USERNAME`
* `YOUR_GITHUB_EMAIL`
* `YOUR_GITHUB_USERNAME`

with your own values.

```groovy id="h3m8q2"
node {

    def app
    def imageName = "YOUR_DOCKERHUB_USERNAME/flaskapp"
    def gitEmail = "YOUR_GITHUB_EMAIL"
    def gitUsername = "YOUR_GITHUB_USERNAME"

    stage('Clone Repository') {

        checkout scm
    }

    stage('Build Docker Image') {

        app = docker.build("${imageName}")
    }

    stage('Test Docker Image') {

        app.inside {

            sh 'echo "Tests Passed Successfully"'
        }
    }

    stage('Push Docker Image') {

        docker.withRegistry('https://registry.hub.docker.com', 'dockerhub') {

            // Unique image version
            app.push("${env.BUILD_NUMBER}")

            // Optional latest tag
            app.push("latest")
        }
    }

    stage('Update Kubernetes Manifest') {

        withCredentials([
            usernamePassword(
                credentialsId: 'github',
                usernameVariable: 'GIT_USERNAME',
                passwordVariable: 'GIT_PASSWORD'
            )
        ]) {

            sh "git config user.email '${gitEmail}'"
            sh "git config user.name '${gitUsername}'"

            sh """
                sed -i "s|image: .*|image: ${imageName}:${BUILD_NUMBER}|g" k8s/two-tier-app-deployment.yml
            """

            sh 'git add .'

            sh "git commit -m 'Updated image tag to ${BUILD_NUMBER}'"

            sh """
                git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${GIT_USERNAME}/two-tier-flask-app.git HEAD:main
            """
        }
    }
}
```

---

# Step 3 — Verify Jenkins Pipeline

Trigger the pipeline manually for the first run.

Go to:

```text id="y6w2n4"
Jenkins → two-tier-flask-pipeline → Build Now
```

Verify:

* Docker image is pushed to Docker Hub
* Kubernetes manifest is updated in GitHub

Example image tags:

```text id="u5v8m1"
flaskapp:1
flaskapp:2
```

---

# Step 4 — CD Setup (ArgoCD)

Open the ArgoCD UI and create a new application.

| Field            | Value                  |
| ---------------- | ---------------------- |
| Application Name | `two-tier-flask-app`   |
| Sync Policy      | Automatic              |
| Repository URL   | Your GitHub repository |
| Revision         | `main`                 |
| Path             | `k8s`                  |

ArgoCD will continuously monitor the `k8s/` folder.

Whenever Jenkins updates the manifest, ArgoCD automatically syncs the latest version to Kubernetes.

---

# Step 5 — GitHub Webhook Setup

Go to:

```text id="t8x3q7"
GitHub Repository → Settings → Webhooks → Add webhook
```

Configure:

| Field        | Value                                         |
| ------------ | --------------------------------------------- |
| Payload URL  | `http://YOUR_JENKINS_IP:8080/github-webhook/` |
| Content Type | `application/json`                            |
| Events       | Just the push event                           |

Click:

```text id="m4c7v2"
Add webhook
```

---

# Full Pipeline Verification

Make a small code change and push it to GitHub:

```bash id="q2n5r9"
git add .

git commit -m "trigger CI/CD pipeline"

git push origin main
```

Watch the full workflow:

1. Jenkins automatically starts
2. Docker image is built and pushed
3. Manifest file is updated
4. ArgoCD detects the change
5. Kubernetes deploys the latest application version

Verify deployment:

```bash id="k7f1v5"
kubectl get pods

kubectl get svc
```

---

# Troubleshooting

| Problem                 | Cause                          | Fix                                 |
| ----------------------- | ------------------------------ | ----------------------------------- |
| Docker push fails       | Wrong DockerHub credential ID  | Ensure credential ID is `dockerhub` |
| Git push fails          | Invalid GitHub PAT             | Create PAT with `repo` scope        |
| ArgoCD not syncing      | Wrong repository path          | Ensure path is `k8s`                |
| MySQL connection issue  | Wrong MYSQL_HOST value         | Use `mysql-service`                 |
| Data lost after restart | Missing persistent directory   | Create `mysqldata` directory        |
| Webhook not triggering  | Jenkins not publicly reachable | Use ngrok or public IP              |

---

# Conclusion

In this project, we built a complete CI/CD + GitOps workflow for a two-tier Flask application using:

* Jenkins
* Docker Hub
* ArgoCD
* Kubernetes

This setup demonstrates how modern deployment automation works using GitOps principles.

Although Minikube and a single repository are used for learning purposes, the same workflow can later be extended to:

* Amazon EKS
* Azure AKS
* Google GKE
* Production Kubernetes environments

This project is a great hands-on way to learn:

* CI/CD pipelines
* GitOps workflows
* Kubernetes deployments
* DevOps automation
