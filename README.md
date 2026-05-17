# Two-Tier Flask App — CI/CD Pipeline (Jenkins + ArgoCD + Kubernetes)

## Architecture Overview

```
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
            │  Git commit to manifest repo
            ▼
         ArgoCD
   (Auto-detects manifest change)
            │
            ▼
      Kubernetes (Minikube)
   ├── Flask App Pod
   └── MySQL Pod
```

**Flow:** Developer pushes code → GitHub Webhook triggers Jenkins → Jenkins builds & pushes Docker image → Jenkins updates image tag in manifest → ArgoCD detects change → ArgoCD syncs app to Kubernetes.

---

## Best Practices (Industry vs. This Setup)

| Area | This Setup (Learning) | Industry Standard |
|---|---|---|
| Cluster | Minikube | Managed K8s (EKS, GKE, AKS) |
| Repositories | Single repo (code + manifests) | Separate repos for code and manifests |
| External traffic | NodePort Service | Ingress + Load Balancer |

### Why These Decisions?

**1. Service name instead of ClusterIP**

```yaml
# ✅ Correct — service name is stable
- name: MYSQL_HOST
  value: "mysql-service"

# ❌ Wrong — ClusterIP changes when pod restarts
- name: MYSQL_HOST
  value: "10.98.19.211"
```

**2. `BUILD_NUMBER` for image tagging**

Each Jenkins build produces a unique, traceable image tag (e.g., `flaskapp:42`). This avoids overwriting and enables rollback.

**3. `hostPath` for MySQL persistence**

```yaml
hostPath:
  path: /home/ubuntu/two-tier-flask-app/mysqldata
```

> ⚠️ Create this directory on your host before deploying: `mkdir -p /home/ubuntu/two-tier-flask-app/mysqldata`

---

## Prerequisites

Before starting, make sure the following are running:

- [ ] Minikube cluster (`minikube start`)
- [ ] Jenkins server (accessible at `http://YOUR_JENKINS_IP:8080`)
- [ ] ArgoCD installed on the cluster
- [ ] DockerHub account
- [ ] GitHub account

---

## Step 1 — Code → GitHub

Fork and clone the repository:

```
https://github.com/bilalamjad-devops/two-tier-flask-app/tree/main
```

```bash
# Fork on GitHub first, then clone your fork
git clone https://github.com/YOUR_GITHUB_USERNAME/two-tier-flask-app.git
cd two-tier-flask-app
```

> **Note:** We are using a single repository for both application code and Kubernetes manifests. In production, these would be separate repositories.

Before proceeding, make the required replacements in two files:

**In `Jenkinsfile`**, replace:
- `YOUR_DOCKERHUB_USERNAME` → your DockerHub username
- `YOUR_GITHUB_EMAIL` → your GitHub email
- `YOUR_GITHUB_USERNAME` → your GitHub username

**In `k8s/two-tier-app-deployment.yml`**, replace:
- `YOUR_DOCKERHUB_USERNAME` → your DockerHub username (in the image field)

---

## Step 2 — CI Setup (Jenkins)

### 2a. Install Plugins

Go to **Jenkins → Manage Jenkins → Plugins → Available plugins** and install:

- Docker Pipeline
- Docker
- GitHub Integration
- Pipeline: GitHub Webhook

Restart Jenkins after installation.

### 2b. Add Credentials

Go to **Jenkins → Manage Jenkins → Credentials → System → Global credentials → Add Credentials**.

**DockerHub credentials:**

| Field | Value |
|---|---|
| Kind | Username with password |
| Username | Your DockerHub username |
| Password | Your DockerHub password |
| ID | `dockerhub` |

**GitHub credentials:**

| Field | Value |
|---|---|
| Kind | Username with password |
| Username | Your GitHub username |
| Password | Your GitHub Personal Access Token |
| ID | `github` |

> To create a GitHub PAT: GitHub → Settings → Developer settings → Personal access tokens → Generate new token. Grant `repo` scope.

### 2c. Create the Pipeline Job

1. Go to **Jenkins → New Item**
2. Enter a name (e.g., `two-tier-flask-pipeline`)
3. Select **Pipeline** → click **OK**

**Configure the job:**

Under **General**:
- ✅ Check **This project is parameterized**
- Add a **String Parameter**:
  - Name: `DOCKERTAG`
  - Default value: `latest`

