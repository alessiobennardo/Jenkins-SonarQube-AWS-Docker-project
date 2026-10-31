# End-to-End Jenkins CI/CD Pipeline with SonarQube, Docker, and AWS EC2

This repository demonstrates the implementation of an Automated Continuous Integration and Continuous Deployment (CI/CD) pipeline using **Jenkins**, **SonarQube**, **Docker**, and **GitHub Webhooks**, deployed across 3 **AWS EC2** instances.

## 🏗️ Architecture Overview

1. **Code Push**: Developer pushes code changes to the GitHub repository.

2. **Webhook Trigger**: A GitHub Webhook triggers the Jenkins build job automatically.

3. **Static Code Analysis**: Jenkins sends the code to **SonarQube** to check for code quality, vulnerabilities, and code smells.

4. **Artifact Transfer**: Jenkins copies the project files to the deployment server (Docker host) using SSH/SCP.

5. **Containerization & Deployment**: Docker builds an image based on Nginx and runs the application inside a container on port 8085.

```
+------------------+       +-------------------+       +--------------------+
|  GitHub Repo     | ----> |  Jenkins Server   | ----> |  SonarQube Server  |
+------------------+       +-------------------+       +--------------------+
                                     |
                                     v (SCP / SSH)
                           +-------------------+
                           |   Docker Host     |
                           |  (Nginx App Container on 8085)
                           +-------------------+

```

## 🚀 Tech Stack & Tools

* **Cloud Provider**: AWS (3 EC2 Instances)

* **CI/CD Server**: Jenkins

* **Code Quality & Analysis**: SonarQube

* **Containerization**: Docker & Nginx

* **Version Control**: Git & GitHub

* **Automation Trigger**: GitHub Webhooks

## 📂 Prerequisites & AWS Setup

### 1. AWS EC2 Instances Setup

Launch 3 separate AWS EC2 instances (Ubuntu 22.04 LTS or 24.04 LTS recommended, `t2.medium` for Jenkins & SonarQube):

1. **Jenkins Instance**: Runs Jenkins CI server.

2. **SonarQube Instance**: Runs SonarQube quality analysis engine.

3. **Docker Host Instance**: Deploys and serves the containerized application.

### 2. Security Group Rules

| Instance | Port / Protocol | Purpose | 
| ----- | ----- | ----- | 
| **All** | `SSH (22)` | Remote terminal access | 
| **Jenkins** | `Custom TCP (8080)` | Jenkins Web Dashboard | 
| **SonarQube** | `Custom TCP (9000)` | SonarQube Web Dashboard | 
| **Docker Host** | `Custom TCP (8085)` | Public Web Application Port | 

## ⚙️ Installation Steps

### 1. Install Java & Jenkins (Jenkins Server)

```
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install OpenJDK 21
sudo apt install openjdk-21-jdk -y

# Add Jenkins repository key and source list
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install and start Jenkins
sudo apt update
sudo apt install jenkins -y
sudo systemctl enable --now jenkins

```

### 2. Install Docker (Docker Host Instance)

```
# Install Docker
sudo apt update
sudo apt install docker.io -y

# Enable and start Docker service
sudo systemctl enable --now docker

# Grant permissions to current user
sudo usermod -aG docker $USER

```

### 3. Setup SonarQube via Docker (SonarQube Server)

```
docker run -d --name sonarqube -p 9000:9000 sonarqube:lts-community

```

* Access SonarQube at `http://<SONARQUBE_EC2_PUBLIC_IP>:9000` (Default login: `admin` / `admin`).

* Generate a Security Token under `Account > Security > Generate Tokens`.

## 🐳 Dockerfile 

Here is the `Dockerfile` used in the project root to bundle the static application into an **Nginx** web server container:

```
# Use official lightweight Nginx image based on Alpine Linux
FROM nginx:alpine

# Copy web application static files to Nginx default HTML directory
COPY . /usr/share/nginx/html

# Expose HTTP port 80 inside the container
EXPOSE 80

# Start Nginx web server in foreground mode
CMD ["nginx", "-g", "daemon off;"]

```

## 🔑 SSH Verification & Fixing `Permission Denied` Errors

Before configuring the Jenkins job, verify that Jenkins can connect to the Docker Host via SSH/SCP.

