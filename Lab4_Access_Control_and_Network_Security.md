# Lab 4 Report: Access Control & Network Security
**Course:** IKB42603 Cloud Computing Security Essentials  
**Institution:** Universiti Kuala Lumpur Malaysian Institute of Information Technology (UniKL MIIT)  
**Instructor:** Prof. Dr. Shahrulniza Musa  
**Lab Focus:** AuthN vs AuthZ, MFA/TOTP, Kubernetes RBAC, Network Segmentation, Default-Deny Firewall, Container Hardening & Vulnerability Scanning  
**Document Name:** `Environment-Setup.md`  

---

## 1. Executive Summary & Lab Learning Outcomes

This report provides a comprehensive, step-by-step implementation guide and technical analysis for **Lab 4: Access Control & Network Security**. Modern cloud-native security relies on the principle that **Identity is the Perimeter** combined with **Defense-in-Depth** across network and container execution layers.

### Key Learning Outcomes:
1. **Authentication (AuthN) vs. Authorization (AuthZ):** Distinguish identity verification (who you are) from permission enforcement (what you are allowed to do) using HTTP Basic Auth and Kubernetes Role-Based Access Control (RBAC).
2. **Multi-Factor Authentication (MFA/TOTP):** Implement and validate a Time-based One-Time Password algorithm (`oathtool`) to secure access against credential-based attacks.
3. **Network Access Control & Segmentation:** Construct a 3-tier container architecture (`frontend-net`, `backend-net`) to isolate database infrastructure from public-facing web tiers.
4. **Host-Level Firewalling (Default-Deny):** Configure `iptables` rules reflecting cloud Security Group models by dropping all traffic by default and explicitly permitting required ports.
5. **Container & Host Hardening:** Deploy unprivileged, non-root (`UID 1000`), read-only root filesystem containers with dropped Linux capabilities (`--cap-drop=ALL`).
6. **Vulnerability Assessment:** Conduct container image vulnerability scanning using Trivy (`aquasec/trivy`) to detect CVEs prior to production deployment.

---

## 2. Technical Prerequisites & Environment Setup

The execution environment was established on a Kali Linux container/VM workspace with the following tools configured:

| Tool | Function / Purpose | Installation / Command |
| :--- | :--- | :--- |
| **Docker** | Container runtime & network isolation | Pre-installed / Docker Engine |
| **kind** | Kubernetes IN Docker cluster deployment | `kind create cluster --name ccse-lab4` |
| **kubectl** | Kubernetes CLI for resource/RBAC management | Native binary |
| **oathtool** | Command-line TOTP token generator | `apt-get install oathtool` |
| **Trivy** | Container vulnerability scanner | `aquasec/trivy` container image |
| **iptables** | Linux kernel packet filtering & firewall | `apk add iptables` |

---

## 3. Session A (Week 7) — Authentication & Authorization

### Task 1 — Authentication: A Password-Protected Service

#### 1.1 Concept & Objective
Authentication proves a user's claimed identity. In Task 1, an HTTP Basic Authentication layer is deployed using Nginx and `htpasswd`. Requests lacking valid credentials are rejected with HTTP Status `401 Unauthorized`, whereas valid credentials return HTTP Status `200 OK`.

#### 1.2 Step-by-Step Implementation

1. **Generate Password File:**  
   Create an encrypted password entry for user `student` with password `P@ssw0rd!` using bcrypt/MD5 hashing via `httpd:alpine`:
   ```bash
   docker run --rm httpd:alpine htpasswd -nbB student 'P@ssw0rd!' > htpasswd.txt
   ```

2. **Configure Nginx HTTP Basic Auth (`default.conf`):**  
   Create Nginx configuration restricting access to `/` and requiring password authentication via `/etc/nginx/.htpasswd`:
   ```bash
   cat > default.conf <<'EOF'
   server { listen 80;
    location / { auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/.htpasswd;
    return 200 'Authenticated OK\n'; } }
   EOF
   ```

3. **Deploy Nginx Container (`authsvc`):**  
   Run Nginx on port 8080 mapping configuration and password files:
   ```bash
   docker run --rm -d --name authsvc -p 8080:80 \
    -v $(pwd)/default.conf:/etc/nginx/conf.d/default.conf \
    -v $(pwd)/htpasswd.txt:/etc/nginx/.htpasswd \
    nginx
   ```

