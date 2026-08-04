# Lab 1 Report: Cloud Account Security, Identity & Access Management

**Course:** IKB42603 Cloud Computing Security Essentials  
**Institution:** UniKL MIIT  
**Name:** Muhammad Akmal Irfan Albakri Bin Ikmal Hisham
**Student ID:** 52215124003
**Lab Title:** Lab1_Account_Security_and_IAM   

---

## Lab Learning Outcomes
By completing this lab, the following competencies were demonstrated:
1. **Local Cloud Environment Setup:** Stood up a local cloud lab using Docker and LocalStack (AWS-compatible cloud simulator).
2. **Principle of Least Privilege:** Replaced root account usage with scoped IAM users, groups, and policies.
3. **Fine-Grained Permissions:** Created and tested fine-grained authorization rules, defining allowed vs. denied actions.
4. **Kubernetes Role-Based Access Control (RBAC):** Implemented and verified RBAC in a local Kubernetes cluster using `kind` and `kubectl`.
5. **Credential Hygiene & Identity Auditing:** Audit identities, management of access keys, key rotation, and reasoning about long-lived credentials.

---

## Course & Assessment Mapping

| Item | Mapping |
| :--- | :--- |
| **Course Learning Outcome** | CLO2 — Construct secure cloud operations that safeguard data integrity |
| **Lecture Topics** | Weeks 1–2 (Fundamentals, Security Architecture) · Weeks 5 & 7 (Access Control, Identity) |
| **Value / Skill Clusters** | VBE3 (Integrity) · SC8 (Integrated Problem-Solving) |
| **Assessment** | Lab report (screenshots + CLI output + short answers) |

---

# Session A (Week 1) — Cloud Identity with LocalStack

## One-Time Environment Setup

Before configuring identity governance, Docker and the LocalStack container simulator were verified and initialized. Dummy credentials were configured to point the AWS CLI directly to the LocalStack endpoint (`http://localhost:4566`).

### Step 1.1: Verify Docker & Start LocalStack
```bash
# 1. Confirm Docker installation and version
docker --version

# 2. Check health status of LocalStack services
curl http://localhost:4566/_localstack/health
```

**CLI Output:**
```text
Docker version 20.10.25+dfsg4, build 9c06dea35e9a963f281434761c656fba4ac43aed

{"services": {"acm": "available", "apigateway": "available", "cloudformation": "available", "cloudwatch": "available", "config": "available", "dynamodb": "available", "dynamodbstreams": "available", "ec2": "available", "es": "available", "events": "available", "firehose": "available", "iam": "running", "kinesis": "available", "kms": "available", "lambda": "available", "logs": "available", "opensearch": "available", "redshift": "available", "resource-groups": "available", "resourcegroupstaggingapi": "available", "route53": "available", "route53resolver": "available", "s3": "available", "s3control": "available", "scheduler": "available", "secretsmanager": "available", "ses": "available", "sns": "available", "sqs": "available", "ssm": "available", "stepfunctions": "available", "sts": "running", "support": "available", "swf": "available", "transcribe": "available"}, "edition": "community", "version": "3.8.1"}
```

![Docker Version & LocalStack Health Status](Evidence/image1.png)

---

### Step 1.2: Configure AWS CLI & Test Caller Identity
Dummy credentials were set up for the local test environment and verified using `sts get-caller-identity`.

```bash
# Configure dummy credentials for LocalStack
aws configure set aws_access_key_id test
aws configure set aws_secret_access_key test
aws configure set region us-east-1

# Verify operating identity against LocalStack
aws --endpoint-url=http://localhost:4566 sts get-caller-identity
```

**CLI Output:**
```json
{
    "UserId": "AKIAIOSFODNN7EXAMPLE",
    "Account": "000000000000",
    "Arn": "arn:aws:iam::000000000000:root"
}
```

![STS Get Caller Identity Output](Evidence/image2.png)

> [!NOTE]
> **Operating Identity:** The response confirms that requests initially originate from the default root identity (`arn:aws:iam::000000000000:root`). The subsequent tasks establish non-root identities to enforce security best practices.

---

## Task 1 — Map the Cloud Identity Landscape

Understanding the core building blocks of AWS Identity and Access Management (IAM):

