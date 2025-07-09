
# 🚀 Jenkins + SonarQube + Prometheus + Grafana Full DevOps Setup (AWS EC2)

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
Login: `admin/admin`

Access SonarQube: `http://<jenkins-sonar-ip>:9000`

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

URL: `http://<prometheus-grafan-ip>:9090`

### ✅ Install Node Exporter
Add job_name in Prometheus config:
```yaml
- job_name: "node_export"
  static_configs:
    - targets: ["localhost:9100"]
```
Check targets: `http://<prometheus-ip>:9090/targets`

### ✅ Install Grafana
```bash
sudo apt-get install -y apt-transport-https software-properties-common
# Add repo and install Grafana
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```
Grafana URL: `http://<prometheus-ip>:3000`  
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
- Docker Pipeline & API

### 🔹 Prometheus Integration
Set system path `/Prometheus` under Jenkins → System → Prometheus section.

### 🔹 Email Configuration
Set up SMTP with:
- Host: `smtp.gmail.com`
- Port: `465`
- SSL enabled
- App password stored in credentials

### 🔹 Tools Configuration
- **JDK:** jdk17 from adoptium.net
- **NodeJS:** node16 from nodejs.org

---

## 🔐 SonarQube Integration

### Sonar Server Setup in Jenkins:
1. Create token in SonarQube.
2. Add token as secret in Jenkins credentials.
3. Configure Jenkins SonarQube settings with:
   - Name: `sonar-server`
   - URL: `http://<jenkins-ip>:9000`
   - Token: `Sonar-token`

### Sonar Scanner:
- Install in Jenkins Tools.
- Name: `sonar-scanner`

### Webhook in SonarQube:
- URL: `http://<jenkins-ip>:9000/sonarqube-webhook/`

---

## 🔍 OWASP Dependency Check
Jenkins → Tools → Dependency-Check → Install version 6.5.1  
Name: `DP-Check`

---

## 🐳 Docker Build & Push (Jenkins)
- Add Docker tool in Jenkins → Tools.
- Add DockerHub credentials in Jenkins → Credentials.

---

## 📜 Pipeline Script & Build
- Add declarative pipeline in Jenkins pipeline job.
- Example: CI/CD for TMDB app including Trivy scan, Sonar analysis, Docker build & push.

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

> 📘 Complete DevOps Monitoring and CI/CD Workflow on AWS EC2 with open-source tools.