Under **Pipeline**:
- Definition: `Pipeline script from SCM`
- SCM: `Git`
- Repository URL: `https://github.com/YOUR_GITHUB_USERNAME/two-tier-flask-app.git`
- Branch: `*/main`
- Script Path: `Jenkinsfile`

Click **Save**.

### 2d. Jenkinsfile

Place this file in the root of your repository. Replace the three placeholder values at the top.

```groovy
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
            // Push with Jenkins BUILD_NUMBER tag for traceability
            app.push("${env.BUILD_NUMBER}")
            // Also push latest tag
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
            // Configure Git identity
            sh "git config user.email '${gitEmail}'"
            sh "git config user.name '${gitUsername}'"

            // Update image tag in deployment manifest
            sh """
                sed -i 's+image:.*+image: ${imageName}:${BUILD_NUMBER}+g' k8s/two-tier-app-deployment.yml
            """

            // Commit and push updated manifest
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

## Step 3 — Pipeline Verification (Manual First Run)

Trigger the pipeline manually to confirm everything works before enabling the webhook.

1. Go to **Jenkins → your pipeline job → Build with Parameters**
2. Leave `DOCKERTAG` as `latest` and click **Build**
3. Monitor the Console Output

**Verify these two outcomes:**

- [ ] Image is visible on DockerHub: `https://hub.docker.com/r/YOUR_DOCKERHUB_USERNAME/flaskapp`
- [ ] Manifest file `k8s/two-tier-app-deployment.yml` in GitHub shows the new image tag (e.g., `flaskapp:1`)

If both pass, your CI pipeline is working correctly.

---

## Step 4 — CD Setup (ArgoCD)

### Connect ArgoCD to your manifest repository

1. Open the ArgoCD UI (usually `http://YOUR_MINIKUBE_IP:PORT`)
2. Go to **Applications → New App**

| Field | Value |
|---|---|
| Application Name | `two-tier-flask-app` |
| Project | `default` |
| Sync Policy | `Automatic` |
| Repository URL | `https://github.com/YOUR_GITHUB_USERNAME/two-tier-flask-app.git` |
| Revision | `main` |
| Path | `k8s` |
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace | `default` |

3. Click **Create**

ArgoCD will now watch the `k8s/` folder in your repository. When Jenkins updates the image tag and pushes to GitHub, ArgoCD will automatically detect the change and redeploy the application on Kubernetes.

---

## Step 5 — GitHub → Jenkins Webhook

This step enables automatic pipeline triggering on every code push.

1. Go to your GitHub repository → **Settings → Webhooks → Add webhook**

| Field | Value |
|---|---|
| Payload URL | `http://YOUR_JENKINS_IP:8080/github-webhook/` |
| Content type | `application/json` |
| Which events? | **Just the push event** |

2. Click **Add webhook**

> ⚠️ Your Jenkins server must be publicly accessible (or reachable from GitHub) for webhooks to work. If Jenkins is on a local/private network, use [ngrok](https://ngrok.com/) to expose it: `ngrok http 8080`

---

## Full Pipeline Verification

Once all 5 steps are complete, test the end-to-end flow:

1. Make a small change to your code (e.g., edit `app.py`)
2. Push to GitHub:
   ```bash
   git add .
   git commit -m "test: trigger CI/CD pipeline"
   git push origin main
   ```
3. Watch Jenkins automatically trigger the build
4. After Jenkins completes, watch ArgoCD sync the new image
5. Verify the updated app is running on Kubernetes:
   ```bash
   kubectl get pods
   kubectl get svc
   ```

---

## Troubleshooting

| Problem | Likely Cause | Fix |
|---|---|---|
| Jenkins build fails at Docker push | Wrong DockerHub credentials ID | Make sure credential ID is exactly `dockerhub` |
| Git push fails in Jenkins | PAT lacks repo scope or wrong credential ID | Recreate PAT with `repo` scope; credential ID must be `github` |
| ArgoCD not syncing | Wrong repo URL or path | Double-check repo URL and path is `k8s` |
| MySQL pod can't connect to Flask | Wrong `MYSQL_HOST` value | Must be `mysql-service`, not an IP |
| MySQL data lost on pod restart | `mysqldata` directory missing | Run `mkdir -p /home/ubuntu/two-tier-flask-app/mysqldata` on host |
| Webhook not triggering Jenkins | Jenkins not reachable from GitHub | Use ngrok or ensure public IP is accessible |