4. **Verification & Testing:**  
   * **Unauthenticated Request (No Credentials):** Expect `HTTP 401`
     ```bash
     curl -s -o /dev/null -w 'no-creds: %{http_code}\n' http://localhost:8080
     # Output: no-creds: 401
     ```
   * **Authenticated Request (Valid Credentials):** Expect `HTTP 200`
     ```bash
     curl -s -u student:'P@ssw0rd!' http://localhost:8080
     # Output: Authenticated OK
     ```

#### 1.3 Evidence

<img width="624" height="127" alt="image" src="https://github.com/user-attachments/assets/00586813-133a-43d8-a17c-f8e57d7b450e" />

*Figure 1.1: Creating Nginx configuration `default.conf` requiring HTTP Basic Authentication.*

<img width="575" height="127" alt="image" src="https://github.com/user-attachments/assets/6679c94d-fb4d-46a6-9ebd-4200684f8fc1" />

*Figure 1.2: Launching `authsvc` container with volume mounts.*

<img width="852" height="162" alt="image" src="https://github.com/user-attachments/assets/1c3b4b86-1212-4241-8ed5-e67b645d6598" />

*Figure 1.3: Verification output showing `no-creds: 401` and valid credentials returning `Authenticated OK` (200).*

---

### Task 2 — Add a Second Factor (MFA / TOTP)

#### 2.1 Concept & Objective
Passwords alone are vulnerable to credential theft, phishing, and brute-force attacks. Multi-Factor Authentication (MFA) introduces a Time-based One-Time Password (TOTP) algorithm (RFC 6238), combining **something you know** (password) with **something you have** (authenticator device/secret key).

#### 2.2 Step-by-Step Implementation

1. **Generate Shared Base32 Secret Key:**  
   Generate a 20-byte random secret key encoded in Base32:
   ```bash
   SECRET=$(head -c20 /dev/urandom | base32)
   echo "Enroll this secret in an authenticator app: $SECRET"
   ```

2. **Generate Current 6-Digit TOTP Code:**  
   Use `oathtool` to compute the current 6-digit TOTP code from the secret key:
   ```bash
   oathtool --totp -b "$SECRET"
   ```

3. **Interactive Code Validation Logic:**  
   Compare user input (`$CODE`) against the dynamically evaluated TOTP code:
   ```bash
   read -p 'Enter the 6-digit code: ' CODE
   [ "$CODE" = "$(oathtool --totp -b "$SECRET")" ] && echo 'MFA OK' || echo 'MFA FAILED'
   ```

#### 2.3 Evidence

<img width="856" height="136" alt="image" src="https://github.com/user-attachments/assets/316fc73d-fe2b-4afe-b298-d42f9997ad28" />

*Figure 2.1: Generating Base32 secret key for authenticator enrollment.*

<img width="855" height="343" alt="image" src="https://github.com/user-attachments/assets/8f50543b-2bfa-4cf3-9aa2-6c9fab2e240c" />

*Figure 2.2: Computing TOTP code via `oathtool`, accepting user input, and validating with `MFA OK`.*

---

### Task 3 — Authorization: RBAC Roles

#### 3.1 Concept & Objective
While authentication proves identity, **authorization** defines permissions (least privilege). In Kubernetes, Role-Based Access Control (RBAC) binds Roles (containing API rules/verbs) to Service Accounts or Users within specific namespaces.

#### 3.2 Step-by-Step Implementation

1. **Initialize Kubernetes Cluster & Namespace:**  
   Deploy a local kind cluster named `ccse-lab4` and create an isolated namespace `app` with service account `dev`:
   ```bash
   kind create cluster --name ccse-lab4
   kubectl create namespace app
   kubectl create serviceaccount dev -n app
   ```

2. **Create Developer Role (`dev-role`) & Binding (`dev-rb`):**  
   Define a restrictive role permitting **only** `get` and `list` operations on `pods` within the `app` namespace:
   ```bash
   kubectl create role dev-role -n app --verb=get,list --resource=pods
   kubectl create rolebinding dev-rb -n app --role=dev-role --serviceaccount=app:dev
   ```

3. **Verify RBAC Permission Enforcement (`kubectl auth can-i`):**  
   Evaluate permissions for service account `system:serviceaccount:app:dev`:
   ```bash
   SA=system:serviceaccount:app:dev

   # 1. List pods (Allowed)
   kubectl auth can-i list pods -n app --as=$SA
   # Output: yes

   # 2. Create deployments (Denied)
   kubectl auth can-i create deploy -n app --as=$SA
   # Output: no

   # 3. Delete pods (Denied)
   kubectl auth can-i delete pods -n app --as=$SA
   # Output: no
   ```

