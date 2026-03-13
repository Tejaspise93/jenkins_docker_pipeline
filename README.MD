# CI/CD Pipeline v2 — Jenkins + Maven + SonarQube + Docker Hub + Docker

## Overview

This project sets up a full CI/CD pipeline using Jenkins to automate building, testing, code quality analysis, Docker image creation, artifact storage on Docker Hub, and automated container deployment of a Java web application.

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
| Docker Hub | - | Container image registry (replaces Nexus) |
| Java | 17 | Runtime for Jenkins and SonarQube |
| Git | - | Source code management |
| GitHub Webhook | - | Auto-trigger Jenkins on code push |

---

## Jenkins Plugins Installed

| Plugin | Purpose |
|---|---|
| **SonarQube Scanner** | Integrates SonarQube analysis into pipeline |
| **Docker Pipeline** | Enables Docker commands inside Jenkins pipeline |
| **SSH Agent** | Allows Jenkins to SSH into Tools Server for deployment |

**How to install any plugin:**
```
Jenkins → Manage Jenkins → Plugins
    ↓
Available plugins tab
    ↓
Search plugin name → Check the box
    ↓
Click: Install
    ↓
Check: Restart Jenkins when installation is complete
    ↓
Verify: Installed plugins tab → search plugin name ✅
```

---

## Server Setup

### Docker Installation (Both Servers)

Run the following on **both** the Jenkins Server and Tools Server:

```bash
# Install Docker
sudo yum install -y docker

# Start Docker service
sudo systemctl start docker

# Enable Docker on boot
sudo systemctl enable docker

# Add user to docker group (ec2-user on Tools Server, jenkins on Jenkins Server)
sudo usermod -aG docker ec2-user       # Tools Server
sudo usermod -aG docker jenkins        # Jenkins Server

# IMPORTANT: Log out and log back in for group change to take effect
exit
# SSH back in, then verify
docker run hello-world
```

> **Note:** The group change only takes effect in a new session. If you run `docker` immediately after `usermod` without logging out, you will get a permission denied error on the Docker socket. This is expected — log out and back in to fix it.

---

### SSH Key Setup (Jenkins Server → Tools Server)

Jenkins needs passwordless SSH access to the Tools Server to deploy containers remotely.

**Step 1 — Generate SSH key pair on Jenkins Server:**
```bash
ssh-keygen -t rsa -b 4096 -C "jenkins"
# Press Enter for all prompts (no passphrase)
```

This creates:
```
~/.ssh/id_rsa        ← private key (stays on Jenkins Server)
~/.ssh/id_rsa.pub    ← public key (goes to Tools Server)
```

**Step 2 — Copy public key to Tools Server:**
```bash
# On Jenkins Server — print the public key
cat ~/.ssh/id_rsa.pub
# Copy the output
```

Then on Tools Server:
```bash
# Switch to ec2-user first if you are logged in as root
su - ec2-user

# Add the key
vi ~/.ssh/authorized_keys
# Append the Jenkins public key on a new line (do not delete existing keys)
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

**Step 3 — Test SSH connection:**
```bash
# On Jenkins Server
ssh ec2-user@<TOOLS-SERVER-PRIVATE-IP>
# Should connect without password
```

**Step 4 — Add private key to Jenkins Credentials:**
```bash
# On Jenkins Server — print the private key
cat ~/.ssh/id_rsa
# Copy everything including BEGIN and END lines
```

```
Jenkins → Manage Jenkins → Credentials → Global → Add Credentials
    Kind:         SSH Username with private key
    ID:           tools-server-ssh-key
    Username:     ec2-user
    Private Key:  Enter directly → paste the private key
    Click: Save
```

---

### Custom Docker Network (Tools Server)

Create a custom bridge network so containers can communicate by name:

```bash
docker network create --driver bridge cicd-net

