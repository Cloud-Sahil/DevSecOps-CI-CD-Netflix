# DevSecOps CI/CD — Netflix Clone Deployment Guide

> **Project:** Deploy a Secure Netflix Clone using DevSecOps Pipeline  
> **Source:** [https://github.com/Bijan1235/Netflix-clone.git](https://github.com/jason-liu22/netflix-clone-react-typescript.git)  
> **Stack:** Jenkins · Docker · SonarQube · Trivy · OWASP · Prometheus · Grafana · Kubernetes  
> **Server:** AWS EC2 Ubuntu 24.04 T2 Large

---

## Architecture Overview

```
Developer (Git Push)
        │
        ▼
   GitHub Repo
        │
        ▼
   Jenkins CI/CD Pipeline
   ┌────────────────────────────────────────────────────────────────┐
   │                                                                │
   │  Checkout → SonarQube Analysis → OWASP Check → Trivy FS Scan  │
   │       → Docker Build → Trivy Image Scan → Docker Push         │
   │       → Deploy Container  →  Deploy to Kubernetes             │
   │                                                                │
   └────────────────────────────────────────────────────────────────┘
        │                          │
        ▼                          ▼
   Docker Hub               Email Notification
        │
        ▼
   Running Netflix App (Port 8081)
        │
        ▼
   Prometheus (Port 9090) → Grafana (Port 3000)
```

---

## Infrastructure Setup

```
Server 1: Jenkins + Docker + SonarQube + Trivy + App
  Type: AWS EC2 t2.large (Ubuntu 24.04)
  Ports: 8080 (Jenkins), 9000 (SonarQube), 8081 (App)

Server 2: Prometheus + Grafana + Node Exporter
  Type: AWS EC2 t2.medium (Ubuntu 22.04)
  Ports: 9090 (Prometheus), 3000 (Grafana), 9100 (Node Exporter)

Server 3 & 4: Kubernetes Master + Worker
  Type: AWS EC2 t2.medium (Ubuntu 20.04)
  Ports: 6443 (K8s API), 30007 (NodePort)
```

---

## Step 1 — Launch AWS EC2 Instance (Ubuntu 24.04 T2 Large)

```
AWS Console → EC2 → Launch Instance

Configuration:
  Name:           Netflix-DevSecOps-Server
  AMI:            Ubuntu Server 24.04 LTS (HVM), SSD Volume Type
  Instance Type:  t2.large  (2 vCPU, 8 GB RAM)
  Key Pair:       Create new → Download .pem file → Keep it safe
  
  Security Group → Add inbound rules:
  ┌─────────┬──────────┬──────────────────────────┐
  │  Port   │ Protocol │ Description              │
  ├─────────┼──────────┼──────────────────────────┤
  │  22     │ TCP      │ SSH access               │
  │  8080   │ TCP      │ Jenkins UI               │
  │  9000   │ TCP      │ SonarQube                │
  │  8081   │ TCP      │ Netflix App              │
  │  9090   │ TCP      │ Prometheus               │
  │  3000   │ TCP      │ Grafana                  │
  │  9100   │ TCP      │ Node Exporter            │
  │  6443   │ TCP      │ Kubernetes API           │
  │  30007  │ TCP      │ K8s NodePort (App)       │
  └─────────┴──────────┴──────────────────────────┘

Storage:
  Root volume: 30 GB (gp3)

Click: Launch Instance
```

**SSH into the server:**

```bash
# Fix key permissions first
chmod 400 your-key.pem

# SSH into EC2
ssh -i your-key.pem ubuntu@<EC2-PUBLIC-IP>
```

---

## Step 2 — Install Jenkins, Docker, and Trivy

### 2A — Install Jenkins

```bash
# Update system first
sudo apt update && sudo apt upgrade -y

# Install Java (Jenkins requires Java 17)
sudo apt install openjdk-17-jre -y

# Verify Java
java -version

# Add Jenkins repository key
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key

# Add Jenkins repository
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install Jenkins
sudo apt-get update
sudo apt-get install jenkins -y

# Start and enable Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Check Jenkins status
sudo systemctl status jenkins

# Get initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

**Access Jenkins:** `http://<EC2-PUBLIC-IP>:8080`

---

### 2B — Install Docker

```bash
# Add Docker's official GPG key
sudo apt-get update
sudo apt-get install ca-certificates curl -y
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin -y

# Give Jenkins and Ubuntu user Docker permission
sudo chmod 666 /var/run/docker.sock

# Verify Docker
docker --version
docker ps
```

---

### 2C — Install Trivy (Security Scanner)

```bash
# Add Trivy repository
sudo apt-get install wget apt-transport-https gnupg lsb-release -y

wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key \
  | sudo apt-key add -

echo "deb https://aquasecurity.github.io/trivy-repo/deb \
  $(lsb_release -sc) main" \
  | sudo tee -a /etc/apt/sources.list.d/trivy.list

# Install Trivy
sudo apt-get update
sudo apt-get install trivy -y

# Verify Trivy
trivy --version
```

---

### 2D — Run SonarQube as Docker Container

```bash
# Pull and run SonarQube (community edition)
docker run -d \
  --name sonar \
  -p 9000:9000 \
  sonarqube:lts-community

# Check it's running
docker ps

# Verify SonarQube is up (wait 1-2 minutes for startup)
curl -s http://localhost:9000/api/system/status | python3 -m json.tool
```

**Access SonarQube:** `http://<EC2-PUBLIC-IP>:9000`  
**Default credentials:** `admin` / `admin`  
(You will be prompted to change password on first login)

---

## Step 3 — Get TMDB API Key

TMDB (The Movie Database) API key is required for the Netflix Clone to show actual movie content.

```
1. Open browser → https://www.themoviedb.org/
2. Click "Sign Up" (top right) → Create a free account
3. After login → Click your profile icon → "Settings"
4. Left sidebar → "API"
5. Click "Create" → Choose "Developer" → Fill in the form
6. Copy your API Key (v3 auth)

   API Key looks like: abc123def456ghi789jkl
   
   Keep this — you'll need it in the Dockerfile / Jenkins pipeline
```

---

## Step 4 — Install Prometheus and Grafana (Monitoring Server)

> Do this on a **separate EC2 instance** (t2.medium, Ubuntu 22.04)

### 4A — Install Prometheus

```bash
# Update system
sudo apt update

# Create prometheus user (no login shell)
sudo useradd --system --no-create-home --shell /bin/false prometheus

# Download Prometheus (latest stable)
wget https://github.com/prometheus/prometheus/releases/download/v2.53.1/prometheus-2.53.1.linux-amd64.tar.gz

# Extract files
tar -xvf prometheus-2.53.1.linux-amd64.tar.gz

# Move into directory
cd prometheus-2.53.1.linux-amd64/

# Create directories
sudo mkdir -p /data /etc/prometheus

# Move binaries and config
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml

# Set ownership
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/

# Create systemd service file
sudo nano /etc/systemd/system/prometheus.service
```

**Paste this content into prometheus.service:**

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start Prometheus
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus

# Access Prometheus
# http://<MONITORING-SERVER-IP>:9090
```

---

### 4B — Install Node Exporter (on Jenkins server)

Node Exporter sends system metrics (CPU, memory, disk) to Prometheus.

```bash
# Create node_exporter user
sudo useradd --system --no-create-home --shell /bin/false node_exporter

# Download Node Exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.2/node_exporter-1.8.2.linux-amd64.tar.gz

# Extract and move binary
tar -xvf node_exporter-1.8.2.linux-amd64.tar.gz
sudo mv node_exporter-1.8.2.linux-amd64/node_exporter /usr/local/bin/

# Cleanup
rm -rf node_exporter*

# Create systemd service
sudo nano /etc/systemd/system/node_exporter.service
```

**Paste this content:**

```ini
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start Node Exporter
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter

# Node Exporter metrics available at:
# http://<JENKINS-SERVER-IP>:9100/metrics
```

---

### 4C — Configure Prometheus to Scrape Targets

```bash
# Edit prometheus config on monitoring server
sudo vim /etc/prometheus/prometheus.yml
```

**Replace the scrape_configs section with:**

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node_export'
    static_configs:
      - targets: ['<JENKINS-SERVER-PUBLIC-IP>:9100']

  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['<JENKINS-SERVER-PUBLIC-IP>:8080']
```

```bash
# Validate the config
promtool check config /etc/prometheus/prometheus.yml

# Reload Prometheus config (no restart needed)
curl -X POST http://localhost:9090/-/reload

# Check targets are UP
# http://<MONITORING-SERVER-IP>:9090/targets
```

---

### 4D — Install Grafana

```bash
# Install dependencies
sudo apt-get install -y apt-transport-https software-properties-common

# Add Grafana GPG key
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -

# Add Grafana repository
echo "deb https://packages.grafana.com/oss/deb stable main" \
  | sudo tee -a /etc/apt/sources.list.d/grafana.list

# Install Grafana
sudo apt-get update
sudo apt-get install grafana -y

# Enable and start Grafana
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server

# Access Grafana
# http://<MONITORING-SERVER-IP>:3000
# Default login: admin / admin
```

**Configure Grafana Data Source:**

```
1. Login to Grafana → http://<MONITORING-IP>:3000 (admin/admin)
2. Left sidebar → "Connections" → "Data Sources"
3. Click "Add data source" → Select "Prometheus"
4. HTTP URL: http://<MONITORING-SERVER-IP>:9090
5. Click "Save & Test" → Should show "Data source is working"
```

**Import Dashboards:**

```
Dashboard 1: Node Exporter Full
  Grafana → Dashboards → Import → ID: 1860 → Load → Select Prometheus → Import

Dashboard 2: Jenkins Performance
  Grafana → Dashboards → Import → ID: 9964 → Load → Select Prometheus → Import
```

---

## Step 5 — Install Prometheus Plugin in Jenkins

```
Jenkins Dashboard
  → Manage Jenkins
  → Plugins
  → Available Plugins
  → Search: "Prometheus"
  → Install: "Prometheus Metrics Plugin"
  → Restart Jenkins after install

After restart:
  → Manage Jenkins → System → Prometheus
  → Path: prometheus
  → Check all "Count duration of builds" options
  → Save

Verify at: http://<JENKINS-IP>:8080/prometheus
```

---

## Step 6 — Email Integration With Jenkins

### 6A — Generate Gmail App Password

```
1. Go to: https://myaccount.google.com/security
2. Ensure 2-Step Verification is ON
3. Search "App passwords" → Click App passwords
4. App name: Jenkins → Generate
5. Copy the 16-character password (e.g., "bkec mhur oddp hppw")
   → You'll use this as your Jenkins SMTP password
```

### 6B — Configure Email in Jenkins

```
Jenkins → Manage Jenkins → System

Scroll to: "E-mail Notification"
  SMTP Server:            smtp.gmail.com
  Default user e-mail:    your-email@gmail.com
  
  Click: Advanced
  ✅ Use SMTP Authentication
  Username:               your-email@gmail.com
  Password:               <16-character App Password>
  ✅ Use SSL
  SMTP Port:              465

Scroll to: "Extended E-mail Notification"
  SMTP Server:            smtp.gmail.com
  SMTP Port:              465
  Credentials:            Add → Jenkins → Username with password
                          Username: your-email@gmail.com
                          Password: <App Password>
  ✅ Use SSL

Click Save
```

---

## Step 7 — Install Jenkins Plugins

```
Jenkins → Manage Jenkins → Plugins → Available Plugins

Install these plugins (select all, then "Install without restart"):

Security & Code Quality:
  ✅ SonarQube Scanner
  ✅ OWASP Dependency-Check

Build Tools:
  ✅ Eclipse Temurin Installer  (for JDK)
  ✅ NodeJS Plugin

Docker:
  ✅ Docker Commons
  ✅ Docker Pipeline
  ✅ Docker API
  ✅ docker-build-step

Notifications:
  ✅ Email Extension Template

After installation → Click "Restart Jenkins when installation is complete"
```

### 7A — Configure Tools in Jenkins

```
Jenkins → Manage Jenkins → Tools

JDK Installation:
  Name:    JDK17
  ✅ Install automatically
  Version: jdk-17.0.8.1+1

NodeJS Installation:
  Name:    node22
  ✅ Install automatically
  Version: NodeJS 22.x

SonarQube Scanner Installation:
  Name:    sonar-scanner
  ✅ Install automatically
  Version: SonarQube Scanner 5.x

Docker Installation:
  Name:    docker
  ✅ Install automatically
  Version: latest

Click Save
```

### 7B — Configure SonarQube in Jenkins

**First — Get SonarQube token:**

```
SonarQube → http://<EC2-IP>:9000
  → Login as admin
  → Administration → Security → Users
  → Click on "admin" → Tokens → Generate Token
  → Name: jenkins-token → Generate → Copy the token
```

**Add to Jenkins credentials:**

```
Jenkins → Manage Jenkins → Credentials
  → System → Global credentials → Add credentials
  → Kind: Secret text
  → Secret: <SonarQube token from above>
  → ID: Sonar-token
  → Description: SonarQube Token
  → Create
```

**Add SonarQube server:**

```
Jenkins → Manage Jenkins → System
  → SonarQube servers → Add SonarQube
  → Name: sonar-server
  → Server URL: http://<EC2-IP>:9000
  → Server authentication token: Sonar-token (credential created above)
  → Save
```

### 7C — Add Docker Hub Credentials

```
Docker Hub → https://hub.docker.com → Login → Account Settings → Security
  → New Access Token → Name: jenkins → Generate → Copy token

Jenkins → Manage Jenkins → Credentials
  → System → Global credentials → Add credentials
  → Kind: Username with password
  → Username: <your-dockerhub-username>
  → Password: <dockerhub-access-token>
  → ID: docker
  → Description: Docker Hub Credentials
  → Create
```

---

## Step 8 — Create Jenkins Pipeline

```
Jenkins Dashboard → New Item
  → Name: Netflix
  → Select: Pipeline
  → Click OK
```

**Pipeline Configuration:**

```
General:
  ✅ Discard old builds
  Max # of builds to keep: 2

Build Triggers:
  ✅ GitHub hook trigger for GITScm polling

Pipeline:
  Definition: Pipeline script (paste the Jenkinsfile below)
```

**Jenkinsfile (Declarative Pipeline):**

```groovy
pipeline {
    agent any

    tools {
        jdk 'JDK17'
        nodejs 'node22'
    }

    environment {
        SCANNER_HOME        = tool 'sonar-scanner'
        DOCKER_IMAGE        = "your-dockerhub-username/netflix:latest"
        SONAR_PROJECT_KEY   = "Netflix"
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout from Git') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Bijan1235/Netflix-clone.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                        $SCANNER_HOME/bin/sonar-scanner \
                          -Dsonar.projectName=Netflix \
                          -Dsonar.projectKey=Netflix
                    """
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false,
                                       credentialsId: 'Sonar-token'
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit',
                                odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Trivy File System Scan') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker',
                                       toolName: 'docker') {
                        sh """
                            docker build \
                              --build-arg TMDB_V3_API_KEY=<YOUR-TMDB-API-KEY> \
                              -t netflix .
                            docker tag netflix ${DOCKER_IMAGE}
                            docker push ${DOCKER_IMAGE}
                        """
                    }
                }
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh """
                    trivy image ${DOCKER_IMAGE} > trivyimage.txt
                """
            }
        }

        stage('Deploy Docker Container') {
            steps {
                sh """
                    docker stop netflix 2>/dev/null || true
                    docker rm netflix 2>/dev/null || true
                    docker run -d \
                      --name netflix \
                      -p 8081:80 \
                      ${DOCKER_IMAGE}
                """
            }
        }
    }

    post {
        always {
            emailext(
                attachLog: true,
                subject: "'${currentBuild.result}' - ${env.JOB_NAME} Build #${env.BUILD_NUMBER}",
                body: """
                    <h2>Project: ${env.JOB_NAME}</h2>
                    <p>Build Number: ${env.BUILD_NUMBER}</p>
                    <p>Build Status: <b>${currentBuild.result}</b></p>
                    <p>Build URL: <a href='${env.BUILD_URL}'>${env.BUILD_URL}</a></p>
                """,
                to: 'your-email@gmail.com',
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
            )
        }
    }
}
```

> ⚠️ Replace `your-dockerhub-username` and `<YOUR-TMDB-API-KEY>` with your actual values.

---

## Step 9 — Install OWASP Dependency Check Plugin

```
Jenkins → Manage Jenkins → Plugins → Available
  → Search: "OWASP Dependency-Check"
  → Install → Restart Jenkins

Configure Tool:
  Jenkins → Manage Jenkins → Tools
  → Dependency-Check installations → Add
  → Name: DP-Check
  ✅ Install automatically
  → Save
```

---

## Step 10 — Docker Image Build and Push

This is handled automatically by the Jenkins pipeline (Step 8).

**Manual commands if needed:**

```bash
# Navigate to project directory
cd /var/lib/jenkins/workspace/Netflix

# Build Docker image with TMDB API key
docker build \
  --build-arg TMDB_V3_API_KEY=<your-api-key> \
  -t netflix .

# Tag for Docker Hub
docker tag netflix your-dockerhub-username/netflix:latest

# Login to Docker Hub
docker login -u your-dockerhub-username -p <access-token>

# Push image
docker push your-dockerhub-username/netflix:latest

# Verify push
docker pull your-dockerhub-username/netflix:latest
```

---

## Step 11 — Deploy the Netflix App Using Docker

```bash
# Stop and remove existing container (if any)
docker stop netflix 2>/dev/null || true
docker rm   netflix 2>/dev/null || true

# Run the Netflix app container
docker run -d \
  --name netflix \
  -p 8081:80 \
  your-dockerhub-username/netflix:latest

# Check it's running
docker ps

# View logs
docker logs netflix

# Access the app
# http://<EC2-PUBLIC-IP>:8081
```

---

## Step 12 — Kubernetes Master and Worker Node Setup

> Use **separate EC2 t2.medium Ubuntu 20.04** instances for K8s  
> Do these steps on **BOTH** master and worker nodes unless noted.

### On BOTH Master and Worker:

```bash
# Update system
sudo apt-get update && sudo apt-get upgrade -y

# Install Docker (same commands as Step 2B)
sudo apt-get install docker.io -y
sudo chmod 666 /var/run/docker.sock
sudo systemctl enable docker
sudo systemctl start docker

# Add Kubernetes repository
sudo apt-get install -y apt-transport-https ca-certificates curl

sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] \
  https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install kubeadm, kubelet, kubectl
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Disable swap (Kubernetes requirement)
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Enable required kernel modules
sudo modprobe br_netfilter
echo "br_netfilter" | sudo tee /etc/modules-load.d/br_netfilter.conf

# Sysctl settings for Kubernetes
sudo tee /etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system
```

### On MASTER Node Only:

```bash
# Initialize Kubernetes cluster
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --ignore-preflight-errors=all

# Configure kubectl for current user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Flannel network plugin (CNI)
kubectl apply -f \
  https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

# Verify master is Ready (takes 1-2 minutes)
kubectl get nodes

# Get join command for worker nodes (COPY THIS OUTPUT)
kubeadm token create --print-join-command
```

### On WORKER Node Only:

```bash
# Paste the join command from master (example format):
sudo kubeadm join <MASTER-IP>:6443 \
  --token <token-value> \
  --discovery-token-ca-cert-hash sha256:<hash-value>
```

### Back on MASTER — Verify cluster:

```bash
# Check all nodes are Ready
kubectl get nodes

# Expected output:
# NAME     STATUS   ROLES           AGE   VERSION
# master   Ready    control-plane   5m    v1.29.x
# worker   Ready    <none>          2m    v1.29.x
```

---

### Deploy Netflix App to Kubernetes

**Create deployment file:**

```bash
sudo nano /home/ubuntu/netflix-deployment.yml
```

**Paste this content:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: netflix-app
  labels:
    app: netflix
spec:
  replicas: 2
  selector:
    matchLabels:
      app: netflix
  template:
    metadata:
      labels:
        app: netflix
    spec:
      containers:
      - name: netflix
        image: your-dockerhub-username/netflix:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: netflix-service
spec:
  selector:
    app: netflix
  type: NodePort
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30007
```

```bash
# Apply the deployment
kubectl apply -f netflix-deployment.yml

# Check deployment status
kubectl get deployments
kubectl get pods
kubectl get services

# Access Netflix App on Kubernetes
# http://<WORKER-NODE-PUBLIC-IP>:30007
```

---

## Step 13 — Access Netflix App in Browser

| Service | URL |
|---------|-----|
| **Netflix App (Docker)** | `http://<EC2-PUBLIC-IP>:8081` |
| **Netflix App (Kubernetes)** | `http://<WORKER-NODE-IP>:30007` |
| **Jenkins** | `http://<EC2-PUBLIC-IP>:8080` |
| **SonarQube** | `http://<EC2-PUBLIC-IP>:9000` |
| **Prometheus** | `http://<MONITORING-IP>:9090` |
| **Grafana** | `http://<MONITORING-IP>:3000` |
| **Node Exporter** | `http://<JENKINS-IP>:9100/metrics` |

---

## Step 14 — Terminate AWS EC2 Instances

When you are done with the project, clean up to avoid charges.

```
AWS Console → EC2 → Instances

Select each instance → Instance state → Terminate instance

Confirm termination for:
  ✅ Jenkins / DevSecOps Server
  ✅ Monitoring Server (Prometheus + Grafana)
  ✅ Kubernetes Master Node
  ✅ Kubernetes Worker Node

Also clean up:
  Elastic IPs  → Release any unattached Elastic IPs
  Volumes      → Delete any orphaned EBS volumes
  Key Pairs    → (Optional) Delete unused key pairs
  Security     → (Optional) Delete custom security groups
  Groups

Verify in Billing:
  AWS → Billing → Cost Explorer
  Confirm no ongoing charges
```

---

## All Commands — Quick Reference

```bash
# ═══════════════════════════════════════════════
# SYSTEM
# ═══════════════════════════════════════════════
sudo apt update && sudo apt upgrade -y

# ═══════════════════════════════════════════════
# JENKINS
# ═══════════════════════════════════════════════
sudo apt install openjdk-17-jre -y
sudo systemctl start jenkins
sudo systemctl status jenkins
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

# ═══════════════════════════════════════════════
# DOCKER
# ═══════════════════════════════════════════════
sudo chmod 666 /var/run/docker.sock
docker ps
docker images
docker logs netflix
docker stop netflix && docker rm netflix

# Run SonarQube
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community

# Run Netflix App manually
docker run -d --name netflix -p 8081:80 your-dockerhub-username/netflix:latest

# ═══════════════════════════════════════════════
# TRIVY SCANS
# ═══════════════════════════════════════════════
# File system scan
trivy fs . > trivyfs.txt
cat trivyfs.txt

# Image scan
trivy image your-dockerhub-username/netflix:latest > trivyimage.txt
cat trivyimage.txt

# ═══════════════════════════════════════════════
# SONARQUBE
# ═══════════════════════════════════════════════
docker ps | grep sonar
docker logs sonar
# Access: http://<IP>:9000  (admin/admin)

# ═══════════════════════════════════════════════
# PROMETHEUS
# ═══════════════════════════════════════════════
sudo systemctl start prometheus
sudo systemctl status prometheus
promtool check config /etc/prometheus/prometheus.yml
curl -X POST http://localhost:9090/-/reload
# Access: http://<IP>:9090

# ═══════════════════════════════════════════════
# NODE EXPORTER
# ═══════════════════════════════════════════════
sudo systemctl start node_exporter
sudo systemctl status node_exporter
# Metrics: http://<IP>:9100/metrics

# ═══════════════════════════════════════════════
# GRAFANA
# ═══════════════════════════════════════════════
sudo systemctl start grafana-server
sudo systemctl status grafana-server
# Access: http://<IP>:3000  (admin/admin)

# Dashboard IDs to import:
# 1860 → Node Exporter Full
# 9964 → Jenkins Performance and Health Overview

# ═══════════════════════════════════════════════
# KUBERNETES
# ═══════════════════════════════════════════════
kubectl get nodes
kubectl get pods
kubectl get services
kubectl get deployments

kubectl apply -f netflix-deployment.yml
kubectl delete -f netflix-deployment.yml
kubectl describe pod <pod-name>
kubectl logs <pod-name>

# Get join command for worker
kubeadm token create --print-join-command

# ═══════════════════════════════════════════════
# DOCKER BUILD (manual)
# ═══════════════════════════════════════════════
docker build --build-arg TMDB_V3_API_KEY=<api-key> -t netflix .
docker tag netflix your-dockerhub-username/netflix:latest
docker login
docker push your-dockerhub-username/netflix:latest
```

---

## Port Reference Table

| Port | Service | Server | Access |
|------|---------|--------|--------|
| 22 | SSH | All | Admin only |
| 8080 | Jenkins UI | Jenkins Server | Team |
| 9000 | SonarQube | Jenkins Server | Team |
| 8081 | Netflix App (Docker) | Jenkins Server | Public |
| 9090 | Prometheus | Monitoring Server | Admin |
| 3000 | Grafana | Monitoring Server | Admin |
| 9100 | Node Exporter | Jenkins Server | Prometheus |
| 6443 | Kubernetes API | K8s Master | Admin |
| 30007 | Netflix App (K8s) | K8s Worker | Public |

---

## Troubleshooting Quick Fixes

```
PROBLEM: Jenkins not accessible at :8080
FIX:     sudo systemctl restart jenkins
         sudo ufw allow 8080/tcp

PROBLEM: SonarQube not starting
FIX:     docker logs sonar
         docker restart sonar
         # SonarQube needs ~2 min to start

PROBLEM: Docker permission denied
FIX:     sudo chmod 666 /var/run/docker.sock
         sudo usermod -aG docker jenkins
         sudo systemctl restart jenkins

PROBLEM: Pipeline fails at Docker build
FIX:     Check TMDB API key is correct and valid
         Verify Docker Hub credentials in Jenkins

PROBLEM: Trivy scan fails
FIX:     trivy image --timeout 5m your-image:tag
         sudo apt-get update && sudo apt-get install trivy -y

PROBLEM: K8s node not Ready
FIX:     kubectl describe node <node-name>
         sudo systemctl restart kubelet
         sudo swapoff -a

PROBLEM: Netflix app shows blank/no movies
FIX:     TMDB API key is wrong or missing
         Rebuild Docker image with correct API key:
         docker build --build-arg TMDB_V3_API_KEY=<correct-key> -t netflix .

PROBLEM: Prometheus targets DOWN
FIX:     Check firewall: sudo ufw allow 9100/tcp
         Verify IP in prometheus.yml is correct
         promtool check config /etc/prometheus/prometheus.yml
         curl -X POST http://localhost:9090/-/reload
```

---

## Security Scan Summary

| Tool | What It Scans | When It Runs |
|------|-------------|-------------|
| **SonarQube** | Code quality, bugs, vulnerabilities, code smells | Stage 3 — after checkout |
| **OWASP Dependency Check** | Known CVEs in npm/Maven dependencies | Stage 5 — after npm install |
| **Trivy FS** | File system — misconfigs, secrets in code | Stage 6 — before Docker build |
| **Trivy Image** | Docker image — OS packages, libraries CVEs | Stage 9 — after image push |

---

