# Lab 0: Lab Environment Setup Report

**Course:** IKB42603 Cloud Computing Security Essentials  
**Institution:** Universiti Kuala Lumpur Malaysian Institute of Information Technology (UniKL MIIT)  
**Author / Instructor:** Prof. Dr. Shahrulniza Musa  
**Lab Document:** Environment-Setup.md  
**Reference Guide:** `IKB42603_Lab0_Environment_Setup_Cheatsheet.pdf`  

---

## 1. Objective

The primary objective of **Lab 0: Environment Setup** is to establish and verify a fully functional, self-contained local cloud computing security laboratory environment on the student host machine. This environment serves as the foundation for all subsequent hands-on cloud security labs (Labs 1 through 5).

Specifically, this setup achieves the following:
1. **Local Infrastructure Emulation:** Installs Docker Engine to host containerized workloads and LocalStack (an offline AWS cloud service emulator), allowing AWS CLI operations without requiring real AWS accounts, credit cards, or active cloud subscriptions.
2. **Kubernetes Cluster Provisioning:** Deploys `kind` (Kubernetes-in-Docker) and `kubectl` CLI tools to enable local Kubernetes security testing and cluster orchestration.
3. **Security & Cryptographic Helper Tools:** Configures auxiliary security utilities including OpenSSL (for certificate/key management), `oathtool` (for Multi-Factor Authentication / TOTP generation), and Trivy (for container vulnerability scanning).
4. **Validation & Verification:** Executes end-to-end sanity checks to confirm system permissions, daemon connectivity, tool versioning, and LocalStack/Kubernetes health status prior to commencing security experiments.

---

## 2. Learning Outcomes

Upon completing this lab environment setup, the student has demonstrated and acquired the following skills:

- **Container Runtime Management:** Understood Docker architecture, group permissions (`usermod -aG docker $USER`), and container lifecycle execution (`docker run`, `docker ps`, `docker pull`).
- **AWS Local Emulation & CLI Configuration:** Mastered configuring AWS CLI v2 against local endpoints (`http://localhost:4566`) and using custom environment variables (`$EP`) to simulate AWS Security Token Service (STS) and IAM interactions.
- **Local Kubernetes Cluster Deployment:** Gained practical experience interacting with `kind` clusters and controlling Kubernetes resources via `kubectl`.
- **Security Tooling Readiness:** Verified the availability of key security tools (OpenSSL, `oathtool`, Trivy) necessary for key management, identity verification, and container security scanning.
- **Linux Security Operating Practices:** Applied proper Linux/Bash terminal operations, ensuring compatibility with heredocs, SHA-256 hashing, and shell variable expansion.

---

## 3. Environment

The lab environment was configured and validated on a Linux operating system (Kali Linux). All tools run locally in an isolated environment.

### Host System Details
- **Operating System:** Linux (Kali Linux `6.16.8+kali-amd64` / `x86_64`)
- **Host User & Hostname:** `anonym22@kali`
- **Primary Shell:** Bash (`/bin/bash`)

### Software & Tool Specifications

| Tool Name | Installed Version | Purpose in Security Labs | Lab Scope |
| :--- | :--- | :--- | :--- |
| **Docker Engine** | `28.5.2+dfsg4` (Build `9cc6dea3`) | Container runtime & LocalStack host engine | All Labs (1 – 5) |
| **AWS CLI v2** | `2.36.10` (Python 3.14.6) | Command-line interface to interact with LocalStack AWS endpoints | Labs 1, 3, 5 |
| **kind** | `v0.23.0` | Kubernetes-in-Docker local cluster orchestrator | Labs 1, 2, 4 |
| **kubectl** | Client `v1.36.3` / Kustomize `v5.8.1` | Kubernetes cluster management and control CLI | Labs 1, 2, 4 |
| **LocalStack** | `3.8.1` (Community Edition) | Local AWS cloud service emulator (Port `4566`) | Labs 1, 3, 5 |
| **OpenSSL** | `3.5.4` (30 Sep 2025) | Cryptographic key generation, TLS certificates, encryption | Lab 3 |
| **oathtool** | `2.6.14` (OATH Toolkit) | Time-based One-Time Password (TOTP) / MFA code generator | Lab 4 |
| **Trivy** | Executed via Docker (`aquasec/trivy`) | Container image vulnerability and misconfiguration scanner | Lab 4 |

---

## 4. Step-by-Step Implementation

### Step 1: Install & Configure Docker Engine
1. **User Group Privilege Assignment:** Add the current user (`anonym22`) to the `docker` security group to allow executing Docker commands without `sudo` privileges:
   ```bash
   sudo usermod -aG docker $USER
   ```