# Verify
docker network ls
```

> **Why a custom network:** Containers on a custom bridge network can reach each other by container name as a hostname. On the default bridge network you need hardcoded IPs. It also isolates your app containers from unrelated containers on the server.

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
# Default login: admin / admin (change on first login)
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

### SonarQube Configuration

**Step 1 — Login to SonarQube:**
```
Browser → http://<TOOLS-SERVER-IP>:9000
Username: admin
Password: admin  (you will be forced to change this on first login)
```

**Step 2 — Generate Authentication Token:**
```
Top right corner → My Account
    ↓
Security tab
    ↓
Generate Tokens section
    Name: jenkins-token
    Type: User Token
    Click: Generate
    ↓
COPY THE TOKEN IMMEDIATELY — it is shown only once
```

**Step 3 — Add Token to Jenkins:**
```
Jenkins → Manage Jenkins → Credentials → Global → Add Credentials
    Kind:        Secret text
    Secret:      <paste the token>
    ID:          sonar-token
    Description: SonarQube Auth Token
    Click: Save
```

**Step 4 — Configure SonarQube Server in Jenkins:**
```
Jenkins → Manage Jenkins → Configure System
    ↓
Scroll to: SonarQube servers
    ↓
Check: ✅ Enable injection of SonarQube server configuration as build environment variables
    ↓
Click: Add SonarQube
    Name:              SonarQube
    Server URL:        http://<TOOLS-SERVER-IP>:9000
    Server auth token: sonar-token
    Click: Save
```

**Step 5 — Configure SonarQube Webhook:**

This is required for the Quality Gate stage to work. Without the webhook, SonarQube never notifies Jenkins that analysis is complete and the pipeline will time out waiting.

```
SonarQube → Administration → Configuration → Webhooks
    ↓
Create
    Name:   jenkins
    URL:    http://<JENKINS-SERVER-PUBLIC-IP>:8080/sonarqube-webhook/
    ↓
Save
```

> **Note:** Use the PUBLIC IP of your Jenkins server in the webhook URL. SonarQube calls back to Jenkins over the network, so it needs the public IP.

---

### Docker Hub Configuration

**Step 1 — Create Docker Hub account and repository:**
```
hub.docker.com → Sign up
    ↓
Create repository
    Name: myweb
    Visibility: Public
```

**Step 2 — Add Docker Hub credentials to Jenkins:**
```
Jenkins → Manage Jenkins → Credentials → Global → Add Credentials
    Kind:        Username with password
    Username:    <your dockerhub username>
    Password:    <your dockerhub password>
    ID:          dockerhub-creds
    Description: Docker Hub credentials
    Click: Save
```

---

### GitHub Webhook Configuration

**Step 1 — Enable webhook trigger in Jenkins:**
```
Jenkins → myweb-ci → Configure
    ↓
Build Triggers section
    ↓
Check: ✅ GitHub hook trigger for GITScm polling
    ↓
Save
```

**Step 2 — Add webhook in GitHub:**
```
GitHub → Repository → Settings → Webhooks → Add webhook
    ↓
Payload URL:  http://<JENKINS-SERVER-PUBLIC-IP>:8080/github-webhook/
Content type: application/json
Secret:       (leave empty)
Events:       ✅ Just the push event
Active:       ✅ checked
    ↓
Click: Add webhook
```

> **Note:** The trailing slash in `/github-webhook/` is required. Port 8080 must be open in the Jenkins Server security group.

**Step 3 — Verify webhook delivery:**
```
GitHub → Repository → Settings → Webhooks
    ↓
Click your webhook → Recent Deliveries
    ↓