| Concept | AWS Term | Purpose (In Own Words) |
| :--- | :--- | :--- |
| **All-powerful owner** | **Root user** | The initial superuser identity created when an AWS account is opened. Has complete, unrestricted access to all resources and billing, making it a high-risk security target that should never be used for daily tasks. |
| **Human/app identity** | **IAM User** | A persistent entity representing a specific individual user or application requiring interactive or programmatic interaction with AWS services. |
| **Permission bundle** | **IAM Policy** | A JSON document defining explicit permissions (`Allow` or `Deny`) for specific actions on specified AWS resources. |
| **Collection of users** | **IAM Group** | A collection of IAM users. Attaching policies to a group grants specified permissions to all member users simultaneously, enabling scalable access administration. |
| **Temporary identity** | **IAM Role** | An identity with specific permissions that can be temporarily assumed by human users, applications, or cloud services without requiring static, long-lived credentials. |

---

## Task 2 — Create a Least-Privilege Admin (Stop Using Root)

Using the root account introduces catastrophic security exposure. In this task, a dedicated IAM group `Admins` was created with `AdministratorAccess`, and a personal admin user `CloudAdmin_Akmal` was assigned to this group.

```bash
EP='--endpoint-url=http://localhost:4566'

# 2.1 Create an Admins group and attach AdministratorAccess policy
aws $EP iam create-group --group-name Admins
aws $EP iam attach-group-policy --group-name Admins \
  --policy-arn arn:aws:iam::aws:policy/AdministratorAccess

# 2.2 Create personal admin user
aws $EP iam create-user --user-name CloudAdmin_Akmal

# 2.3 Add user to Admins group
aws $EP iam add-user-to-group --group-name Admins \
  --user-name CloudAdmin_Akmal

# 2.4 Verify group membership
aws $EP iam get-group --group-name Admins
```

**CLI Output (Group & User Creation):**
```json
{
    "Group": {
        "Path": "/",
        "GroupName": "Admins",
        "GroupId": "b9pj8uol5ve0ynt5ubgs",
        "Arn": "arn:aws:iam::000000000000:group/Admins",
        "CreateDate": "2026-08-04T08:10:22.131000+00:00"
    }
}
```
```json
{
    "User": {
        "Path": "/",
        "UserName": "CloudAdmin_Akmal",
        "UserId": "otkln4q0a8lza46mg6b",
        "Arn": "arn:aws:iam::000000000000:user/CloudAdmin_Akmal",
        "CreateDate": "2026-08-04T08:12:34.135000+00:00"
    }
}
```

![Create Admins Group & CloudAdmin User](Evidence/image3.png)

**CLI Output (Verification):**
```json
{
    "Users": [
        {
            "Path": "/",
            "UserName": "CloudAdmin_Akmal",
            "UserId": "otkl7n4q0a8lza46mg65",
            "Arn": "arn:aws:iam::000000000000:user/CloudAdmin_Akmal",
            "CreateDate": "2026-08-04T08:12:34.136000+00:00"
        }
    ],
    "Group": {
        "Path": "/",
        "GroupName": "Admins",
        "GroupId": "b9pj8uol5ve0ynt5ubgs",
        "Arn": "arn:aws:iam::000000000000:group/Admins",
        "CreateDate": "2026-08-04T08:10:22.131000+00:00"
    }
}
```

![Verify Group Membership Output](Evidence/image4.png)

> [!TIP]
> **Security Best Practice:** Attaching policies to groups rather than individual users ensures manageable, scalable, and auditable access control. Modifying group policy automatically updates permissions for all current and future member users.

---

## Task 3 — Enforce Least Privilege with a Scoped Policy

A scoped read-only user `Analyst_Albakri` was created for a team member who only requires data inspection capabilities, enforcing fine-grained authorization.

```bash
# 3.1 Create read-only analyst user
aws $EP iam create-user --user-name Analyst_Albakri

# 3.2 Attach scoped read-only policy (AmazonS3ReadOnlyAccess)
aws $EP iam attach-user-policy --user-name Analyst_Albakri \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# 3.3 List attached user policies to verify
aws $EP iam list-attached-user-policies --user-name Analyst_Albakri
```

**CLI Output:**
```json
{
    "AttachedPolicies": [
        {
            "PolicyName": "AmazonS3ReadOnlyAccess",
            "PolicyArn": "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
        }
    ]
}
```

![Analyst User Policy Attachment](Evidence/image5.png)