2. **Version Verification:** Confirm Docker installation and binary build details:
   ```bash
   docker --version
   ```
   *Result:* Confirmed Docker version `28.5.2+dfsg4`.
3. **Runtime Verification:** Run the official lightweight test container to ensure the Docker daemon is active and responsive:
   ```bash
   docker run --rm hello-world
   ```
   *Result:* Successfully pulled and executed `hello-world` image from Docker Hub.

### Step 2: Install & Verify AWS CLI v2
1. **AWS CLI Version Check:** Verify that AWS CLI v2 is correctly installed and accessible in the system `$PATH`:
   ```bash
   aws --version
   ```
   *Result:* Output confirmed `aws-cli/2.36.10`.

### Step 3: Install & Verify kind and kubectl
1. **kind Version Check:** Confirm the Kubernetes-in-Docker (`kind`) binary version:
   ```bash
   kind --version
   ```
   *Result:* Confirmed `kind version 0.23.0`.
2. **kubectl Client Version Check:** Verify the Kubernetes control CLI version:
   ```bash
   kubectl version --client
   ```
   *Result:* Confirmed `Client Version: v1.36.3`.
3. **Cluster Health & Context Verification:** Inspect the active Kubernetes cluster context (`kind-ccse`):
   ```bash
   kubectl cluster-info --context kind-ccse
   ```
   *Result:* Kubernetes control plane confirmed running at `https://127.0.0.1:33331`.

### Step 4: Verify Helper Security Tools (OpenSSL & oathtool)
1. **OpenSSL Verification:** Check OpenSSL toolkit version:
   ```bash
   openssl version
   ```
   *Result:* Confirmed `OpenSSL 3.5.4 30 Sep 2025`.
2. **oathtool Verification:** Verify OATH Toolkit CLI installation:
   ```bash
   oathtool --version
   ```
   *Result:* Confirmed `oathtool (OATH Toolkit) 2.6.14`.

### Step 5: Start & Verify LocalStack Environment
1. **Pull LocalStack Image:** Download LocalStack version 3.8 container image:
   ```bash
   docker pull localstack/localstack:3.8
   ```
2. **Launch LocalStack Container:** Deploy container in detached mode with port `4566` mapped to host:
   ```bash
   docker run -d --name localstack -p 4566:4566 localstack/localstack:3.8
   ```
3. **Container Process Check:** Verify running containers via Docker process listing:
   ```bash
   docker ps
   ```
   *Result:* Confirmed container `localstack` running on `0.0.0.0:4566->4566/tcp` and container `ccse-control-plane` on `127.0.0.1:33331->6443/tcp`.
4. **Health Endpoint Inspection:** Query LocalStack health status endpoint:
   ```bash
   curl http://localhost:4566/_localstack/health
   ```
   *Result:* Received JSON response confirming services (`iam`, `s3`, `sts`, `kms`, `cloudwatch`, etc.) status `"available"`.

### Step 6: One-Time AWS CLI Configuration & STS Identity Validation
1. **Set Dummy AWS Credentials:** Configure dummy credentials so AWS CLI operates smoothly with LocalStack:
   ```bash
   aws configure set aws_secret_access_key test
   aws configure set aws_access_key_id test
   aws configure set region us-east-1
   ```
2. **Define Local Endpoint Shortcut:** Define shell variable `$EP` pointing to local LocalStack port:
   ```bash
   EP='--endpoint-url=http://localhost:4566'
   ```
3. **Caller Identity Verification:** Test AWS STS `get-caller-identity` to verify AWS CLI communication with LocalStack:
   ```bash
   aws $EP sts get-caller-identity
   ```
   *Result:* Returned root account identity JSON with `Account: "000000000000"` and `Arn: "arn:aws:iam::000000000000:root"`.

---

## 5. Commands Used

Below is the complete reference list of terminal commands executed during the lab setup and verification phase:

```bash
# 1. Docker Setup & Permission Grants
sudo usermod -aG docker $USER
docker --version
docker run --rm hello-world

# 2. AWS CLI v2 Verification
aws --version

# 3. kind & kubectl Verification
kind --version
kubectl version --client
kubectl cluster-info --context kind-ccse

# 4. Security Helper Tools Verification
openssl version
oathtool --version

# 5. LocalStack Deployment & Verification
docker pull localstack/localstack:3.8
docker run -d --name localstack -p 4566:4566 localstack/localstack:3.8
docker ps
curl http://localhost:4566/_localstack/health

# 6. AWS CLI LocalStack Credentials & STS Identity Test
aws configure set aws_secret_access_key test
aws configure set aws_access_key_id test
aws configure set region us-east-1
EP='--endpoint-url=http://localhost:4566'
aws $EP sts get-caller-identity
```

