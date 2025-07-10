
# 🚀 Jenkins + SonarQube + Prometheus + Grafana Full DevOps Setup (AWS EC2)

![Screenshot](flow.png)

This guide covers setting up a full CI/CD and monitoring infrastructure using Jenkins, SonarQube, Prometheus, and Grafana across two AWS EC2 instances running Ubuntu 22.04.

---

## 🔹 Step 1: Create Two AWS EC2 Instances

| Instance Name        | OS          | Type     | Storage |
|----------------------|-------------|----------|---------|
| jenkins-sonar        | Ubuntu 22.04| t2.large | 40 GB   |
| prometheus-grafana   | Ubuntu 22.04| t2.large | 20 GB   |

---

## 🛠️ Jenkins-Sonar Instance Setup

### ✅ Install Jenkins
```bash
nano jenkins.sh
```
```script
#!/bin/bash
 
set -e
 
echo "Updating system packages..."
sudo apt update && sudo apt upgrade -y
 
echo "Installing Java (OpenJDK 17)..."
sudo apt install openjdk-17-jdk -y
 
echo "Adding Jenkins GPG key..."
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
 
echo "Adding Jenkins repository..."
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
 
echo "Updating package list with Jenkins repo..."
sudo apt update
 
echo "Installing Jenkins..."
sudo apt install jenkins -y
 
echo "Starting and enabling Jenkins service..."
sudo systemctl start jenkins
sudo systemctl enable jenkins
 
echo "Allowing firewall on port 8080 (if UFW is active)..."
sudo ufw allow 8080 || true
sudo ufw reload || true
 
echo "Jenkins installation completed!"
echo
echo "Access Jenkins via: http://<your-server-ip>:8080"
echo "Initial admin password:"
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```
```bash
chmod +x jenkins.sh
```
```bash
./jenkins.sh
```
Access Jenkins: `http://<jenkins-ip>:8080`  


### ✅ Install Docker
```bash
sudo apt update
sudo apt install docker.io -y
sudo usermod -aG docker $USER
newgrp docker
sudo chmod 777 /var/run/docker.sock
```

### ✅ Install SonarQube (Docker)
```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```
Access SonarQube: `http://<jenkins-sonar-ip>:9000`

Login: `admin/admin`

### ✅ Install Trivy (Security Scanner)
```bash
nano trivy.sh
```
```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y
```
```bash
chmod +x trivy.sh
```
```bash
./trivy.sh
```

---

## 🌐 TMDB API Key Setup
- Register at https://www.themoviedb.org/
- Navigate: Profile → Settings → API → Developer → Create API Key

---

## 📈 Prometheus-Grafana Instance Setup

### ✅ Install Prometheus
Follow detailed steps to download binary, create service, configure Prometheus.yml, and enable the service.
```bash
sudo useradd \
    --system \
    --no-create-home \
    --shell /bin/false prometheus
```
```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
rm -rf prometheus-2.47.1.linux-amd64.tar.gz
sudo mkdir -p /data /etc/prometheus
mv prometheus-2.47.1.linux-amd64/ prometheus
cd prometheus
```
```bash
sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/
prometheus --version
```
```bash
sudo nano /etc/systemd/system/prometheus.service
```
```service
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
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
journalctl -u prometheus -f --no-pager
```

URL: `http://<prometheus-grafan-ip>:9090`

### ✅ Install Node Exporter
Add job_name in Prometheus config:
```bash
sudo useradd \
    --system \
    --no-create-home \
    --shell /bin/false node_exporter
```
```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
```
```bash
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
rm -rf node_exporter-1.6.1.linux-amd64.tar.gz
mv node_exporter-1.6.1.linux-amd64/ node_exporter
mv node_exporter /usr/local/bin/
node_exporter --version
```
```bash
sudo nano /etc/systemd/system/node_exporter.service
```
```service
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
ExecStart=/usr/local/bin/node_exporter \
    --collector.logind
[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
journalctl -u node_exporter -f --no-pager
```
```bash
sudo nano /etc/prometheus/prometheus.yml
```
```yaml
- job_name: "node_export"
  static_configs:
    - targets: ["localhost:9100"]
```

- job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['<jenkins-ip>:8080']

```bash
promtool check config /etc/prometheus/prometheus.yml
curl -X POST http://localhost:9090/-/reload
```
Check targets: `http://<prometheus-grafana-ip>:9090/targets`

### ✅ Install Grafana
```bash
sudo apt-get install -y apt-transport-https software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
```
```bash
sudo apt update -y
sudo apt-get -y install grafana
```
# Add repo and install Grafana
```bash
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```
Grafana URL: `http://<prometheus-grafana -ip>:3000`  
Login: `admin` / `admin`  

Add Prometheus as a data source.

---

## 🔧 Jenkins Configuration

### 🔹 Install Plugins
- Pipeline Stage View
- Prometheus Metrics
- Email Extension
- Eclipse Temurin Installer
- SonarQube Scanner
- NodeJS Plugin
- OWASP Dependency-Check
- Docker
- Docker Pipeline
- Docker API
- Docker-Build-Step

### 🔹 Prometheus Integration
- Prometheus

- Once that is done you will Prometheus is set to /Prometheus path in system configurations

- go to dashboard > manage Jenkins > system 

- Once that is done you will Prometheus is set to /Prometheus path in system configurations

- Apply and Save

### 🔹 Email Configuration
- go to dashboard > manage jenkins > system

- E-mail Notification section configure