### 1. Test SSH Access from Jenkins Server

Log into your Jenkins EC2 instance via SSH and attempt to connect to the Docker Host:

```
# Test connection using private key
ssh -i /path/to/your-key.pem ubuntu@<DOCKER_HOST_PRIVATE_IP>

```

If the connection succeeds, you are ready to configure Jenkins. If you get **`Permission denied (publickey)`**, follow the troubleshooting steps below.

### 2. Fixing `Permission Denied (publickey)` on Ubuntu EC2

This issue occurs if key/password authentication parameters are misconfigured or overridden by cloud-init on Ubuntu.

#### Step A: Modify Main SSH Configuration (`sshd_config`)

On the **Docker Host Instance**, open `/etc/ssh/sshd_config`:

```
sudo nano /etc/ssh/sshd_config

```

Ensure the following parameters are set:

```
PubkeyAuthentication yes
PasswordAuthentication yes

```

#### Step B: Check and Modify Cloud-Init Overrides

Ubuntu instances on AWS often use `/etc/ssh/sshd_config.d/50-cloud-init.conf` which overrides `/etc/ssh/sshd_config`.

Open the file on the **Docker Host**:

```
sudo nano /etc/ssh/sshd_config.d/50-cloud-init.conf

```

Ensure `PasswordAuthentication` or `PubkeyAuthentication` is not disabled here:

```
PasswordAuthentication yes
PubkeyAuthentication yes

```

#### Step C: Set User Password & Restart SSH Service

If connecting via password, set a password for the `ubuntu` user:

```
sudo passwd ubuntu

```

Restart the SSH daemon to apply all changes:

```
sudo systemctl restart ssh

```

#### Step D: Verify SSH Key Permissions

Ensure correct permissions on the SSH directory and authorized keys on the Docker Host:

```
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

```

On your local/Jenkins machine, secure the `.pem` key:

```
chmod 400 your-key.pem

```

## 🔧 Jenkins Job Configuration

### 1. Source Code Management (SCM)

* Select **Git**.

* Enter your **Repository URL** (e.g., `https://github.com/your-user/your-repo.git`).

* Specify Branch: `*/main`.

### 2. Build Triggers

* Check **GitHub hook trigger for GITScm polling**.

### 3. Build Steps: SonarQube Code Analysis

* Add Build Step: **Execute SonarQube Scanner**.

* Configure Analysis properties:

  ```
  sonar.projectKey=MyWebProject
  sonar.projectName=MyWebProject
  sonar.sources=.
  
  ```

### 4. Build Steps: Transfer Files to Docker Host

* Add Build Step: **Execute shell** (copy files via SCP to the Docker server):

  ```
  scp -i /path/to/key.pem -r . ubuntu@<DOCKER_HOST_PRIVATE_IP>:/home/ubuntu/app/
  
  ```

### 5. Build Steps: Remote Execution & Docker Deployment

* Add Build Step: **Execute shell via SSH** on Docker Host:

  ```
  cd /home/ubuntu/app/
  
  # Build new Docker image from Dockerfile
  docker build -t nginx-web-app .
  
  # Run container mapping port 8085 to container port 80
  docker run -d -p 8085:80 --name web-app nginx-web-app
  
  ```

## 🔗 GitHub Webhook Setup

1. Go to **GitHub Repo > Settings > Webhooks > Add webhook**.

2. **Payload URL**: `http://<JENKINS_PUBLIC_IP>:8080/github-webhook/`

3. **Content type**: `application/json`

4. Select event: **Just the push event**.

## 🛠️ Troubleshooting Summary

* **`Permission denied (publickey)`**: Refer to the dedicated [SSH Verification & Fixing Permission Denied Errors](#-ssh-verification--fixing-permission-denied-errors) section above to update `/etc/ssh/sshd_config` and `/etc/ssh/sshd_config.d/50-cloud-init.conf`.

* **Low Memory / Disk Space on `/tmp`**:
  Override Jenkins JVM options in systemd configuration (`sudo systemctl edit jenkins`):

  ```
  [Service]
  Environment="JAVA_OPTS=-Djava.io.tmpdir=/var/lib/jenkins/tmp"
  
  ```

  Then reload and restart: `sudo systemctl daemon-reload && sudo systemctl restart jenkins`.