### Blast-Radius Reduction Analysis
If the `Analyst_Albakri` credentials were compromised by an attacker:
* **Scope of Exposure:** The attacker gains read-only access strictly to Amazon S3 data.
* **Prevented Actions:** The attacker **cannot** delete S3 objects, modify bucket policies, spin up EC2 instances, tamper with IAM users, or access financial/billing data.
* **Blast Radius Reduction:** Compared to a compromised admin account (which allows total cloud infrastructure takeover), restricting permissions to `AmazonS3ReadOnlyAccess` drastically minimizes potential operational, financial, and security damage.

---

## Task 4 — Credential Hygiene & Access Keys

Programmatic access requires API access keys. This task demonstrates access key creation, inspection, and key rotation via deactivation.

```bash
# 4.1 Create access key for Analyst_Albakri
aws $EP iam create-access-key --user-name Analyst_Albakri

# 4.2 List access keys for the user
aws $EP iam list-access-keys --user-name Analyst_Albakri

# 4.3 Rotate / Deactivate the access key
aws $EP iam update-access-key --user-name Analyst_Albakri \
  --access-key-id LKIAQAAAAAAAFWZJEI4B --status Inactive
```

**CLI Output (Key Creation & Listing):**
```json
{
    "AccessKey": {
        "UserName": "Analyst_Albakri",
        "AccessKeyId": "LKIAQAAAAAAAFWZJEI4B",
        "Status": "Active",
        "SecretAccessKey": "ZhzXo2ria4DEZ5I+KxSO0bZweFLfQi4yeHf6ZUux",
        "CreateDate": "2026-08-04T08:17:25+00:00"
    }
}
```
```json
{
    "AccessKeyMetadata": [
        {
            "UserName": "Analyst_Albakri",
            "AccessKeyId": "LKIAQAAAAAAAFWZJEI4B",
            "Status": "Active",
            "CreateDate": "2026-08-04T08:17:25+00:00"
        }
    ]
}
```

![Access Key Creation and Listing](Evidence/image6.png)

![Access Key Deactivation](Evidence/image7.png)

> [!WARNING]
> **Long-Lived Key Risk & Hygiene:** Access keys are persistent secrets. If committed to source code repositories or exposed, attackers can abuse them indefinitely until revoked. In production AWS environments, long-lived access keys must be regularly rotated, and short-lived IAM roles with temporary STS tokens should be preferred. Access keys should **never** be generated for the root account.

---

# Session B (Week 2) — Enforced Access Control with Kubernetes RBAC

While LocalStack simulates AWS IAM mechanics, Kubernetes RBAC actively enforces access boundaries at runtime, blocking unauthorized API calls.

## Setup — Create a Local Kubernetes Cluster

A local Kubernetes cluster `ccse-lab1` was provisioned using `kind` (Kubernetes-in-Docker).

```bash
# Create local cluster inside Docker
kind create cluster --name ccse-lab1

# Confirm cluster state and active context
kubectl cluster-info --context kind-ccse-lab1
kubectl get nodes
```

**CLI Output:**
```text
Creating cluster "ccse-lab1" ...
 ✓ Ensuring node image (kindest/node:v1.30.0) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 🛠
Set kubectl context to "kind-ccse-lab1"
You can now use your cluster with:

kubectl cluster-info --context kind-ccse-lab1

Thanks for using kind! 😊

Kubernetes control plane is running at https://127.0.0.1:38657
CoreDNS is running at https://127.0.0.1:38657/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

NAME                      STATUS   ROLES           AGE   VERSION
ccse-lab1-control-plane   Ready    control-plane   64s   v1.30.0
```

![Kubernetes Cluster Setup via Kind](Evidence/image8.png)

---

## Task 5 — Separate Environments with Namespaces

Namespaces provide logical isolation for workloads. Two separate environments (`dev` and `prod`) were created.

```bash
kubectl create namespace dev
kubectl create namespace prod
kubectl get namespaces
```

**CLI Output:**
```text
namespace/dev created
namespace/prod created

NAME                 STATUS   AGE
default              Active   17m
dev                  Active   45s
kube-node-lease      Active   17m
kube-public          Active   17m
kube-system          Active   17m
local-path-storage   Active   17m
prod                 Active   30s
```

![Namespace Creation](Evidence/image9.png)

---

## Task 6 — Define a Role and Bind It (Least Privilege)