- SMTP: smtp.gmail.com

 - click advanced 
 	 - checkout:- use SMTP authentication
		 - username - your gmail_id
		 - password - paste gmail_app_password

 - checkout: use ssl
 - smtp port - 465

- Apply and save.

**Click on Manage Jenkins–> credentials and add your mail username and generated password:**

 - go to dashboard > manage jenkins > credential > system > global credential

  - username - your_gmail_id
  - password - app_password
  - id - mail
  - description - mail

- Create

**Extended E-mail Notification section configure:**
- go to dashboard > manage jenkins > system

- SMTP: smtp.gmail.com
- smtp port - 465
- click advanced
  - credential - add your credential with user

- checkout: use ssl

- scroll down > default content type - HTML (text/html)

- add default trigger - checkout - always , failure any

- Apply and Save

### 🔹 Tools Configuration
- **JDK:**
- add jdk
- Name: jdk17
	  - checkout- install automatically
		 - install from adoptium.net
		 - version - jdk17.0.8.1+1

- **NodeJS:**
- Name: node16
 	 - checkout- install automatically
 		  - install from nodejs.org
 		  - version - Nodejs16.2.0
  
---

## 🔐 SonarQube Integration

### Sonar Server Setup in Jenkins:
1. Click on Administration → Security → Users → Click on Tokens and Update Token → Give it a name → and click on Generate Token
   - Copy Token
2. Goto Jenkins Dashboard → Manage Jenkins → Credentials -> add
   - kind - secret text
   - secret - paste token here
   - id - Sonar-token
   - description - Sonar-token
- Create
3. Now, go to Dashboard → Manage Jenkins → System and Add
   - SonarQube servers
   - SonarQube installation
   - Name - sonar-server

Access SonarQube: <prometheus-grafana-ip>:9000
- server auth token - Sonar-token (as add from credential)
- Apply and Save

### Sonar Scanner:
- dashboard -> manage Jenkins -> tools
- SonarQube sacanner installation
- add SonarQube scanner

	 - SonarQube Scanner
	 - name - sonar-scanner
		- checkout install automatically 
		- version - SonarQube Scanner 5.0.1.3006

- Apply and Save

### Webhook in SonarQube:
- Administration–> Configuration–>Webhooks

- Click on Create & Add details

Access SonarQube: <http://jenkins-public-ip:9000>/sonarqube-webhook/

---

## 🔍 OWASP Dependency Check
- Go to Dashboard → Manage Jenkins → Tools 
- Dependency-check

- Name - DP-Check

- checkout install automaticly
	 - install from GitHub.com
	 - version - Dependency-check 6.5.1

- Apply and Save

---

## 🐳 Docker Build & Push (Jenkins)
- Docker Image Build and Push

- go to Dashboard → Manage Jenkins → Tools →

- Docker Installation

- Name - docker

  - checkout install automaticly
	   - download from docker.com
	   - docker version - latest

**Add DockerHub Username and Password under Global Credentials:**

- go to dashboard > manage Jenkins > credential > sustem > global credential

- kind - username with password
- username - dockerhub username
- password - dockerhub password
- id -docker
- description - docker 

- Create


---

## 📜 Pipeline Script & Build
- Jenkins > Click New Item > Click Pipeline

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
            steps {
                cleanWs()
            }
        }
 
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/Aj7Ay/Netflix-clone.git'
            }
        }
 
        stage('Sonarqube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=Netflix \
                        -Dsonar.projectKey=Netflix'''
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
            steps {
                sh 'npm install'
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
                sh 'trivy fs . > trivyfs.txt'
            }
        }
 
        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker') {
                        sh 'docker build --build-arg TMDB_V3_API_KEY=626fe3f7d2e721c6d51e76f35e8d74da -t netflix .'
                        sh 'docker tag netflix 9016514790/netflix:latest'
                        sh 'docker push 9016514790/netflix:latest'
                    }
                }
            }
        }
 
        stage('TRIVY Image Scan') {
            steps {
                sh 'trivy image 9016514790/netflix:latest > trivyimage.txt'
            }
        }
 
        stage('Deploy to Container') {
            steps {
                script {
                    // Optional: Stop & remove existing container if it exists
                    sh '''
                        docker rm -f netflix || true
                        docker run -d --name netflix -p 8081:80 9016514790/netflix:latest
                    '''
                }
            }
        }
    }
 
    post {
        always {
            emailext(
                attachLog: true,
                subject: "'${currentBuild.result}'",
                body: """
                    Project: ${env.JOB_NAME}<br/>
                    Build Number: ${env.BUILD_NUMBER}<br/>
                    URL: <a href='${env.BUILD_URL}'>${env.BUILD_URL}</a>
                """,
                to: 'meetparmar14790@gmail.com',
                attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
            )
        }
    }
}
```
---

## 📊 Grafana Dashboard Import
- Use IDs: `1860`, `9964` to import dashboards.
- Select Prometheus data source.

---

## ✅ Final Output
- Jenkins running at `http://<jenkins-ip>:8080`
- SonarQube at `http://<jenkins-ip>:9000`
- Prometheus at `http://<prometheus-ip>:9090`
- Grafana at `http://<prometheus-ip>:3000`
- Node Exporter metrics: `http://<prometheus-ip>:9100`

---

---

![Output Screenshot](All-Pipeline-Success.png)

![Output Screenshot](Dependency-Check-Trend.png)

![Output Screenshot](Dependency-Check-Report.png)

![Output Screenshot](Netflix.png)

---
