# CI/CD Pipeline — Jenkins + Maven + SonarQube + Docker Hub + Docker

## Overview

This project sets up a full CI/CD pipeline using Jenkins to automate building, testing, code quality analysis, Docker image creation, and container deployment of a Java web application.

The pipeline is split into two parts — a CI pipeline that builds and pushes the Docker image, and a CD pipeline that deploys the container to the Tools Server.

---

## Infrastructure

| Server | Purpose |
|---|---|
| Jenkins Server | Jenkins, Maven, Docker |
| Tools Server | SonarQube (port 9000), Docker, App Container (port 5000) |

---

## Tools Used

| Tool | Version | Purpose |
|---|---|---|
| Jenkins | Latest LTS | CI/CD orchestration |
| Maven | 3.x | Build, test, and package Java project |
| SonarQube | 9.1.0 | Static code quality analysis |
| Docker | Latest | Build and run containers |
| Docker Hub | - | Container image registry |
| Java | 17 | Runtime for Jenkins and SonarQube |
| Git | - | Source code management |
| GitHub Webhook | - | Auto-trigger Jenkins on code push |

---

## Jenkins Plugins Required

| Plugin | Purpose |
|---|---|
| SonarQube Scanner | Integrates SonarQube analysis into pipeline |
| Docker Pipeline | Enables Docker commands inside Jenkins pipeline |
| SSH Agent | Allows Jenkins to SSH into Tools Server for deployment |

**How to install a plugin:**

Jenkins -> Manage Jenkins -> Plugins -> Available plugins -> search plugin name -> check the box -> Install -> check "Restart Jenkins when installation is complete" -> verify under Installed plugins tab

---

## Server Setup

### Docker Installation (Both Servers)

Run the following on both the Jenkins Server and Tools Server:

```bash
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker

# Add the appropriate user to the docker group
sudo usermod -aG docker ec2-user    # Tools Server
sudo usermod -aG docker jenkins     # Jenkins Server

# Log out and back in for the group change to take effect
exit
```

> **Note:** The group change only takes effect in a new session. Run `docker run hello-world` after logging back in to verify.

---

### SSH Key Setup (Jenkins Server -> Tools Server)

Jenkins needs passwordless SSH access to the Tools Server to deploy containers remotely.

**Step 1 — Generate SSH key pair on Jenkins Server:**
```bash
ssh-keygen -t rsa -b 4096 -C "jenkins"
# Press Enter for all prompts (no passphrase)
```

This creates:
- `~/.ssh/id_rsa` — private key (stays on Jenkins Server)
- `~/.ssh/id_rsa.pub` — public key (goes to Tools Server)

**Step 2 — Copy public key to Tools Server:**
```bash
# On Jenkins Server
cat ~/.ssh/id_rsa.pub
# Copy the output
```

```bash
# On Tools Server
su - ec2-user
vi ~/.ssh/authorized_keys
# Paste the Jenkins public key on a new line
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

**Step 3 — Test the connection:**
```bash
# On Jenkins Server
ssh ec2-user@<TOOLS-SERVER-PRIVATE-IP>
# Should connect without a password prompt
```

**Step 4 — Add private key to Jenkins Credentials:**

```bash
# On Jenkins Server
cat ~/.ssh/id_rsa
# Copy everything including the BEGIN and END lines
```

Jenkins -> Manage Jenkins -> Credentials -> Global -> Add Credentials
- Kind: SSH Username with private key
- ID: tools-server-ssh-key
- Username: ec2-user
- Private Key: Enter directly -> paste the private key
- Save

---

### Custom Docker Network (Tools Server)

```bash
docker network create --driver bridge cicd-net
docker network ls
```

Containers on a custom bridge network can reach each other by container name. The default bridge network requires hardcoded IPs.

---

### SonarQube (Tools Server)

```bash
yum install -y java-17
cd /opt/
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.1.0.47736.zip
unzip sonarqube-9.1.0.47736.zip
useradd sonar
chown -R sonar:sonar sonarqube-9.1.0.47736
cd sonarqube-9.1.0.47736/bin/linux-x86-64

