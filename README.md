Hey! Here's a **comprehensive and professional README.md** for this DevSecOps project. It covers setup, deployment, security, CI/CD, monitoring, Kubernetes integration, and more.

---

````markdown
# DevSecOps Project - Netflix Clone Deployment with CI/CD, Security, and Monitoring

## Table of Contents

- [Project Overview](#project-overview)
- [Phase 1: Initial Setup and Deployment](#phase-1-initial-setup-and-deployment)
- [Phase 2: Security Integration](#phase-2-security-integration)
- [Phase 3: CI/CD Pipeline with Jenkins](#phase-3-cicd-pipeline-with-jenkins)
- [Phase 4: Monitoring with Prometheus and Grafana](#phase-4-monitoring-with-prometheus-and-grafana)
- [Phase 5: Notifications](#phase-5-notifications)
- [Phase 6: Kubernetes & ArgoCD Deployment](#phase-6-kubernetes--argocd-deployment)
- [Phase 7: Cleanup](#phase-7-cleanup)
- [License](#license)

---

## Project Overview

This project demonstrates a full DevSecOps pipeline implementation using a Netflix Clone application. The process includes application deployment on AWS EC2, securing the code with tools like SonarQube, Trivy, and Dependency-Check, and automating CI/CD with Jenkins. Additionally, we integrate Prometheus and Grafana for monitoring, and deploy to Kubernetes via ArgoCD.

---

## Phase 1: Initial Setup and Deployment

### ✅ Launch EC2 (Ubuntu 22.04)

1. Launch an EC2 instance with Ubuntu 22.04.
2. Connect via SSH:
   ```bash
   ssh -i <key.pem> ubuntu@<ec2-public-ip>
````

### ✅ Clone Application Repository

Update packages and clone the repo:

```bash
sudo apt update -y
sudo apt install git -y
git clone https://github.com/N4si/DevSecOps-Project.git
cd DevSecOps-Project
```

### ✅ Install Docker & Run the App

```bash
sudo apt install docker.io -y
sudo usermod -aG docker $USER
newgrp docker
sudo chmod 777 /var/run/docker.sock
```

Build and run Docker container:

```bash
docker build -t netflix .
docker run -d --name netflix -p 8081:80 netflix:latest
```

App will show an error without TMDB API key.

### ✅ Get TMDB API Key

* Visit [TMDB](https://www.themoviedb.org/) → Login → Settings → API → Create API Key.

Rebuild Docker image with your key:

```bash
docker build --build-arg TMDB_V3_API_KEY=<your-api-key> -t netflix .
```

---

## Phase 2: Security Integration

### ✅ SonarQube Installation

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

Access: `http://<public-ip>:9000` (default login: `admin` / `admin`)

### ✅ Trivy Installation

```bash
sudo apt install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt update
sudo apt install trivy -y
```

Scan Docker images:

```bash
trivy image netflix
```

---

## Phase 3: CI/CD Pipeline with Jenkins

### ✅ Jenkins Setup

Install Jenkins:

```bash
sudo apt update
sudo apt install fontconfig openjdk-17-jre -y
java -version
```

Add Jenkins repo and install:

```bash
wget -O /usr/share/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt update
sudo apt install jenkins -y
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

Access: `http://<public-ip>:8080`

### ✅ Jenkins Plugins

Install via `Manage Jenkins → Plugins`:

* Eclipse Temurin Installer
* SonarQube Scanner
* NodeJs Plugin
* OWASP Dependency-Check
* Docker Plugins (Commons, Pipeline, API, build-step)
* Email Extension Plugin

### ✅ Jenkins Global Tool Configuration

Configure:

* JDK 17
* Node.js 16
* SonarQube Scanner
* Dependency-Check Tool (`DP-Check`)

### ✅ Configure Credentials

* Add **Sonar Token** as Secret Text
* Add **DockerHub Credentials** (`username/password`) as Secret Text

---

## Jenkins Pipeline (Example)

```groovy
pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('Clean Workspace') {
            steps { cleanWs() }
        }
        stage('Checkout') {
            steps { git branch: 'main', url: 'https://github.com/N4si/DevSecOps-Project.git' }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix -Dsonar.projectKey=Netflix'
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps { sh 'npm install' }
        }
        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('Trivy FS Scan') {
            steps { sh 'trivy fs . > trivyfs.txt' }
        }
        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh 'docker build --build-arg TMDB_V3_API_KEY=<yourapikey> -t netflix .'
                        sh 'docker tag netflix your-dockerhub-username/netflix:latest'
                        sh 'docker push your-dockerhub-username/netflix:latest'
                    }
                }
            }
        }
        stage('Trivy Image Scan') {
            steps { sh 'trivy image your-dockerhub-username/netflix:latest > trivyimage.txt' }
        }
        stage('Deploy to Container') {
            steps { sh 'docker run -d --name netflix -p 8081:80 your-dockerhub-username/netflix:latest' }
        }
    }
}
```

---

## Phase 4: Monitoring with Prometheus and Grafana

### ✅ Prometheus Installation

Follow these steps:

```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
# Move binaries and configs as needed, create user, set up systemd service
```

Systemd service file: `/etc/systemd/system/prometheus.service`

### ✅ Node Exporter Setup

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
# Move binary, set up systemd
```

### ✅ Grafana Installation

```bash
sudo apt-get install -y apt-transport-https software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt-get update
sudo apt-get install grafana
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

Access Grafana: `http://<server-ip>:3000` (default user: `admin`)

### ✅ Add Prometheus Data Source and Dashboards

* Data Source URL: `http://localhost:9090`
* Import dashboards (e.g., ID `1860`)

---

## Phase 5: Notifications

* Configure Jenkins Email plugin or Slack integrations.
* Setup triggers for failure/success notifications.

---

## Phase 6: Kubernetes & ArgoCD Deployment

### ✅ Create Kubernetes Cluster with Node Groups

* Use EKS or Minikube for local testing.

### ✅ Install Node Exporter via Helm

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
kubectl create namespace prometheus-node-exporter
helm install prometheus-node-exporter prometheus-community/prometheus-node-exporter --namespace promet
```