#### 3.3 Evidence

<img width="767" height="157" alt="image" src="https://github.com/user-attachments/assets/9c38d799-4989-49d8-a497-57b32c31e601" />

*Figure 3.1: Creating namespace `app` and service account `dev`.*

<img width="852" height="142" alt="image" src="https://github.com/user-attachments/assets/52318b0d-f329-4fc9-89f9-2ce4e5396228" />

*Figure 3.2: Creating `dev-role` (read-only pods) and binding `dev-rb`.*

<img width="855" height="285" alt="image" src="https://github.com/user-attachments/assets/10941483-3ace-4c75-bd3d-dcdcb38350e4" />

*Figure 3.3: Evaluating permissions with `can-i`: `list pods` -> yes, `create deploy` -> no, `delete pods` -> no.*

---

## 4. Session B (Week 8) — Network Security & Hardening

### Task 4 — Network Segmentation (Three-Tier Architecture)

#### 4.1 Concept & Objective
Network segmentation implements **Defense-in-Depth** by isolating container workloads onto separate virtual bridge networks. In a 3-tier web application architecture:
- `web` (Frontend): Connected **only** to `frontend-net`.
- `db` (Database): Connected **only** to `backend-net`.
- `app` (Application / Middleware): Connected to **both** `frontend-net` and `backend-net`.

This prevents a compromised frontend container from directly communicating with or attacking the database server.

```
 [ Public Internet ]
          │
          ▼
    ┌───────────┐
    │    web    │ (frontend-net)
    └─────┬─────┘
          │ (BLOCKED from db)
          ▼
    ┌───────────┐
    │    app    │ (frontend-net & backend-net)
    └─────┬─────┘
          │ (REACHABLE to db)
          ▼
    ┌───────────┐
    │    db     │ (backend-net)
    └───────────┘
```

#### 4.2 Step-by-Step Implementation

1. **Create Segmented Docker Networks:**
   ```bash
   docker network create frontend-net
   docker network create backend-net
   ```

2. **Deploy Tiered Containers:**
   ```bash
   # DB tier on backend-net only
   docker run -d --name db --network backend-net redis:alpine

   # App tier on backend-net, then connected to frontend-net
   docker run -d --name app --network backend-net nginx
   docker network connect frontend-net app

   # Web tier on frontend-net only
   docker run -d --name web --network frontend-net nginx
   ```

3. **Verify Network Isolation:**
   * **Web -> DB (Frontend to Database Direct Access):** Must FAIL (`BLOCKED`)
     ```bash
     docker exec web sh -c 'apk add -q curl; curl -s -m 3 db:6379 || echo BLOCKED'
     # Output: BLOCKED
     ```
   * **App -> DB (Backend to Database Access):** Must SUCCEED (`REACHABLE`)
     ```bash
     docker exec app sh -c 'apk add -q curl; nc -z -w3 db 6379 && echo REACHABLE'
     # Output: Connection to db (172.20.0.2) 6379 port [tcp/*] succeeded! REACHABLE
     ```

#### 4.3 Evidence

<img width="852" height="168" alt="image" src="https://github.com/user-attachments/assets/7f1b647b-09b6-42d7-b65d-215d7b760868" />

*Figure 4.1: Creating Docker networks `frontend-net` and `backend-net`.*

<img width="852" height="322" alt="image" src="https://github.com/user-attachments/assets/f984b30a-b75f-4a4a-b535-253b69f09b41" />

*Figure 4.2: Launching database container `db` on `backend-net`.*

<img width="852" height="222" alt="image" src="https://github.com/user-attachments/assets/943d829b-136f-4fb4-88ab-a24c0fd17ab5" />

*Figure 4.3: Deploying `app` on `backend-net`, connecting `app` to `frontend-net`, and deploying `web` on `frontend-net`.*

<img width="852" height="216" alt="image" src="https://github.com/user-attachments/assets/efd19b59-cd1a-4937-afcc-5e149a63accb" />

*Figure 4.4: Probing network routes: `web` -> `db` is BLOCKED, while `app` -> `db` is REACHABLE.*

---

### Task 5 — Firewall Rules (Default-Deny Policy)

