# CI/CD Pipeline — Jenkins + Maven + SonarQube + Docker Hub + EKS

## Overview

This project sets up a full CI/CD pipeline using Jenkins to automate building, testing, code quality analysis, Docker image creation, and deployment of a Java web application to AWS EKS.

The pipeline is split into two parts — a CI pipeline that builds and pushes the Docker image, and a CD pipeline that deploys the container to EKS.

---

## Infrastructure

| Server | Purpose |
|---|---|
| Jenkins Server | Jenkins, Maven, Docker |
| Tools Server | SonarQube (port 9000) |
| AWS EKS | Kubernetes cluster for application deployment |

---

## Tools Used

| Tool | Version | Purpose |
|---|---|---|
| Jenkins | Latest LTS | CI/CD orchestration |
| Maven | 3.x | Build, test, and package Java project |
| SonarQube | 9.1.0 | Static code quality analysis |
| Docker | Latest | Build and run containers |
| Docker Hub | - | Container image registry |
| kubectl | Latest | Kubernetes CLI for EKS deployments |
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
sudo yum update
sudo yum install -y maven
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
- Server URL: http://TOOLS-SERVER-IP:9000
- Server auth token: sonar-token
- Check: Enable injection of SonarQube server configuration as build environment variables
- Save

**Step 4 — Configure SonarQube webhook:**

This is required for the Quality Gate stage to work. Without it, SonarQube never notifies Jenkins that analysis is complete and the pipeline will time out.

SonarQube -> Administration -> Configuration -> Webhooks -> Create
- Name: jenkins
- URL: http://JENKINS-SERVER-PUBLIC-IP:8080/sonarqube-webhook/
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

### EKS Kubeconfig

**Step 1 — Install AWS CLI on Jenkins Server:**

```bash
# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Verify installation
aws --version
```

**Step 2 — Install kubectl on Jenkins Server:**

```bash
# Download the latest stable kubectl binary
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

# Make it executable and move to PATH
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl

# Verify installation
kubectl version --client
```

**Step 3 — Configure AWS CLI credentials for the jenkins user:**

Switch to the jenkins user and configure its AWS profile with an IAM user that has EKS access:

```bash
# Switch to the jenkins user
sudo su - jenkins

# Configure AWS credentials
aws configure
# AWS Access Key ID:     <your-access-key-id>
# AWS Secret Access Key: <your-secret-access-key>
# Default region name:   <your-region>   e.g. us-east-1
# Default output format: json
```

This creates `~/.aws/credentials` and `~/.aws/config` under the jenkins user's home directory. Jenkins pipeline commands running as the jenkins user will automatically use these credentials.

> **Note:** The IAM user must have at minimum the following permissions: `eks:DescribeCluster` and be mapped inside the EKS cluster's `aws-auth` ConfigMap. See Step 4.

> **Note:** IAM user to EKS cluster mapping (`aws-auth` ConfigMap) is handled in Terraform. Ensure the jenkins IAM user is included in the `mapUsers` block in your Terraform EKS configuration before proceeding.

**Step 4 — Generate the kubeconfig file as the jenkins user:**

```bash
# Still as the jenkins user
aws eks update-kubeconfig --region <your-region> --name <your-cluster-name> --kubeconfig ~/kubeconfig

# Verify cluster connectivity
kubectl --kubeconfig=~/kubeconfig get nodes
# Should list the EKS worker nodes as Ready
```

**Step 5 — Add kubeconfig to Jenkins Credentials:**

Jenkins -> Manage Jenkins -> Credentials -> Global -> Add Credentials
- Kind: Secret file
- File: upload the `~/kubeconfig` file from the Jenkins server
- ID: k8s-kubeconfig
- Save

---

### GitHub Webhook

**Step 1 — Enable webhook trigger in Jenkins:**

Jenkins -> myweb-ci -> Configure -> Build Triggers -> check "GitHub hook trigger for GITScm polling" -> Save

