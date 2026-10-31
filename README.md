# End-to-End Jenkins CI/CD Pipeline with SonarQube, Docker, and AWS EC2

This repository demonstrates the implementation of an Automated Continuous Integration and Continuous Deployment (CI/CD) pipeline using **Jenkins**, **SonarQube**, **Docker**, and **GitHub Webhooks**, deployed on 3 **AWS EC2** instances.

## 🏗️ Architecture Overview

1. **Code Push**: Developer pushes code changes to the GitHub repository.

2. **Webhook Trigger**: A GitHub Webhook triggers the Jenkins build job automatically.

3. **Static Code Analysis**: Jenkins sends the code to **SonarQube** to check for code quality, vulnerabilities, and code smells.

4. **Build & Package**: Jenkins builds the application artifacts.

5. **Containerization**: Jenkins builds a Docker image of the application and runs it inside a Docker container on the AWS EC2 instance.

6. **Deployment**: The application is served publicly via designated EC2 ports.

## 🚀 Tech Stack & Tools

* **Cloud Provider**: AWS (EC2 Instances)

* **CI/CD Server**: Jenkins

* **Code Quality & Analysis**: SonarQube

* **Containerization**: Docker & Docker Hub

* **Version Control**: Git & GitHub

* **Automation Trigger**: GitHub Webhooks

## 📂 Prerequisites & AWS Setup

### 1. AWS EC2 Setup

* Launch an AWS EC2 instance (Ubuntu 22.04 LTS or 24.04 LTS recommended, `t2.medium` or higher for running Jenkins and SonarQube smoothly).

* Configure **Security Group** Inbound Rules:

  * `SSH` (Port 22): For remote terminal access.

  * `HTTP` (Port 80): General web traffic.

  * `Jenkins` (Port 8080): For accessing the Jenkins dashboard.

  * `SonarQube` (Port 9000): For accessing SonarQube dashboard.

  * `App Port` (e.g., Port 8000 / 8081): For accessing the running application container.

## ⚙️ Installation & Configuration Steps

### 1. Install Java & Jenkins

Jenkins requires Java to run (Java 17/21 for modern Jenkins versions):

```
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install OpenJDK 21
sudo apt install openjdk-21-jdk -y

# Add Jenkins repository and key
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install Jenkins
sudo apt update
sudo apt install jenkins -y

# Start and enable Jenkins
sudo systemctl enable --now jenkins

```

### 2. Install Docker & Grant Permissions

```
# Install Docker
sudo apt install docker.io -y

# Enable and start Docker service
sudo systemctl enable --now docker

# Add Jenkins user to the Docker group (crucial for Jenkins to run docker commands)
sudo usermod -aG docker jenkins
sudo usermod -aG docker $USER

# Reboot or restart Jenkins to apply group permissions
sudo systemctl restart jenkins

```

### 3. Setup SonarQube via Docker

Run SonarQube easily using a Docker container:

```
docker run -d --name sonarqube -p 9000:9000 sonarqube:lts-community

```

* Access SonarQube at `http://<YOUR_EC2_PUBLIC_IP>:9000` (Default credentials: `admin` / `admin`).

* Generate a **SonarQube Token** (`Account > Security > Generate Token`).

## 🔧 Jenkins Configuration

1. **Install Plugins**:

   * *SonarQube Scanner*

   * *Docker Pipeline*

   * *GitHub Integration*

2. **Add Credentials in Jenkins**:

   * Go to `Manage Jenkins > Credentials > System > Global credentials`.

   * Add **SonarQube Secret Text Token**.

   * Add **Docker Hub / GitHub Credentials** (Username & Password/Secret).

3. **Configure System Tools**:

   * Configure `SonarQube Server` URL (`http://<EC2-IP>:9000`) and attach the token.

   * Configure `SonarQube Scanner` under **Global Tool Configuration**.

## 🔗 GitHub Webhook Setup

1. Go to your GitHub Repository > **Settings** > **Webhooks** > **Add Webhook**.

2. Set **Payload URL**: `http://<YOUR_EC2_PUBLIC_IP>:8080/github-webhook/`

3. Set **Content type**: `application/json`

4. Select event: *Just the push event*.

## 📜 Declarative Jenkinsfile Example

Below is a sample `Jenkinsfile` structure used for this pipeline:

```
pipeline {
    agent any

    environment {
        SCANNER_HOME = tool 'SonarScanner'
        DOCKER_HUB_USER = 'your-dockerhub-username'
        APP_NAME = 'my-app'
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/your-username/your-repo.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube-Server') {
                    sh "${SCANNER_HOME}/bin/sonar-scanner -Dsonar.projectName=${APP_NAME} -Dsonar.projectKey=${APP_NAME}"
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_HUB_USER}/${APP_NAME}:latest ."
            }
        }

        stage('Deploy Container') {
            steps {
                sh "docker stop ${APP_NAME} || true"
                sh "docker rm ${APP_NAME} || true"
                sh "docker run -d --name ${APP_NAME} -p 8000:8000 ${DOCKER_HUB_USER}/${APP_NAME}:latest"
            }
        }
    }
}

```

## 🛠️ Troubleshooting Notes

* **Permission Denied on `/var/run/docker.sock`**: Ensure `jenkins` user belongs to the `docker` group (`sudo usermod -aG docker jenkins`) and restart Jenkins (`sudo systemctl restart jenkins`).

* **Low Memory / Disk Space Issue**: If running low on `/tmp` space, override Jenkins JVM options with `-Djava.io.tmpdir=/var/lib/jenkins/tmp` or upgrade EC2 instance specs (`t2.medium` recommended).