#### 5.1 Concept & Objective
A **default-deny policy** enforces least privilege at the host network layer. By dropping all inbound traffic (`INPUT DROP`) by default and explicitly allowing required services (e.g., HTTPS on port 443 and local loopback `lo`), the attack surface is restricted. This directly mirrors Cloud Security Group rules (e.g., AWS Security Groups, Azure NSGs).

#### 5.2 Step-by-Step Implementation

Execute a throwaway Alpine Linux container with kernel network administration privileges (`--cap-add=NET_ADMIN`) to configure `iptables`:

```bash
docker run --rm --cap-add=NET_ADMIN alpine sh -c '
 apk add -q iptables; \
 iptables -P INPUT DROP; \
 iptables -A INPUT -p tcp --dport 443 -j ACCEPT; \
 iptables -A INPUT -i lo -j ACCEPT; \
 iptables -L INPUT -n'
```

**Rule Explanation:**
- `iptables -P INPUT DROP`: Sets default policy to DROP for all incoming packets.
- `iptables -A INPUT -p tcp --dport 443 -j ACCEPT`: Appends rule accepting TCP traffic targeting port 443 (HTTPS).
- `iptables -A INPUT -i lo -j ACCEPT`: Accepts all traffic on the loopback interface (`127.0.0.1`).
- `iptables -L INPUT -n`: Lists active rules in numeric format.

#### 5.3 Evidence

<img width="856" height="266" alt="image" src="https://github.com/user-attachments/assets/7c29696e-9b9f-42f5-a758-19f0c9057eb2" />

*Figure 5.1: `iptables` output confirming default policy DROP with explicit ACCEPT rules for TCP port 443 and loopback.*

---

### Task 6 — Container / Host Hardening & Vulnerability Scanning

#### 6.1 Concept & Objective
Container hardening reduces runtime vulnerabilities by removing unnecessary Linux privileges and restricting file system access:
- **Non-root Execution (`--user 1000:1000`):** Prevents container escapes from acquiring host root privileges.
- **Read-Only Root Filesystem (`--read-only`):** Prevents malware persistence and unauthorized script execution.
- **Capability Dropping (`--cap-drop=ALL`):** Strips kernel privileges (`CAP_SYS_ADMIN`, `CAP_NET_ADMIN`, etc.).
- **Privilege Escalation Prevention (`--security-opt no-new-privileges`):** Prevents setuid/setgid execution.
- **Temporary In-Memory Storage (`--tmpfs /tmp`):** Provides a transient writable directory for non-persistent runtime files.

#### 6.2 Step-by-Step Implementation

1. **Deploy Hardened Container:**
   ```bash
   docker run -d --name hardened \
    --user 1000:1000 \
    --read-only \
    --cap-drop=ALL \
    --security-opt no-new-privileges \
    --tmpfs /tmp \
    nginxinc/nginx-unprivileged
   ```

2. **Verify Hardening Flags via `docker inspect`:**
   ```bash
   docker inspect hardened --format 'User={{.Config.User}} ReadOnly={{.HostConfig.ReadonlyRootfs}}'
   # Expected Output: User=1000:1000 ReadOnly=true
   ```

3. **Container Image Vulnerability Scanning (`Trivy`):**  
   Scan the `nginx:alpine` image for `HIGH` and `CRITICAL` severity CVEs:
   ```bash
   docker run --rm aquasec/trivy image --severity HIGH,CRITICAL nginx:alpine | head -20
   ```

#### 6.3 Evidence

<img width="855" height="433" alt="image" src="https://github.com/user-attachments/assets/9290177c-fd2b-497d-bc13-683073c12fce" />

*Figure 6.1: Running `hardened` container using `nginxinc/nginx-unprivileged` with security flags.*

<img width="852" height="112" alt="image" src="https://github.com/user-attachments/assets/3ee3ae2d-bb62-4c1b-a544-47e9f67338db" />

*Figure 6.2: Inspect output confirming `User=1000:1000` and `ReadOnly=true`.*

<img width="852" height="430" alt="image" src="https://github.com/user-attachments/assets/17ee7234-cf1d-4587-9985-269de383d7c9" />

*Figure 6.3: Initializing Trivy vulnerability database download.*

<img width="856" height="422" alt="image" src="https://github.com/user-attachments/assets/3f23cf53-d1ff-428e-a078-27ff806e5b55" />

*Figure 6.4: Trivy vulnerability report summary for `nginx:alpine (alpine 3.24.1)` identifying 2 HIGH severity findings.*

