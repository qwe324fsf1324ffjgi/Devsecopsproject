Below is a polished **README.md** 

---

````markdown
 🎬 Netflix Clone – Cloud CI/CD & DevSecOps Pipeline

 🏗 Overview  
This project delivers a security-first CI/CD pipeline for deploying a Netflix-clone app on AWS. It spans:

- Compute: Two Ubuntu EC2s – one for CI/CD and security scanning, one for monitoring  
- CI/CD: Jenkins automates build → test → scan → deploy  
- Security: SonarQube, OWASP Dependency‑Check, Trivy  
- CI/CD Workflow: Docker builds → push → ArgoCD sync → Amazon EKS  
- Monitoring: Prometheus + Grafana  
- Notifications: Jenkins email alerts

---

 🛠 Architecture  

```
EC2 1 (Ubuntu – CI/CD & Security)
 ├─ Jenkins → Pipeline
 ├─ SonarQube → Static Code Analysis
 ├─ OWASP Dependency‑Check → SCA
 ├─ Trivy → Container Scanning
 └─ Docker → Image Build → DockerHub

     ↓ DockerHub push
 +------------------------------+
 | ArgoCD (in EKS) auto-syncs  |
 | → Deploys app to Kubernetes |
 +------------------------------+

EC2 2 (Ubuntu – Monitoring)
 ├─ Prometheus → Scrapes Jenkins + App + Node Exporter
 └─ Grafana → Visualizes metrics
````

---

🚀 Phase 1: Initial Deployment

 1. Launch EC2 (Ubuntu 22.04)

```bash
 SSH into your EC2 instance…
ssh ubuntu@<YOUR_EC2_IP>
```

 2. Clone & Build Docker App

```bash
sudo apt update
git clone https://https://github.com/qwe324fsf1324ffjgi/Devsecopsproject.git
cd DevSecOps-Project
docker build -t netflix .
docker run -d --name netflix -p 8081:80 netflix
 —> It will error due to missing TMDB API key
```

 3. Add TMDB API Key

```bash
 Obtain from your TMDB account → Settings → API
docker build --build-arg TMDB_V3_API_KEY=<YOUR_KEY> -t netflix .
docker run -d -p 8081:80 netflix
```

---

## 🔐 Phase 2: Security Scanning

 SonarQube

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
 Access: http://<EC2_IP>:9000 (admin/admin)
```

 Trivy

```bash
sudo apt install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main \
  | sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt update && sudo apt install trivy -y
trivy image netflix:latest
```

---

## ⚙️ Phase 3: CI/CD with Jenkins

 Install Jenkins & Plugins

```bash
sudo apt install openjdk-17-jre -y
wget -O jenkins.key https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key  
echo deb ... jenkins.list
sudo apt update && sudo apt install jenkins -y
sudo systemctl enable --now jenkins
```

* Plugins: **SonarQube Scanner**, **NodeJS**, **Dependency‑Check**, **Docker**, **Docker Pipeline**, **Email Extension**

 Configure Tools & Credentials

* **Global Tools**: Set JDK17, NodeJS16, SonarQube Scanner, Dependency‑Check
* **Credentials**: Add DockerHub secret + SonarQube token via Jenkins → Credentials

### Sample Jenkins Pipeline (`Jenkinsfile`)

```groovy
pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/qwe324fsf1324ffjgi/Devsecopsproject.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            } 
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                       sh "docker build --build-arg TMDB_V3_API_KEY=<yourtmdbapikey> -t netflix ."
                       sh "docker tag netflix zerah550/netflix:latest "
                       sh "docker push zerah550/netflix:latest "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image zerah550/netflix:latest > trivyimage.txt" 
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -d -p 8081:80 zerah550/netflix:latest'
            }
        }
        stage('Deploy to kubernets'){
            steps{
                script{
                    dir('Kubernetes') {
                        withKubeConfig(caCertificate: '', clusterName: '', contextName: '', credentialsId: 'k8s', namespace: '', restrictKubeConfigAccess: false, serverUrl: '') {
                                sh 'kubectl apply -f deployment.yml'
                                sh 'kubectl apply -f service.yml'
                        }   
                    }
                }
            }
        }

    }
    post {
     always {
        emailext attachLog: true,
            subject: "'${currentBuild.result}'",
            body: "Project: ${env.JOB_NAME}<br/>" +
                "Build Number: ${env.BUILD_NUMBER}<br/>" +
                "URL: ${env.BUILD_URL}<br/>",
            to: 'zerahabba1.com',                                #change mail here
            attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
        }
    }
}
```

---

## 📈 Phase 4: Monitoring with Prometheus & Grafana

 Prometheus & Node Exporter

```bash
# Prometheus install...
# Node Exporter install...

# Example scrape config in /etc/prometheus/prometheus.yml
scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['<EC2_IP>:8080']
```

### Grafana Setup

```bash
# Install Grafana, enable service
# Add Prometheus as data source (http://localhost:9090)
# Import Dashboard ID 1860 to visualize metrics
```

---

## ⚡ Phases 5–6: Notifications & Kubernetes

* **Phase 5**: Jenkins sends email alerts on build/success/fail (configured via `emailext`)
* **Phase 6**:

  * Create Amazon EKS cluster (node groups)
  * Install Prometheus + Node Exporter via Helm:

    ```bash
    helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
    helm install pe-node-exporter prometheus-community/prometheus-node-exporter \
      --namespace monitoring --create-namespace
    ```
  * Deploy Netflix clone via ArgoCD:

    1. Install ArgoCD in EKS
    2. Point to your GitHub repo + Kubernetes manifests
    3. Configure `syncPolicy` for auto-deploy (prune + self-heal)
    4. Expose the service (ex: NodePort 30007) -> access at `http://<NODE_IP>:30007`

---

## 🧹 Phase 7: Cleanup

```bash
# Terminate EC2 instances
# Optional: delete EKS cluster
```
### 👨‍💻 Author

**Zerah Abba** — Cloud & DevSecOps Engineer
📧 zerahabba1.com