---

## 6. Screenshots

### Screenshot 1: Docker User Permission & Version Check
![Docker User Permission & Version Check](Evidence/image1.png)
<img width="613" height="183" alt="image" src="https://github.com/user-attachments/assets/60439a79-bcf8-442d-bfd5-82fa0c3459ac" />


* **Terminal Prompt:** `(anonym22㉿kali)-[~]`
* **Command Executed:**
  ```bash
  sudo usermod -aG docker $USER
  docker --version
  ```
* **Output / Verification:**
  `Docker version 28.5.2+dfsg4, build 9cc6dea35e9a963f281434761c656fba4ac43aed`
* **Description:** Demonstrates adding user `anonym22` to the `docker` group and verifying Docker Engine version.

---

### Screenshot 2: Docker Hello-World Container Verification
![Docker Hello-World Container Verification](Evidence/image2.png)

* **Terminal Prompt:** `(anonym22㉿kali)-[~]`
* **Command Executed:**
  ```bash
  docker run --rm hello-world
  ```
* **Output / Verification:**
  ```text
  Hello from Docker!
  This message shows that your installation appears to be working correctly.
  ...
  ```
* **Description:** Confirms the Docker daemon is operational and able to pull/run container images from Docker Hub.

---

### Screenshot 3: AWS CLI v2 Version Verification
![AWS CLI v2 Version Verification](Evidence/image3.png)

* **Terminal Prompt:** `(anonym22㉿kali)-[~]`
* **Command Executed:**
  ```bash
  aws --version
  ```
* **Output / Verification:**
  `aws-cli/2.36.10 Python/3.14.6 Linux/6.16.8+kali-amd64 exe/x86_64.kali.2025`
* **Description:** Verifies AWS CLI v2 is properly installed on the host system.

---

### Screenshot 4: kind & kubectl Tool Versions
![kind & kubectl Tool Versions](Evidence/image4.png)

* **Terminal Prompt:** `(anonym22㉿kali)-[~]`
* **Command Executed:**
  ```bash
  kind --version
  kubectl version --client
  ```
* **Output / Verification:**
  * `kind version 0.23.0`
  * `Client Version: v1.36.3`
  * `Kustomize Version: v5.8.1`
* **Description:** Validates local Kubernetes tools (`kind` and `kubectl`) are ready for cluster management.

---

### Screenshot 5: Kubernetes Cluster Context & Health Status
![Kubernetes Cluster Context & Health Status](Evidence/image5.png)

* **Terminal Prompt:** `(anonym22㉿kali)-[~]`
* **Command Executed:**
  ```bash
  kubectl cluster-info --context kind-ccse
  ```
* **Output / Verification:**
  ```text
  Kubernetes control plane is running at https://127.0.0.1:33331
  CoreDNS is running at https://127.0.0.1:33331/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
  ```
* **Description:** Confirms active Kubernetes cluster `kind-ccse` control plane and CoreDNS services are up and accessible.

---

### Screenshot 6: Helper Security Tools Version Verification
![Helper Security Tools Version Verification](Evidence/image6.png)

* **Terminal Prompt:** `(anonym22㉿kali)-[~]`
* **Command Executed:**
  ```bash
  openssl version
  oathtool --version
  ```
* **Output / Verification:**
  * `OpenSSL 3.5.4 30 Sep 2025 (Library: OpenSSL 3.5.4 30 Sep 2025)`
  * `oathtool (OATH Toolkit) 2.6.14`
* **Description:** Verifies cryptographic (OpenSSL) and authentication (`oathtool`) helper utilities.

---

### Screenshot 7: LocalStack Docker Pull & Container Execution
![LocalStack Docker Pull & Container Execution](Evidence/image7.png)

* **Terminal Prompt:** `(anonym22㉿kali)-[~]`
* **Command Executed:**
  ```bash
  docker pull localstack/localstack:3.8
  docker run -d --name localstack -p 4566:4566 localstack/localstack:3.8
  docker ps
  ```
* **Output / Verification:**
  * Container ID: `4ea61a7ff920`
  * Active Containers in `docker ps`:
    * `localstack/localstack:3.8` (Name: `localstack`, Port: `0.0.0.0:4566->4566/tcp`)
    * `kindest/node:v1.30.0` (Name: `ccse-control-plane`, Port: `127.0.0.1:33331->6443/tcp`)
* **Description:** Demonstrates starting the LocalStack cloud simulator container and verifying running containers.

---

### Screenshot 8: LocalStack Health Check Output
![LocalStack Health Check Output](Evidence/image8.png)

* **Terminal Prompt:** `(anonym22㉿kali)-[~]`
* **Command Executed:**
  ```bash
  curl http://localhost:4566/_localstack/health
  ```