Should show ✅ green tick on the ping event
```

---

## Jenkins Credentials Summary

| Credential ID | Kind | Used For |
|---|---|---|
| `sonar-token` | Secret text | SonarQube authentication |
| `dockerhub-creds` | Username with password | Docker Hub push and pull |
| `tools-server-ssh-key` | SSH Username with private key | SSH into Tools Server for deployment |

---

## Dockerfile

Add this file to the root of your Java project repository:

```dockerfile
FROM tomcat:8-jre8
COPY target/*.war /usr/local/tomcat/webapps/myweb.war
```

The WAR is renamed to `myweb.war` explicitly so Tomcat always deploys it at `/myweb` regardless of the version number in the filename.

---

## Pipeline 1 — myweb-ci (CI Pipeline)

```groovy
pipeline {
    agent any

    environment {
        TOOLS_SERVER_IP = "<TOOLS-SERVER-PRIVATE-IP>"
        DOCKERHUB_REPO  = "<your-dockerhub-username>/myweb"
        SONAR_HOST_URL  = "http://${TOOLS_SERVER_IP}:9000"
        SONAR_TOKEN     = credentials('sonar-token')
        GIT_REPO        = "https://github.com/Tejaspise93/my-app-java.git"
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

## Pipeline 2 — myweb-cd (CD Pipeline)

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

### Pipeline 1 — myweb-ci

| Stage | What It Does |
|---|---|
| **Git Checkout** | Pulls code from GitHub on the specified branch. Reads `pom.xml` to extract project version and stores it as `env.POM_VERSION` for all downstream stages. |
| **Build** | Runs `mvn clean package -DskipTests` to compile and produce a `.war` file in `target/`. |
| **Test** | Runs `mvn test` to execute all JUnit tests. Publishes surefire XML reports to Jenkins. Pipeline stops if any test fails. |
| **SonarQube Analysis** | Scans code quality and sends the report to SonarQube. Must be wrapped in `withSonarQubeEnv` — without this wrapper the Quality Gate stage cannot find the analysis and will fail. |
| **SonarQube Quality Gate** | Waits for SonarQube to evaluate the gate (max 5 minutes). Aborts pipeline if gate fails. Requires the SonarQube webhook to be configured, otherwise Jenkins times out waiting for the callback. |
| **Docker Build & Push** | Builds the Docker image from Dockerfile, tags it with the version from `pom.xml` and as `latest`, pushes both tags to Docker Hub, then removes the local image to keep the Jenkins Server clean. |

### Pipeline 2 — myweb-cd

| Stage | What It Does |
|---|---|
| **Deploy Container** | SSHes into Tools Server. Pulls latest image from Docker Hub. Stops and removes the old container if running. Starts new container on `cicd-net` network at port 5000. |
| **Health Check** | Waits 15 seconds for container to fully start, then runs `docker ps` on the Tools Server to confirm the container is running. |

---

## Pipeline Flow

```
Developer pushes code to GitHub
            ↓
GitHub sends webhook to Jenkins (automatic)
            ↓
    [myweb-ci Stage 1] Git Checkout
        → pulls branch: params.BRANCH
        → extracts version from pom.xml → stored as env.POM_VERSION
            ↓
    [myweb-ci Stage 2] Build
        → mvn clean package
        → generates target/myweb-x.x.x.war
            ↓
    [myweb-ci Stage 3] Test
        → mvn test
        → JUnit tests must pass
        → surefire reports published to Jenkins
            ↓
    [myweb-ci Stage 4] SonarQube Analysis
        → withSonarQubeEnv wrapper registers the analysis
        → scans code quality
        → report sent to Tools Server:9000
            ↓
    [myweb-ci Stage 5] SonarQube Quality Gate
        → waits for SonarQube webhook callback
        → pipeline aborts if gate fails
            ↓
    [myweb-ci Stage 6] Docker Build & Push
        → builds image from Dockerfile
        → tags: <your-dockerhub-username>/myweb:0.0.x and <your-dockerhub-username>/myweb:latest
        → pushes both tags to Docker Hub
        → removes local image
            ↓
    [myweb-ci Post] Triggers myweb-cd on success
            ↓
    [myweb-cd Stage 1] Deploy Container
        → SSH into Tools Server
        → docker pull <your-dockerhub-username>/myweb:latest
        → stop and remove old container
        → docker run on cicd-net port 5000
            ↓
    [myweb-cd Stage 2] Health Check
        → wait 15 seconds
        → docker ps confirms container running
            ↓
    App live at Tools Server:5000/myweb
```

---

## Two-Pipeline Design

Pipeline 1 (`myweb-ci`) owns building, testing, and publishing the image. When it succeeds it triggers Pipeline 2 via the `build job: 'myweb-cd'` call in its `post { success }` block.

Pipeline 2 (`myweb-cd`) owns deployment. It pulls the latest image from Docker Hub and runs it on the Tools Server.

---

## Key Design Decisions

**Docker Hub over Nexus** — Docker Hub is the natural registry for container images. Nexus is removed since WAR files are no longer deployed directly — the WAR is baked into the Docker image at build time.

**Two separate pipelines** — CI and CD have different concerns, different failure modes, and potentially different triggers or permissions. Keeping them separate makes each easier to debug and allows the CD pipeline to be re-run independently without rebuilding the image.

**Pull from Docker Hub to deploy** — The Tools Server pulls the image from Docker Hub rather than receiving it directly from Jenkins. This validates the full chain — the image that was pushed to the registry is exactly what gets deployed, and the image is independently accessible for rollback or audit.

**Custom Docker network** — `cicd-net` isolates app containers from unrelated containers and allows containers to address each other by name if needed in future.

**`withSonarQubeEnv` wrapper** — Required for `waitForQualityGate` to work. The wrapper registers the analysis with the Jenkins pipeline context. Without it, Jenkins has no record of a SonarQube analysis happening and the Quality Gate stage throws an `IllegalStateException`.

**SonarQube webhook** — Required for the Quality Gate to receive a callback from SonarQube instead of polling. Without it the pipeline times out waiting for the gate result.

**Private IP for inter-server communication** — Jenkins and Tools Server communicate over the AWS private IP (`172.31.80.113`). The app is accessed from a browser using the Tools Server public IP on port 5000.

---

## Suggested Improvements

The following improvements were identified but not implemented. They are documented here for future reference.

### Move trigger to Pipeline 2 using upstream trigger

Currently Pipeline 1 triggers Pipeline 2 via `build job: 'myweb-cd'` in its `post { success }` block. This means Pipeline 1 has knowledge of Pipeline 2, which creates a coupling between CI and CD.

The better approach is to remove the trigger from Pipeline 1 entirely and configure Pipeline 2 to watch Pipeline 1 using the `upstream` trigger:

```groovy
// Add this to Pipeline 2 (myweb-cd)
triggers {
    upstream(
        upstreamProjects: 'myweb-ci',
        threshold: hudson.model.Result.SUCCESS
    )
}
```

And remove the `build job: 'myweb-cd'` from Pipeline 1's `post` block.

This way Pipeline 1 has no knowledge of Pipeline 2. The CD pipeline owns the decision of when to deploy. This follows the single responsibility principle and is more scalable — additional pipelines can also watch Pipeline 1 independently without modifying it.

> **Note:** The `upstream` trigger only activates after the pipeline has been run at least once manually. Run `myweb-cd` manually once after adding the trigger so Jenkins reads and registers it.


---

## Verifying Results

**SonarQube Dashboard:**
```
http://<TOOLS-SERVER-IP>:9000 → Projects → myweb
```

**Docker Hub:**
```
hub.docker.com → <your-dockerhub-username>/myweb → Tags
→ should show: latest and version tag (e.g. 0.0.9)
```

**Container running on Tools Server:**
```bash
# SSH into Tools Server
docker ps
# Should show myweb container running
```

**Live Application:**
```
http://<TOOLS-SERVER-PUBLIC-IP>:5000/myweb
```

**Webhook Deliveries:**
```
GitHub → Repo → Settings → Webhooks → Recent Deliveries
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
- **Packaging:** WAR → Docker image
- **Docker Hub Repository:** tejaspise/myweb
- **App Repository:** https://github.com/Tejaspise93/my-app-java