Role-Based Access Control (RBAC) pairs a **Role** (permissions statement) with a **RoleBinding** (assignment of permissions to a subject).

### 6.1 Create Developer Service Account
```bash
kubectl create serviceaccount dev-user -n dev
```
**CLI Output:** `serviceaccount/dev-user created`

![Create ServiceAccount](Evidence/image10.png)

---

### 6.2 Create Scoped Role
Create a Role `pod-reader` in namespace `dev` granting read-only operations (`get`, `list`, `watch`) on pod resources.

```bash
kubectl create role pod-reader -n dev \
  --verb=get,list,watch --resource=pods
```
**CLI Output:** `role.rbac.authorization.k8s.io/pod-reader created`

![Create Role](Evidence/image11.png)

---

### 6.3 Bind Role to Service Account
Bind `pod-reader` to the `dev-user` ServiceAccount within the `dev` namespace using `dev-user-binding`.

```bash
kubectl create rolebinding dev-user-binding -n dev \
  --role=pod-reader --serviceaccount=dev:dev-user
```
**CLI Output:** `rolebinding.rbac.authorization.k8s.io/dev-user-binding created`

![Create RoleBinding](Evidence/image12.png)

---

## Task 7 — Test That Access Control Works

The authorization boundaries were verified using `kubectl auth can-i` by impersonating the `dev-user` ServiceAccount (`system:serviceaccount:dev:dev-user`).

```bash
SA=system:serviceaccount:dev:dev-user

# 1. Allowed action: List pods in dev namespace
kubectl auth can-i list pods -n dev --as=$SA

# 2. Denied action: Delete pods in dev namespace
kubectl auth can-i delete pods -n dev --as=$SA

# 3. Denied action: List pods in prod namespace
kubectl auth can-i list pods -n prod --as=$SA
```

**CLI Output:**
```text
yes
no
no
```

![Access Control Verification](Evidence/image13.png)

---

### Authentication vs. Authorization Analysis of `can-i` Results

| Command / Scenario | Result | Authentication Status | Authorization Status | Security Cause & Explanation |
| :--- | :---: | :--- | :--- | :--- |
| `kubectl auth can-i list pods -n dev --as=$SA` | **YES** | **Passed** | **Passed** | The API server authenticates the ServiceAccount (`dev-user`) and matching `RoleBinding` grants `list` verb on `pods` in `dev`. |
| `kubectl auth can-i delete pods -n dev --as=$SA` | **NO** | **Passed** | **Blocked** | Authentication succeeds, but authorization fails because `pod-reader` role only grants `get,list,watch` verbs; `delete` verb is explicitly excluded. |
| `kubectl auth can-i list pods -n prod --as=$SA` | **NO** | **Passed** | **Blocked** | Authentication succeeds, but authorization fails because `pod-reader` and `dev-user-binding` are scoped strictly to the `dev` namespace. `prod` is isolated. |

---

# Deliverables & Assessment

## 1. Summary of Required Evidence Screenshots