---

### Task 6 Extension — Custom Hardened Dockerfile Build

As part of advanced container hardening, a custom unprivileged Nginx image was defined via `Dockerfile`, built, and deployed under strict security parameters.

#### Custom `Dockerfile` Content:
```dockerfile
FROM nginxinc/nginx-unprivileged
USER 1000:1000
```

#### Execution & Inspection Commands:
```bash
# 1. Build image
docker build -t hardened-nginx .

# 2. Run hardened instance
docker run -d --name hardened \
 --read-only \
 --cap-drop=ALL \
 --security-opt no-new-privileges \
 --tmpfs /tmp \
 hardened-nginx

# 3. Inspect configuration
docker inspect hardened --format 'User={{.Config.User}} ReadOnly={{.HostConfig.ReadonlyRootfs}}'
```

#### Extension Evidence

<img width="787" height="155" alt="image" src="https://github.com/user-attachments/assets/53d582b0-2181-478e-9119-88a0837f3a19" />

*Figure 6.5: Creating Dockerfile specifying `FROM nginxinc/nginx-unprivileged` and `USER 1000:1000`.*

<img width="855" height="320" alt="image" src="https://github.com/user-attachments/assets/cfe94d2d-7e72-4d02-892a-871bedfd2b9c" />

*Figure 6.6: Successfully building `hardened-nginx:latest`.*

<img width="852" height="345" alt="image" src="https://github.com/user-attachments/assets/aecd1549-b8cb-4bc5-86e4-143d4c119cd2" />

*Figure 6.7: Deploying custom `hardened-nginx` image and verifying `User=1000:1000` and `ReadOnly=true`.*

---

## 5. Deliverables & Assessment: Short-Answer Questions

### Q1. Explain the difference between authentication and authorization using Tasks 1 and 3.

* **Authentication (AuthN - Task 1):** Proves **who you claim to be** (identity verification). In Task 1, Nginx HTTP Basic Authentication asks for credentials. When valid credentials (`student:P@ssw0rd!`) are supplied, identity is proven, returning `HTTP 200 Authenticated OK`. Unauthenticated requests are rejected with `HTTP 401 Unauthorized`. Authentication does not evaluate what actions the user is allowed to perform once inside.
* **Authorization (AuthZ - Task 3):** Determines **what actions an authenticated identity is permitted to execute** (access control enforcement). In Task 3, after identity is established, Kubernetes RBAC evaluates permissions for service account `app:dev`. Although authenticated, `app:dev` is granted a role permitting only `get` and `list` verbs on `pods`. When `app:dev` attempts `list pods`, authorization returns `yes`. When attempting `create deploy` or `delete pods`, authorization returns `no`. This demonstrates that authentication proves identity, while authorization enforces boundaries.

---

### Q2. Why is MFA so effective, and which attacks does it defeat?

* **Why MFA is Effective:** Multi-Factor Authentication (MFA) requires two or more orthogonal authentication factors from distinct categories:
  1. *Knowledge Factor:* Something you know (e.g., password).
  2. *Possession Factor:* Something you have (e.g., TOTP secret key / authenticator device).
  
  Even if an attacker captures the user's password, they cannot generate the valid, time-sensitive 6-digit TOTP code (`oathtool --totp`), which dynamically changes every 30 seconds based on HMAC-SHA1.

* **Attacks Defeated by MFA:**
  1. **Credential Stuffing:** Automated reuse of breached username/password combinations across multiple platforms.
  2. **Brute-Force & Dictionary Attacks:** High-speed guessing of static passwords.
  3. **Phishing (Standard):** Stolen static passwords become useless to attackers without real-time physical possession of the TOTP token generator.
  4. **Keylogging / Shoulder Surfing:** Intercepted password data is insufficient to compromise the account.

---

### Q3. How does network segmentation limit the damage of a compromised web server?

* **Mechanism:** Network segmentation divides a infrastructure into isolated broadcast/routing domains. In Task 4, the `web` container resides strictly on `frontend-net`, while `db` resides on `backend-net`. Only `app` is attached to both networks.
* **Blast Radius Containment:** If an attacker exploits a remote code execution (RCE) vulnerability in the public-facing `web` container, network routing blocks direct IP traffic to `db` (verified by `curl db:6379` resulting in `BLOCKED`). The attacker cannot directly pivot, query, dump, or overwrite database tables. Lateral movement is contained to the compromised tier, upholding **Defense-in-Depth**.