# Edit sonar.sh — uncomment RUN_AS_USER and set it to sonar
vi sonar.sh

sh sonar.sh start
sh sonar.sh status
# Access at: http://<TOOLS-SERVER-IP>:9000
# Default login: admin / admin (you will be forced to change this on first login)
```

---

### Maven (Jenkins Server)

```bash
sudo apt update
sudo apt install -y maven
mvn --version
```

---

## Configurations

### SonarQube

**Step 1 — Generate an authentication token:**

SonarQube -> top right corner -> My Account -> Security tab -> Generate Tokens -> set name to `jenkins-token`, type: User Token -> Generate

Copy the token immediately — it is shown only once.

**Step 2 — Add token to Jenkins:**

Jenkins -> Manage Jenkins -> Credentials -> Global -> Add Credentials
- Kind: Secret text
- Secret: paste the token
- ID: sonar-token
- Save

**Step 3 — Configure SonarQube server in Jenkins:**

Jenkins -> Manage Jenkins -> Configure System -> SonarQube servers -> Add SonarQube
- Name: SonarQube
- Server URL: http://<TOOLS-SERVER-IP>:9000
- Server auth token: sonar-token
- Check: Enable injection of SonarQube server configuration as build environment variables
- Save

**Step 4 — Configure SonarQube webhook:**

This is required for the Quality Gate stage to work. Without it, SonarQube never notifies Jenkins that analysis is complete and the pipeline will time out.

SonarQube -> Administration -> Configuration -> Webhooks -> Create
- Name: jenkins
- URL: http://<JENKINS-SERVER-PUBLIC-IP>:8080/sonarqube-webhook/
- Save

> **Note:** Use the public IP of the Jenkins server in the webhook URL. SonarQube calls back to Jenkins over the network.

---

### Docker Hub

**Step 1 — Create a Docker Hub account and repository:**

hub.docker.com -> Sign up -> Create repository -> name it `myweb`, visibility: Public

**Step 2 — Add Docker Hub credentials to Jenkins:**

Jenkins -> Manage Jenkins -> Credentials -> Global -> Add Credentials
- Kind: Username with password
- Username: your Docker Hub username
- Password: your Docker Hub password
- ID: dockerhub-creds
- Save

---

### GitHub Webhook

**Step 1 — Enable webhook trigger in Jenkins:**

Jenkins -> myweb-ci -> Configure -> Build Triggers -> check "GitHub hook trigger for GITScm polling" -> Save

**Step 2 — Add webhook in GitHub:**

GitHub -> Repository -> Settings -> Webhooks -> Add webhook
- Payload URL: http://<JENKINS-SERVER-PUBLIC-IP>:8080/github-webhook/
- Content type: application/json
- Events: Just the push event
- Active: checked
- Add webhook

> **Note:** The trailing slash in `/github-webhook/` is required. Port 8080 must be open in the Jenkins Server security group.

**Step 3 — Verify webhook delivery:**

GitHub -> Repository -> Settings -> Webhooks -> click your webhook -> Recent Deliveries — should show a green tick on the ping event.

---

## Jenkins Credentials Summary

| Credential ID | Kind | Used For |
|---|---|---|
| sonar-token | Secret text | SonarQube authentication |
| dockerhub-creds | Username with password | Docker Hub push and pull |
| tools-server-ssh-key | SSH Username with private key | SSH into Tools Server for deployment |

---

## Dockerfile

Place this in the root of the Java project repository:

```dockerfile
FROM tomcat:8-jre8
COPY target/*.war /usr/local/tomcat/webapps/myweb.war
```

The WAR is renamed to `myweb.war` so Tomcat always deploys it at `/myweb` regardless of the version number in the filename.

---

## Pipeline 1 — myweb-ci

```groovy
pipeline {
    agent any

    environment {
        TOOLS_SERVER_IP = "<TOOLS-SERVER-PRIVATE-IP>"
        DOCKERHUB_REPO  = "<your-dockerhub-username>/myweb"
        SONAR_HOST_URL  = "http://${TOOLS_SERVER_IP}:9000"
        SONAR_TOKEN     = credentials('sonar-token')
        GIT_REPO        = "<your-github-repo-url>"
    }

    parameters {
        string(name: 'BRANCH', defaultValue: 'master', description: 'Branch to build')
    }

    stages {

        stage('Git Checkout') {
            steps {
                git branch: "${params.BRANCH}",
                    url: "${GIT_REPO}"
                script {
                    env.POM_VERSION = sh(
                        script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout",
                        returnStdout: true
                    ).trim()
                    echo "--- Detected Version: ${env.POM_VERSION} ---"
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        mvn sonar:sonar \
                            -Dsonar.projectKey=myweb \
                            -Dsonar.host.url=${SONAR_HOST_URL} \
                            -Dsonar.login=\$SONAR_TOKEN
                    """
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh """
                            docker build -t ${DOCKERHUB_REPO}:${env.POM_VERSION} .
                            docker tag ${DOCKERHUB_REPO}:${env.POM_VERSION} ${DOCKERHUB_REPO}:latest
                            echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
                            docker push ${DOCKERHUB_REPO}:${env.POM_VERSION}
                            docker push ${DOCKERHUB_REPO}:latest
                            docker rmi ${DOCKERHUB_REPO}:${env.POM_VERSION} ${DOCKERHUB_REPO}:latest
                        """
                    }
                }
            }
        }

    }

    post {
        success {
            build job: 'myweb-cd', wait: false
        }
        always {
            cleanWs()
        }
    }
}
```

---

## Pipeline 2 — myweb-cd

```groovy
pipeline {
    agent any

    environment {
        TOOLS_SERVER_IP = "<TOOLS-SERVER-PRIVATE-IP>"
        DOCKERHUB_REPO  = "<your-dockerhub-username>/myweb"
        APP_PORT        = "5000"
        CONTAINER_NAME  = "myweb"
        NETWORK_NAME    = "cicd-net"
    }

    stages {

        stage('Deploy Container') {
            steps {
                sshagent(['tools-server-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ec2-user@${TOOLS_SERVER_IP} '
                            docker pull ${DOCKERHUB_REPO}:latest &&
                            docker stop ${CONTAINER_NAME} || true &&
                            docker rm ${CONTAINER_NAME} || true &&
                            docker run -d \
                                --name ${CONTAINER_NAME} \
                                --network ${NETWORK_NAME} \
                                -p ${APP_PORT}:8080 \
                                ${DOCKERHUB_REPO}:latest
                        '
                    """
                }
            }
        }

        stage('Health Check') {
            steps {
                sshagent(['tools-server-ssh-key']) {
                    script {
                        sleep(time: 15, unit: 'SECONDS')
                        sh """
                            ssh -o StrictHostKeyChecking=no ec2-user@${TOOLS_SERVER_IP} \
                                'docker ps | grep ${CONTAINER_NAME}'
                        """
                    }
                }
            }
        }

    }

    post {
        success {
            echo "--- Deployment successful ---"
            echo "--- App live at: http://${TOOLS_SERVER_IP}:${APP_PORT}/myweb ---"
        }
        failure {
            echo "--- Deployment failed ---"
        }
        always {
            cleanWs()
        }
    }
}
```

---

## Pipeline Stage Breakdown

### myweb-ci

| Stage | What It Does |
|---|---|
| Git Checkout | Pulls code from GitHub on the specified branch. Reads `pom.xml` to extract project version and stores it as `env.POM_VERSION`. |
| Build | Runs `mvn clean package -DskipTests` to compile and produce a `.war` file in `target/`. |
| Test | Runs `mvn test`. Publishes surefire XML reports to Jenkins. Pipeline stops if any test fails. |
| SonarQube Analysis | Scans code quality and sends the report to SonarQube. Must use the `withSonarQubeEnv` wrapper — without it the Quality Gate stage cannot find the analysis. |
| SonarQube Quality Gate | Waits up to 5 minutes for SonarQube to evaluate the gate. Aborts the pipeline if the gate fails. Requires the SonarQube webhook to be configured. |
| Docker Build & Push | Builds the Docker image, tags it with the version from `pom.xml` and as `latest`, pushes both tags to Docker Hub, then removes the local image. |

### myweb-cd

| Stage | What It Does |
|---|---|
| Deploy Container | SSHes into Tools Server. Pulls the latest image from Docker Hub. Stops and removes the old container. Starts the new container on `cicd-net` at port 5000. |
| Health Check | Waits 15 seconds, then runs `docker ps` on the Tools Server to confirm the container is running. |

---

## Pipeline Flow

```
Developer pushes code to GitHub
    |
    v
GitHub sends webhook to Jenkins
    |
    v
[CI] Git Checkout
    pulls branch, extracts version from pom.xml
    |
    v
[CI] Build
    mvn clean package -> target/myweb-x.x.x.war
    |
    v
[CI] Test
    mvn test -> JUnit results published to Jenkins
    |
    v
[CI] SonarQube Analysis
    scans code quality -> report sent to Tools Server:9000
    |
    v
[CI] SonarQube Quality Gate
    waits for webhook callback -> aborts if gate fails
    |
    v
[CI] Docker Build & Push
    builds image -> tags with version and latest -> pushes to Docker Hub
    |
    v
[CI] post success -> triggers myweb-cd
    |
    v
[CD] Deploy Container
    SSH into Tools Server -> pull image -> stop old container -> run new container
    |
    v
[CD] Health Check
    wait 15s -> docker ps confirms container running
    |
    v
App live at Tools Server:5000/myweb
```

---

## Design Decisions

**Docker Hub over Nexus** — Docker Hub is the natural registry for container images. The WAR is baked into the Docker image at build time so a separate artifact store is not needed.

**Two separate pipelines** — CI and CD have different concerns and failure modes. Keeping them separate makes each easier to debug and allows the CD pipeline to be re-run independently without rebuilding the image.

**Pull from Docker Hub to deploy** — The Tools Server pulls the image from Docker Hub rather than receiving it directly from Jenkins. This validates the full chain — the image that was pushed is exactly what gets deployed.

**Custom Docker network** — `cicd-net` isolates app containers from unrelated ones and allows containers to address each other by name if needed.

**SonarQube webhook** — Required for the Quality Gate to receive a callback from SonarQube. Without it the pipeline times out waiting for the gate result.

---

## Suggested Improvement

Currently Pipeline 1 triggers Pipeline 2 via `build job: 'myweb-cd'` in its `post { success }` block, which means CI has knowledge of CD.

A cleaner approach is to remove that trigger from Pipeline 1 and configure Pipeline 2 to watch Pipeline 1 using the `upstream` trigger:

```groovy
// Add to Pipeline 2 (myweb-cd)
triggers {
    upstream(
        upstreamProjects: 'myweb-ci',
        threshold: hudson.model.Result.SUCCESS
    )
}
```

This way Pipeline 1 has no knowledge of Pipeline 2. The CD pipeline owns the decision of when to deploy, and additional pipelines can watch Pipeline 1 independently without modifying it.

> **Note:** Run `myweb-cd` manually once after adding the trigger so Jenkins reads and registers it.

---

## Verifying Results

**SonarQube Dashboard:**
```
http://<TOOLS-SERVER-IP>:9000 -> Projects -> myweb
```

**Docker Hub:**
```
hub.docker.com -> <your-dockerhub-username>/myweb -> Tags
should show: latest and version tag (e.g. 0.0.9)
```

**Container on Tools Server:**
```bash
docker ps
# myweb container should be running
```

**Live Application:**
```
http://<TOOLS-SERVER-PUBLIC-IP>:5000/myweb
```

**Webhook Deliveries:**
```
GitHub -> Repository -> Settings -> Webhooks -> Recent Deliveries
```

---

## AWS Security Group — Required Open Ports

| Server | Port | Purpose |
|---|---|---|
| Jenkins Server | 8080 | Jenkins UI and GitHub webhook receiver |
| Tools Server | 9000 | SonarQube |
| Tools Server | 5000 | Live application |

---

## Project Info

- **Application:** myweb
- **Group ID:** in.javahome
- **Packaging:** WAR -> Docker image
- **Docker Hub Repository:** tejaspise/myweb
- **App Repository:** https://github.com/Tejaspise93/my-app-java