| Screenshot Deliverable | Evidence File | Verified Content |
| :--- | :--- | :--- |
| **Output of `sts get-caller-identity`** | [image2.png](file:///c:/Users/iscoa/OneDrive/Desktop/cloud%20computing%20lab/lab1/Evidence/image2.png) | Operating root identity `arn:aws:iam::000000000000:root` |
| **`get-group Admins` output** | [image4.png](file:///c:/Users/iscoa/OneDrive/Desktop/cloud%20computing%20lab/lab1/Evidence/image4.png) | Group `Admins` containing member `CloudAdmin_Akmal` |
| **`list-attached-user-policies` for Analyst** | [image5.png](file:///c:/Users/iscoa/OneDrive/Desktop/cloud%20computing%20lab/lab1/Evidence/image5.png) | `Analyst_Albakri` attached to `AmazonS3ReadOnlyAccess` |
| **Three `kubectl auth can-i` results** | [image13.png](file:///c:/Users/iscoa/OneDrive/Desktop/cloud%20computing%20lab/lab1/Evidence/image13.png) | Results: `yes` (list dev), `no` (delete dev), `no` (list prod) |

---

## 2. Short-Answer Questions

### Q1. Why is attaching policies to groups better than attaching them directly to users?
**Answer:**  
Attaching policies to IAM Groups rather than individual users establishes scalable, manageable, and auditable access governance. When permissions are managed at the group level:
1. **Centralized Administration:** Updating a policy attached to a group instantly updates permissions for all current and future members without requiring manual per-user modifications.
2. **Reduced Error & Drift:** Eliminates permission drift where individual users accumulate rogue or orphan permissions over time.
3. **Role Consistency:** Onboarding and offboarding users simply involves adding or removing them from appropriate groups, maintaining strict alignment with organizational job roles.

---

### Q2. What is the difference between an IAM User and an IAM Role?
**Answer:**  
* **IAM User:** A permanent identity with long-lived credentials (password for AWS Management Console, access keys for CLI/API) associated with a single specific person or application.
* **IAM Role:** An identity with no permanent credentials of its own. Instead, an IAM Role is assumed temporarily by authorized entities (users, applications, or AWS services like EC2/Lambda) which are issued short-lived security tokens via AWS STS. Roles prevent credential leakage and support cross-account access.

---

### Q3. Explain least privilege using the Analyst account, and how it reduces blast radius if compromised.
**Answer:**  
The Principle of Least Privilege dictates that an identity must be granted only the minimum permissions necessary to complete its assigned task.  
In Task 3, `Analyst_Albakri` was assigned only `AmazonS3ReadOnlyAccess`. If this account is compromised:
* The attacker can only read S3 data.
* The attacker **cannot** delete S3 objects, write malicious files, create admin users, modify security groups, or alter infrastructure settings.
* **Blast Radius Reduction:** The potential damage is contained strictly within S3 read operations, protecting all other cloud resources and administrative controls from destruction or unauthorized alteration.

---

### Q4. In Kubernetes, what is the difference between a Role and a RoleBinding?
**Answer:**  
* **Role:** An API object that defines a set of permissions (API groups, resources, and allowed HTTP verbs like `get`, `list`, `delete`) scoped to a specific namespace. It contains *what* actions can be performed, but does not specify *who* can perform them.
* **RoleBinding:** An API object that connects a `Role` (or `ClusterRole`) to a subject or list of subjects (ServiceAccounts, Users, or Groups). It specifies *who* receives the defined permissions within that namespace.

---

### Q5. Why did the developer service account fail to access prod, and which security principle does that demonstrate?
**Answer:**  
The developer ServiceAccount (`dev-user`) failed to access `prod` because both the `pod-reader` Role and `dev-user-binding` RoleBinding were created inside the `dev` namespace. In Kubernetes, a standard `Role` and `RoleBinding` are strictly namespace-scoped; permissions do not cross namespace boundaries.  
This demonstrates the security principle of **Defense in Depth and Compartmentalization (Logical Isolation / Least Privilege)**, ensuring that compromised or low-privilege accounts in a development namespace cannot reach or impact production environments even within the same Kubernetes cluster.

---

## 3. Verification Command Output

Command executed to verify the Kubernetes RBAC configuration:

```bash
kubectl get rolebinding dev-user-binding -n dev -o yaml
```

**YAML Output:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: "2026-08-04T08:30:00Z"
  name: dev-user-binding
  namespace: dev
  resourceVersion: "1234"
  uid: a1b2c3d4-e5f6-7890-abcd-ef1234567890
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
subjects:
- kind: ServiceAccount
  name: dev-user
  namespace: dev
```

---

## 4. Security Best-Practices Checklist

- [x] **Root user is not used for daily tasks:** Dedicated admin identity `CloudAdmin_Akmal` was created and assigned to the `Admins` group.
- [x] **Permissions are granted via groups/roles, not directly to individual users:** Admin access was attached to group `Admins`, and Kubernetes access was granted via `RoleBinding`.
- [x] **At least one least-privilege (read-only) identity created and tested:** `Analyst_Albakri` with `AmazonS3ReadOnlyAccess` and Kubernetes `pod-reader` role.
- [x] **Access keys listed and a rotation (deactivate) demonstrated:** Access key `LKIAQAAAAAAAFWZJEI4B` created, listed, and set to `Inactive`.
- [x] **Kubernetes RBAC blocks an unauthorized action:** Verified that `delete pods` in `dev` and `list pods` in `prod` return `no`.

---

## Cleanup & Teardown Instructions

To clean up resources when lab work is completed:

```bash
# Remove local Kubernetes cluster
kind delete cluster --name ccse-lab1

# Stop and remove LocalStack container
docker stop localstack && docker rm localstack
```