---

### Q4. What does a default-deny firewall policy achieve, and how does it relate to cloud security groups?

* **Achievement of Default-Deny:** A default-deny policy (e.g., `iptables -P INPUT DROP`) automatically rejects/drops all incoming packets unless an explicit rule allows them (e.g., `ACCEPT` port 443). This enforces the **Principle of Least Privilege** at the network layer, ensuring unneeded ports (such as SSH, database, remote management, or debug ports) remain completely stealth and unreachable to external networks.
* **Relation to Cloud Security Groups:** Cloud Security Groups (e.g., AWS SGs, Azure NSGs, GCP Firewall Rules) operate on an implicit default-deny architecture. All inbound traffic is blocked by default until security engineers add explicit inbound allow rules (e.g., allow TCP 443 from `0.0.0.0/0`). Task 5 mirrors this cloud model at the container host interface level.

---

### Q5. List the hardening measures you applied and the attack surface each one removes.

| Hardening Measure | Configuration / Flag Applied | Attack Surface / Threat Removed |
| :--- | :--- | :--- |
| **1. Non-Root Execution** | `--user 1000:1000` / `nginx-unprivileged` | Eliminates root privilege inside container (`UID 0`). Prevents container escape exploits from acquiring root control of host OS kernel. |
| **2. Read-Only Root Filesystem** | `--read-only` | Prevents attackers from downloading malware payloads, writing web shells, altering system binaries, or establishing persistent backdoors. |
| **3. Dropping Linux Capabilities** | `--cap-drop=ALL` | Strips kernel-level capabilities (e.g., `CAP_SYS_ADMIN`, `CAP_NET_ADMIN`). Prevents kernel privilege escalation, raw socket manipulation, and host tampering. |
| **4. Prevent Privilege Escalation** | `--security-opt no-new-privileges` | Blocks processes from acquiring additional privileges via `setuid` or `setgid` binaries (SUID/SGID execution prevention). |
| **5. Transient Writable Volume** | `--tmpfs /tmp` | Provides temporary, volatile in-memory storage for necessary app logs/sockets without exposing persistent disk write permissions. |
| **6. Image Vulnerability Scanning** | `aquasec/trivy` | Identifies known CVEs and vulnerable software packages in base images before deployment, allowing proactive patch management. |

---

## 6. Verification & Audit Commands

To verify the lab security controls post-deployment, run the following verification commands:

### 6.1 Kubernetes RBAC RoleBinding Audit
Verify that `dev-rb` binds `dev-role` to service account `app:dev` in namespace `app`:
```bash
kubectl get rolebinding dev-rb -n app -o yaml
```

### 6.2 Docker Capability Drop Audit
Verify that all Linux capabilities have been dropped from the `hardened` container:
```bash
docker inspect hardened --format '{{json .HostConfig.CapDrop}}'
# Expected Output: ["ALL"]
```

---

## 7. Security Best-Practices Checklist

- [x] **Service Authentication:** Unauthenticated requests are rejected (`HTTP 401`); valid credentials accepted (`HTTP 200`).
- [x] **Multi-Factor Authentication (MFA):** TOTP algorithm implemented and validated via `oathtool` returning `MFA OK`.
- [x] **Authorization & RBAC:** Role-Based Access Control enforced in Kubernetes (`list pods` -> allowed; administrative actions -> denied).
- [x] **Network Segmentation:** 3-tier architecture deployed; `web` tier isolated from direct access to `db` tier (`BLOCKED`).
- [x] **Default-Deny Firewall:** `iptables` default policy set to DROP with explicit ALLOW rules for port 443 and loopback.
- [x] **Container Hardening:** Container configured with non-root user (`1000:1000`), read-only root filesystem, dropped capabilities (`ALL`), and scanned via Trivy.

---

## 8. Cleanup & Teardown Instructions

After completing verification and report generation, tear down all temporary containers, networks, and Kubernetes clusters:

```bash
# 1. Stop and remove all lab containers
docker rm -f authsvc db app web hardened 2>/dev/null

# 2. Remove custom networks
docker network rm frontend-net backend-net 2>/dev/null

# 3. Delete kind Kubernetes cluster
kind delete cluster --name ccse-lab4
```

---
*Report generated successfully following guide `IKB42603_Lab4_Access_Control_and_Network_Security.pdf` and verified evidence files.*