**Step 2 — Add webhook in GitHub:**

GitHub -> Repository -> Settings -> Webhooks -> Add webhook
- Payload URL: http://JENKINS-SERVER-PUBLIC-IP:8080/github-webhook/
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
| tools-server-ssh-key | SSH Username with private key | SSH into Tools Server |
| k8s-kubeconfig | Secret file | kubectl access to EKS cluster |


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
| Validate Inputs | Checks that `IMAGE_TAG` parameter is provided. Fails fast if it is empty. |
| Checkout K8s Manifests | Clones the Git repo to retrieve `deployment.yaml` and `service.yaml`. |
| Deploy to EKS | Uses `sed` to replace `IMAGE_TAG_PLACEHOLDER` in `deployment.yaml` with the actual version. Applies both manifests to EKS and waits for the rollout to complete (timeout: 300s). |
| Verify Deployment | Prints pod status, service info, and full deployment details for confirmation. |

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
[CI] post success -> "--- Image upload successful ---"
    (manually trigger myweb-cd with IMAGE_TAG parameter)
    |
    v
[CD] Validate Inputs
    checks IMAGE_TAG is provided
    |
    v
[CD] Checkout K8s Manifests
    clones repo -> retrieves deployment.yaml and service.yaml
    |
    v
[CD] Deploy to EKS
    sed replaces IMAGE_TAG_PLACEHOLDER -> kubectl apply -> rollout status
    |
    v
[CD] Verify Deployment
    pod status, service info, deployment describe
    |
    v
[CD] post success
    fetches LoadBalancer hostname -> echoes full app URL
    App accessible at: http://<elb-hostname>/myweb

    on failure -> kubectl rollout undo -> rollback to previous version
```

---

## Design Decisions

**Docker Hub over Nexus** — Docker Hub is the natural registry for container images. The WAR is baked into the Docker image at build time so a separate artifact store is not needed.

**Two separate pipelines** — CI and CD have different concerns and failure modes. Keeping them separate makes each easier to debug and allows the CD pipeline to be re-run independently without rebuilding the image.

**IMAGE_TAG as a parameter** — The CD pipeline accepts the image version as a parameter rather than always deploying `latest`. This makes it possible to deploy a specific version or roll back to a previous one by re-running the pipeline with a known-good tag.

**IMAGE_TAG_PLACEHOLDER in manifests** — The `deployment.yaml` uses a placeholder that `sed` replaces at deploy time. This keeps the manifest file clean in source control and avoids committing version-specific values.

**Rollback on failure** — The CD pipeline's `post { failure }` block runs `kubectl rollout undo` automatically if the deploy fails, restoring the previous working version.

**URL echoed on success** — The `post { success }` block queries the LoadBalancer hostname from the service and prints the full application URL so it is immediately visible in the build log.

**EKS over Docker container deployment** — EKS provides rolling updates, self-healing, replica management, and a managed LoadBalancer, giving the application high availability and zero-downtime deployments.

---

## Verifying Results

**SonarQube Dashboard:**
```
http://<TOOLS-SERVER-IP>:9000 -> Projects -> myweb
```

**Docker Hub:**
```
hub.docker.com -> tejaspise/myweb -> Tags
should show: latest and version tag (e.g. 8.2.12)
```

**EKS Pods:**
```bash
kubectl get pods -l app=myweb -n default -o wide
# Both pods should show STATUS: Running
```

**Live Application:**
```
http://<elb-hostname>/myweb
# ELB hostname is printed in the CD pipeline post success block
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
| EKS LoadBalancer | 80 | Live application (myweb-service) |

---

## Project Info

- **Application:** myweb
- **Group ID:** in.javahome
- **Packaging:** WAR -> Docker image
- **Docker Hub Repository:** tejaspise/myweb
- **App Repository:** https://github.com/Tejaspise93/myweb.git