* **Output / Verification:**
  ```json
  {"services": {"acm": "available", "apigateway": "available", "cloudformation": "available", "cloudwatch": "available", "config": "available", "dynamodb": "available", "dynamodbstreams": "available", "ec2": "available", "es": "available", "events": "available", "firehose": "available", "iam": "available", "kinesis": "available", "kms": "available", "lambda": "available", "logs": "available", "opensearch": "available", "redshift": "available", "resource-groups": "available", "resourcegroupstaggingapi": "available", "route53": "available", "route53resolver": "available", "s3": "available", "s3control": "available", "scheduler": "available", "secretsmanager": "available", "ses": "available", "sns": "available", "sqs": "available", "ssm": "available", "stepfunctions": "available", "sts": "available", "support": "available", "swf": "available", "transcribe": "available"}, "edition": "community", "version": "3.8.1"}
  ```
* **Description:** Confirms LocalStack Community Edition 3.8.1 is fully healthy and all AWS core services are available.

---

### Screenshot 9: AWS CLI Configuration & STS Identity Verification
![AWS CLI Configuration & STS Identity Verification](Evidence/image9.png)

* **Terminal Prompt:** `(anonym22㉿kali)-[~]`
* **Command Executed:**
  ```bash
  aws configure set aws_secret_access_key test
  aws configure set aws_access_key_id test
  aws configure set region us-east-1
  EP='--endpoint-url=http://localhost:4566'
  aws $EP sts get-caller-identity
  ```
* **Output / Verification:**
  ```json
  {
      "UserId": "AKIAIOSFODNN7EXAMPLE",
      "Account": "000000000000",
      "Arn": "arn:aws:iam::000000000000:root"
  }
  ```
* **Description:** Validates that AWS CLI v2 successfully communicates with LocalStack endpoints and retrieves the default root identity.

---

## 7. Challenges Encountered

| Challenge / Symptom | Root Cause | Solution / Prevention |
| :--- | :--- | :--- |
| **Docker Permission Denied** (`permission denied while trying to connect to the Docker daemon socket`) | The current user lacked read/write access to Unix socket `/var/run/docker.sock`. | Executed `sudo usermod -aG docker $USER` and refreshed user session group membership. |
| **AWS CLI Routing to Real AWS** (`Could not connect to the endpoint URL`) | AWS CLI defaults to global AWS endpoints (e.g. `sts.amazonaws.com`) if no local override is specified. | Appended `--endpoint-url=http://localhost:4566` (using the `$EP` environment variable) to all AWS CLI commands. |
| **Port 4566 Binding Conflict** (`port is already allocated`) | An existing container or background process bound to port 4566. | Checked running containers with `docker ps` and freed port using `docker rm -f localstack` before starting fresh instance. |
| **Windows Shell Incompatibility (General Note)** | PowerShell/CMD lack native support for heredocs, single quotes, and `sha256sum`. | Ensured all lab execution steps are conducted inside a native Linux environment (Kali Linux / Bash). |

---

## 8. Lessons Learned

1. **Cost & Risk Mitigation via Cloud Emulation:** Local emulators like LocalStack and `kind` enable comprehensive hands-on cloud security training without incurring cloud billing risks, credential leaks, or internet dependency.
2. **Environment Standardization:** Maintaining explicit software version alignment (e.g., LocalStack 3.8.1, kind 0.23.0) prevents subtle API behavior discrepancies across different lab modules.
3. **Efficient CLI Workflow:** Utilizing environment variables like `EP='--endpoint-url=http://localhost:4566'` streamlines multi-command terminal sessions and reduces typing errors.
4. **Prerequisite Verification:** Performing systematic health checks (`curl .../_localstack/health`, `kubectl cluster-info`) before starting complex lab exercises ensures underlying services are healthy, minimizing troubleshooting overhead later.

---

## 9. References

1. **Lab Guide:** UniKL MIIT — Prof. Dr. Shahrulniza Musa, *IKB42603 Cloud Computing Security Essentials: Lab 0 Setup Cheatsheet*.
2. **Docker Documentation:** [Docker Engine Installation & User Guide](https://docs.docker.com/)
3. **AWS CLI v2 Command Reference:** [AWS CLI User Guide for Local & Custom Endpoints](https://docs.aws.amazon.com/cli/)
4. **LocalStack Documentation:** [LocalStack Docker Quickstart & Health API](https://docs.localstack.cloud/)
5. **Kubernetes in Docker (kind):** [kind User Guide & Cluster Management](https://kind.sigs.k8s.io/)
6. **OATH Toolkit:** [oathtool Command Line Tool Documentation](https://www.nongnu.org/oath-toolkit/)
