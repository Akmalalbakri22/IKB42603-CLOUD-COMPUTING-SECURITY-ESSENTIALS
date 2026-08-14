# Lab 2 Report: Secure Isolation & Multi-Tenancy in Kubernetes & Docker

**Course:** IKB42603 Cloud Computing Security Essentials  
**Institution:** Universiti Kuala Lumpur Malaysian Institute of Information Technology (UniKL MIIT)  
**Lab Instructor:** Prof. Dr. Shahrulniza Musa  
**Session Scope:** Week 3 (Session A: Compute Isolation) & Week 4 (Session B: Network & Storage Isolation)  
**Environment:** Linux / Kali Linux (`anonym22@kali`), Docker Desktop/Engine, `kind` (Kubernetes in Docker v1.30.0), `kubectl`, Project Calico CNI (v3.27.0)

---

## Executive Summary & System Architecture

Multi-tenancy in cloud computing requires strict logical and physical isolation across compute, network, and storage resources. By default, shared Kubernetes infrastructure operates under a **default-open** networking model and unconstrained compute resource allocation. Without explicit security controls, pods residing in distinct namespaces can communicate freely across tenant boundaries, exhaust node CPU/memory capacity (the "noisy neighbour" problem), and potentially access sensitive secrets or data volumes belonging to other tenants.

This lab report details the step-by-step implementation, testing, and verification of secure isolation mechanisms across two distinct tenants (`tenant-a` and `tenant-b`) running on a shared Kubernetes cluster (`ccse-lab2`).


---

## 1. Technical Prerequisites & Environment Setup

### 1.1 Technical Prerequisites
* **Hardware:** Workstation/Laptop with at least 8 GB RAM and administrator/root privileges.
* **Container Runtime:** Docker Desktop / Docker Engine (active and running).
* **Cluster Tooling:** `kind` (Kubernetes in Docker) and `kubectl` CLI installed.
* **Network CNI:** Calico CNI manifest (`calico.yaml` v3.27.0).

### 1.2 Cluster Creation with Default CNI Disabled
Standard `kind` clusters use `kindnet`, which does **not** enforce Kubernetes `NetworkPolicy` objects. To ensure isolation rules take effect, a custom `kind` configuration is created disabling the default CNI and setting the pod network subnet (`192.168.0.0/16`).

#### Configuration File: `kind-config.yaml`
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  disableDefaultCNI: true
  podSubnet: 192.168.0.0/16
```

#### Execution Command:
```bash
kind create cluster --name ccse-lab2 --config kind-config.yaml
```

<img width="624" height="215" alt="image" src="https://github.com/user-attachments/assets/e4633f16-f72b-4413-aa53-9bb5d11f66f0" />


#### Terminal Verification Output:
```text
Creating cluster "ccse-lab2" ...
 ✓ Ensuring node image (kindest/node:v1.30.0) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-ccse-lab2"
You can now use your cluster with:

kubectl cluster-info --context kind-ccse-lab2
```

---

### 1.3 Deploying Project Calico CNI
With the default CNI disabled, cluster nodes remain in a `NotReady` state until Calico is applied to manage container networking and policy enforcement.

#### Execution Command:
```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.0/manifests/calico.yaml
```

<img width="624" height="451" alt="image" src="https://github.com/user-attachments/assets/a5df7ebb-372a-4ba2-b17e-219fecbe6dc5" />



#### Verifying Calico Node Rollout Status:
```bash
kubectl -n kube-system rollout status daemonset/calico-node --timeout=180s
```

<img width="624" height="60" alt="image" src="https://github.com/user-attachments/assets/c0a24a9c-33ef-4185-bbef-36c19f43315a" />


---

## 2. Session A (Week 3) — Compute Isolation & The Default-Open Risk

Session A demonstrates multi-tenant workload deployment on shared physical infrastructure and highlights the risks of Kubernetes' default-open networking model and unmanaged compute resource consumption.

---

### 2.1 Task 1 — Provisioning Two Tenant Workloads on One Cluster

Two logical tenants are established using Kubernetes namespaces: `tenant-a` and `tenant-b`. Each tenant deploys an Nginx web application and exposes it internally via a ClusterIP service.

#### Step 1: Create Tenant Namespaces
```bash
kubectl create namespace tenant-a
kubectl create namespace tenant-b
```
*Output:*
```text
namespace/tenant-a created
namespace/tenant-b created
```

#### Step 2: Deploy Web Applications for Both Tenants
```bash
kubectl -n tenant-a create deployment web --image=nginx
kubectl -n tenant-b create deployment web --image=nginx
```
*Output:*
```text
deployment.apps/web created
deployment.apps/web created
```

#### Step 3: Expose Deployments as Internal Services (Port 80)
```bash
kubectl -n tenant-a expose deployment web --port=80
kubectl -n tenant-b expose deployment web --port=80
```
*Output:*
```text
service/web exposed
service/web exposed
```

<img width="624" height="329" alt="image" src="https://github.com/user-attachments/assets/ab1c63b1-ebc2-495a-8426-17e298efe730" />



#### Step 4: Verify Tenant A Resources
```bash
kubectl get pods,svc -n tenant-a
```

<img width="777" height="143" alt="image" src="https://github.com/user-attachments/assets/9fa7c97c-bbc6-4075-adaa-65d009ae63eb" />


---

### 2.2 Task 2 — Observing & Proving the Default-Open Risk

By default, Kubernetes does not restrict traffic between namespaces. Any pod in any namespace can reach any service IP across the cluster.

#### Step 1: Retrieve Tenant B's Service IP
```bash
kubectl get svc web -n tenant-b -o jsonpath='{.spec.clusterIP}'; echo
```
*Output (Example ClusterIP):*
`10.96.39.3`

#### Step 2: Execute Cross-Tenant Network Probe from `tenant-a` to `tenant-b`
An ephemeral `curlimages/curl` container is executed in `tenant-a` targeting `tenant-b`'s service IP address:
```bash
kubectl -n tenant-a run probe --rm -it --image=curlimages/curl --image-pull-policy=IfNotPresent --restart=Never \
  -- curl -s -m 5 http://<B_IP> -o /dev/null -w 'HTTP %{http_code}\n'
```

<img width="776" height="173" alt="image" src="https://github.com/user-attachments/assets/72d61617-62a3-4e09-ba28-6b812898428d" />



#### Terminal Execution & Result:
```text
HTTP 200
pod "probe" deleted from tenant-a namespace
```

> [!WARNING]
> **Security Analysis (Default-Open Risk):**  
> An HTTP status code of **200** confirms that `tenant-a` successfully accessed `tenant-b`'s web server across namespace boundaries. On shared cloud infrastructure, default-open networking allows untrusted or compromised tenant workloads to scan, probe, and attack adjacent tenant workloads unless network isolation policies are explicitly enforced.

---

### 2.3 Task 3 — Contain the Noisy Neighbour (Resource Quotas)

Compute isolation requires preventing any single tenant from consuming an unfair share of shared cluster CPU, memory, or process handles (the "noisy neighbour" problem).

#### Step 1: Apply ResourceQuota to `tenant-a`
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: tenant-a-quota
  namespace: tenant-a
spec:
  hard:
    requests.cpu: "1"
    requests.memory: 512Mi
    pods: "5"
EOF
```
*Output:*
```text
resourcequota/tenant-a-quota created
```

#### Step 2: Inspect Applied Resource Quota
```bash
kubectl describe resourcequota tenant-a-quota -n tenant-a
```

<img width="624" height="304" alt="image" src="https://github.com/user-attachments/assets/f4a0be90-4aa6-4a19-b538-51b4495edb8f" />



#### Terminal Output:
```text
Name:            tenant-a-quota
Namespace:       tenant-a
Resource         Used  Hard
--------         ----  ----
pods             1     5
requests.cpu     0     1
requests.memory  0     512Mi
```

> [!TIP]
> **Resource Enforcement Significance:**  
> Enforcing `ResourceQuota` limits `tenant-a` to a maximum of 1 CPU core, 512 MiB of RAM, and 5 pods. If `tenant-a` attempts to launch additional workloads exceeding these limits, the Kubernetes API Server rejects the request, protecting `tenant-b` from resource starvation.

---

## 3. Session B (Week 4) — Network & Storage Isolation

Session B enforces network isolation using default-deny NetworkPolicies, demonstrates Role-Based Access Control (RBAC) secret isolation, and examines data remanence in container storage volumes.

---

### 3.1 Task 4 — Enforcing Default-Deny Network Isolation

Following the principle of least privilege / zero-trust segmentation, a **default-deny ingress** policy is applied to `tenant-b`. All incoming traffic is blocked unless explicitly permitted by an ingress rule.

#### Step 1: Apply Default-Deny Ingress Policy to `tenant-b`
```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: tenant-b
spec:
  podSelector: {}
  policyTypes: [Ingress]
EOF
```

<img width="624" height="153" alt="image" src="https://github.com/user-attachments/assets/8d4ea77f-ea13-430d-893b-f1f0096c635e" />



#### Step 2: Re-test Cross-Tenant Probe from `tenant-a` to `tenant-b`
Re-running the identical curl probe from Task 2 to verify policy enforcement:

```bash
kubectl -n tenant-a run probe --rm -it --image=curlimages/curl --restart=Never \
  -- curl -s -m 5 http://<B_IP> -o /dev/null -w 'HTTP %{http_code}\n'
```

<img width="776" height="92" alt="image" src="https://github.com/user-attachments/assets/5fce4d97-da4f-4e87-9a75-81fd9e936354" />



#### Operational Note (ResourceQuota Impact on Ephemeral Pods):
When running an ephemeral probe in `tenant-a` after applying `tenant-a-quota`, the Kubernetes API server enforces that all new pods must explicitly declare CPU/memory resource requests:
```text
Error from server (Forbidden): pods "probe" is forbidden: failed quota: tenant-a-quota: must specify requests.cpu for: probe; requests.memory for: probe
```

To run probes successfully within a quota-constrained namespace, resource requests are supplied inline or quota overrides are configured:
```bash
kubectl -n tenant-a run probe --rm -it --image=curlimages/curl --restart=Never \
  --overrides='{"spec":{"containers":[{"name":"probe","image":"curlimages/curl","resources":{"requests":{"cpu":"100m","memory":"64Mi"}}}]}}' \
  -- curl -s -m 5 http://10.96.23.191 -o /dev/null -w 'HTTP %{http_code}\n'
```

#### Terminal Execution & Result:
```text
command terminated with exit code 28
```

> [!IMPORTANT]
> **Before vs. After Isolation Proof:**
> * **Before Policy (Task 2):** Resulted in `HTTP 200` (Traffic Allowed).
> * **After Policy (Task 4):** Resulted in `Exit Code 28` / Timeout (Traffic Blocked by Calico CNI).
> 
> This before-and-after comparison provides undeniable runtime evidence of enforced network segmentation between cloud tenants.

---

### 3.2 Task 5 — Storage & Secret Isolation via Kubernetes RBAC

To prevent cross-tenant access to sensitive cryptographic keys or credentials, access to Kubernetes `Secret` objects must be constrained using Role-Based Access Control (RBAC).

#### Step 1: Create Tenant Secrets
```bash
kubectl -n tenant-a create secret generic data --from-literal=value=SECRET_A
kubectl -n tenant-b create secret generic data --from-literal=value=SECRET_B
```

#### Step 2: Configure Scoped ServiceAccount & Role in `tenant-a`
```bash
kubectl -n tenant-a create serviceaccount app-a
kubectl -n tenant-a create role reader --verb=get --resource=secrets
```

<img width="624" height="189" alt="image" src="https://github.com/user-attachments/assets/35f3c40d-6d2d-45df-8e82-6b92e1b277f2" />



#### Step 3: Create RoleBinding
```bash
kubectl -n tenant-a create rolebinding rb --role=reader --serviceaccount=tenant-a:app-a
```

<img width="624" height="55" alt="image" src="https://github.com/user-attachments/assets/7be565da-9561-4f1b-bcf8-f55427cd309c" />



#### Step 4: Verify Authorization with `kubectl auth can-i`
Test whether ServiceAccount `app-a` can read secrets within its own namespace versus `tenant-b`:

```bash
SA=system:serviceaccount:tenant-a:app-a

# Check access in tenant-a
kubectl auth can-i get secrets -n tenant-a --as=$SA

# Check access in tenant-b
kubectl auth can-i get secrets -n tenant-b --as=$SA
```

<img width="624" height="131" alt="image" src="https://github.com/user-attachments/assets/2c4655b8-ea18-441a-85b4-8efcb6df983c" />



#### Terminal Output:
```text
yes
no
```

> [!TIP]
> **RBAC Security Analysis:**  
> The output confirms `app-a` holds read access to `SECRET_A` within `tenant-a` (`yes`), but is explicitly denied access to `tenant-b` (`no`). Storage isolation is successfully enforced at the Kubernetes API Control Plane layer.

---

### 3.3 Task 6 — Data Remanence & Secure Deletion

Data remanence occurs when deleted data remains partially readable on un-sanitized storage volumes, allowing unauthorized recovery.

#### Step 1: Demonstrate Data Remanence Risk
A file containing sensitive patient records is written to a shared Docker volume `ccse-vol`, deleted using standard file system `rm`, and scanned for residual raw bytes:

```bash
docker run --rm -v ccse-vol:/data alpine sh -c \
 'echo SENSITIVE-PATIENT-RECORD > /data/phi.txt; sync; rm /data/phi.txt; \
 grep -a SENSITIVE /data/* 2>/dev/null; echo scan-done'
```

<img width="624" height="155" alt="image" src="https://github.com/user-attachments/assets/632e921c-7116-4b94-b852-56745931cdaa" />



#### Terminal Output:
```text
Unable to find image 'alpine:latest' locally
latest: Pulling from library/alpine
Digest: sha256:...
Status: Downloaded newer image for alpine:latest
scan-done
```

#### Step 2: Demonstrate Secure Wipe (Zero-Fill Overwriting)
To ensure complete sanitization before unmounting or releasing storage volumes, sensitive files must be overwritten with zeroes (`dd if=/dev/zero`) or random noise prior to deletion:

```bash
docker run --rm -v ccse-vol:/data alpine sh -c \
 'echo SENSITIVE > /data/phi2.txt; sync; \
 dd if=/dev/zero of=/data/phi2.txt bs=1k count=1 conv=notrunc; rm /data/phi2.txt; echo wiped'
```

<img width="624" height="83" alt="image" src="https://github.com/user-attachments/assets/6f876f93-6336-4f19-96ce-639ebe86904c" />



#### Terminal Output:
```text
wiped
```

> [!NOTE]
> **Cloud Storage Sanitization Rationale:**  
> In multi-tenant cloud environments (e.g., AWS EBS, Azure Disk, GCP Persistent Disk), tenants rarely have access to raw physical storage blocks. Therefore, the primary defense against data remanence in public cloud infrastructure is **Cryptographic Erasure** (destroying the encryption key managing the encrypted block volume).

---

## 4. Assessment & Short-Answer Questions

### Q1. Why can containers in different namespaces reach each other by default, and why is that dangerous in a multi-tenant cloud?
**Answer:**  
By default, Kubernetes implements a flat, un-segmented network model where all pods can route IP packets to any other pod across all namespaces without restriction (Container Network Interface default-open behavior). In a multi-tenant cloud, this default-open state is dangerous because an attacker who compromises a container in `tenant-a` can perform network reconnaissance, port scanning, and exploit unauthenticated microservices in `tenant-b`, leading to lateral movement and data breaches.

### Q2. Explain the default-deny principle and how your NetworkPolicy implements it.
**Answer:**  
The default-deny principle specifies that all network communications are implicitly blocked unless an explicit permissive rule allows them (Zero Trust). In Task 4, the `default-deny-ingress` NetworkPolicy specifies `podSelector: {}` (matching all pods in `tenant-b`) and `policyTypes: [Ingress]`, without declaring any allowed `ingress.from` sources. This instructs the Calico CNI controller to drop all incoming packets directed at pods in `tenant-b`, isolating the tenant from unauthorized external traffic.

### Q3. How do virtual machines and containers differ in isolation strength? When would you add a VM boundary?
**Answer:**  
Containers share the host operating system kernel and isolate workloads using Linux kernel primitives (`namespaces`, `cgroups`, `seccomp`). A kernel vulnerability (e.g., kernel privilege escalation or container escape) allows an attacker to compromise the host and all adjacent containers. Virtual Machines (VMs) provide stronger isolation by leveraging hardware-assisted virtualization (hypervisors like KVM/ESXi) with dedicated guest kernels. A VM boundary should be added when hosting untrusted code, processing highly regulated compliance data (e.g., PCI-DSS, HIPAA), or separating completely untrusted external multi-tenant workloads.

### Q4. What is data remanence, and why is cryptographic erasure the preferred cloud solution?
**Answer:**  
Data remanence is the residual physical representation of data that remains on storage media even after standard file deletion operations (`rm`). In cloud environments, storage is abstracted across virtual SANs/NAS arrays where physical disk sectors are continuously re-allocated to different customers. Because tenants cannot physically overwrite underlying hardware blocks, **cryptographic erasure** (encrypting data at rest and securely deleting the KMS decryption key) renders residual data permanently unreadable instantly.

### Q5. Which of the three isolation dimensions (compute, network, storage) did each task exercise?
**Answer:**
* **Task 1 & Task 3 (Compute Isolation):** Namespace workload partitioning and `ResourceQuota` CPU/memory limits prevent resource exhaustion.
* **Task 2 & Task 4 (Network Isolation):** Proving default-open risk and enforcing default-deny `NetworkPolicy` ingress filtering.
* **Task 5 & Task 6 (Storage Isolation):** Enforcing Kubernetes RBAC secret permissions and analyzing data remanence/secure disk sanitization.

---

## 5. Verification Commands & Best-Practices Checklist

### 5.1 Verification Commands Execution

#### Command 1: Inspect Active Namespaces
```bash
kubectl get namespaces
```

<img width="624" height="132" alt="image" src="https://github.com/user-attachments/assets/8b9a7069-3bc9-4ac2-905c-1a58a07a6583" />



#### Command 2: Inspect All Active NetworkPolicies
```bash
kubectl get networkpolicy -A
```

<img width="624" height="67" alt="image" src="https://github.com/user-attachments/assets/c19bfb09-aca3-4d17-b341-fedae4cb3f8d" />



*Output:*
```text
NAMESPACE   NAME                   POD-SELECTOR   AGE
tenant-b    default-deny-ingress   <none>         12m
```

#### Command 3: Inspect Applied Tenant ResourceQuotas
```bash
kubectl describe resourcequota tenant-a-quota -n tenant-a
```
*Output:*
```text
Name:            tenant-a-quota
Namespace:       tenant-a
Resource         Used  Hard
--------         ----  ----
pods             1     5
requests.cpu     0     1
requests.memory  0     512Mi
```

---

### 5.2 Security Best-Practices Checklist

| Security Control | Implementation Status | Verification Evidence |
| :--- | :---: | :--- |
| **Tenant Namespace Separation** | ✅ Verified | `tenant-a` and `tenant-b` isolated in distinct namespaces. |
| **Default-Deny Network Policy** | ✅ Verified | Blocked cross-tenant traffic (Before: HTTP 200, After: Timeout). |
| **Resource Quota Enforcement** | ✅ Verified | `tenant-a-quota` enforced (1 CPU, 512Mi RAM, 5 Pods). |
| **Per-Tenant RBAC Storage Isolation** | ✅ Verified | ServiceAccount `app-a` access: `tenant-a` = `yes`, `tenant-b` = `no`. |
| **Data Remanence & Sanitization** | ✅ Verified | Zero-fill overwriting (`dd`) and cryptographic erasure understood. |

---

## 6. Cleanup & Teardown

To release local workstation resources and delete the test environment:

```bash
# Delete Kubernetes Kind Cluster
kind delete cluster --name ccse-lab2

# Remove Storage Volume
docker volume rm ccse-vol
```

*Output:*
```text
Deleting cluster "ccse-lab2" ...
ccse-vol
```

---

## 7. Advanced Expansion Topics & Implementation Details

To achieve advanced security posture beyond standard requirements, three expansion topics were executed and documented during the lab:

### 7.1 Expansion 1: Egress Default-Deny & Micro-Segmentation

In addition to ingress isolation, restricting outbound (egress) network traffic prevents compromised containers from exfiltrating data or establishing command-and-control (C2) channels.

#### Step 1: Apply Default-Deny Egress Policy in `tenant-a`
```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: tenant-a
spec:
  podSelector: {}
  policyTypes:
  - Egress
EOF
```

<img width="624" height="171" alt="image" src="https://github.com/user-attachments/assets/057fc09c-3998-4132-b9bd-bfbdf51979f1" />



#### Step 2: Verify Egress Policy Creation in `tenant-a`
```bash
kubectl get networkpolicy -n tenant-a
```

<img width="624" height="59" alt="image" src="https://github.com/user-attachments/assets/a53e561e-0b34-44ea-b213-e2f34c4e3f28" />



#### Step 3: Selectively Allow DNS Egress to `kube-system`
```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: tenant-a
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: kube-system
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
EOF
```

<img width="624" height="305" alt="image" src="https://github.com/user-attachments/assets/752760e2-641c-4217-8c60-e24ba0bcd686" />



#### Step 4: Verify Egress Policies in `tenant-a`
```bash
kubectl get networkpolicy -n tenant-a
```

<img width="624" height="81" alt="image" src="https://github.com/user-attachments/assets/680bca52-d7ba-4ec3-b1c9-2117dc6a1058" />



*Output:*
```text
NAME                  POD-SELECTOR   AGE
allow-dns             <none>         32s
default-deny-egress   <none>         2m3s
```

#### Step 5: Configure Targeted Cross-Tenant Communication (Micro-segmentation)
Allowing egress from `tenant-a` specifically to `tenant-b` on HTTP port 80:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-to-tenant-b
  namespace: tenant-a
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: tenant-b
    ports:
    - protocol: TCP
      port: 80
EOF
```

<img width="624" height="332" alt="image" src="https://github.com/user-attachments/assets/1ea9da64-0ffe-4233-9bbf-179363a42364" />



Configuring corresponding ingress permission in `tenant-b` to accept traffic specifically from `tenant-a`:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-tenant-a
  namespace: tenant-b
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: tenant-a
    ports:
    - protocol: TCP
      port: 80
EOF
```

<img width="624" height="292" alt="image" src="https://github.com/user-attachments/assets/8cde1395-96e2-441c-a5d2-05c5b472bd45" />



#### Step 6: Testing Targeted Connectivity under ResourceQuota Rules
```bash
kubectl get pods -n tenant-b --show-labels
kubectl get svc web -n tenant-b -o jsonpath='{.spec.clusterIP}'; echo
```

<img width="777" height="265" alt="image" src="https://github.com/user-attachments/assets/2290c2e9-782a-4c80-ab14-df704d7a036f" />


---

### 7.2 Expansion 2: Enforcing Pod Security Admission (Restricted Profile)

Kubernetes Pod Security Admission (PSA) enforces security controls that prevent containers from running as root, escalating privileges, or sharing host namespaces.

#### Step 1: Label Tenant Namespaces with `restricted` Enforcement Level
```bash
kubectl label namespace tenant-a pod-security.kubernetes.io/enforce=restricted
kubectl label namespace tenant-b pod-security.kubernetes.io/enforce=restricted
```

<img width="624" height="280" alt="image" src="https://github.com/user-attachments/assets/c3cb10bd-8e89-48f4-b32f-a04a128febd4" />


#### Step 2: Verify Namespace Labels
```bash
kubectl get ns tenant-a tenant-b -L pod-security.kubernetes.io/enforce
```

<img width="624" height="140" alt="image" src="https://github.com/user-attachments/assets/5b4f80c8-1c67-43cb-a385-0c855f2faa5b" />

<img width="624" height="83" alt="image" src="https://github.com/user-attachments/assets/f3bec298-8a96-48b3-a0d8-1dbd1378d407" />


*Output:*
```text
NAME       STATUS   AGE     ENFORCE
tenant-a   Active   3h11m   restricted
tenant-b   Active   3h11m   restricted
```

#### Step 3: Test Privileged Pod Execution Rejection
Attempting to run a privileged pod within the restricted namespace:

```bash
kubectl -n tenant-a run privileged-test --image=nginx \
  --overrides='{"spec":{"containers":[{"name":"privileged-test","image":"nginx","securityContext":{"privileged":true}}]}}'
```

<img width="624" height="296" alt="image" src="https://github.com/user-attachments/assets/60b2530b-7f77-47e1-95a8-f7957ea8ebf9" />

<img width="624" height="256" alt="image" src="https://github.com/user-attachments/assets/8766ff28-4429-4c34-a4c5-b85ad3a683e3" />


#### Terminal Execution & Result:
```text
Error from server (Forbidden): pods "privileged-test" is forbidden: violates PodSecurity "restricted:latest": 
privileged (container "privileged-test" must not set securityContext.privileged=true), 
allowPrivilegeEscalation != false, unrestricted capabilities, runAsNonRoot != true, seccompProfile
```

#### Step 4: Test Unconfigured Pod Rejection under Restricted Policy
```bash
kubectl -n tenant-a run normal-test --image=nginx
```

<img width="624" height="179" alt="image" src="https://github.com/user-attachments/assets/225d54c6-2332-4d81-839c-af642f7ac76d" />

<img width="624" height="199" alt="image" src="https://github.com/user-attachments/assets/65b35efe-ce1d-4c54-af15-62da00d8ebb6" />



> [!TIP]
> **Pod Security Impact:**  
> Enforcing the `restricted` Pod Security Standard successfully blocks any attempt to run privileged containers, mitigating host-takeover and container-escape risks.

---

### 7.3 Expansion 3: Cluster-Wide Global Network Policy with Calico

While native Kubernetes `NetworkPolicy` objects are scoped per namespace, Calico Custom Resource Definitions (CRDs) allow cluster administrators to define `GlobalNetworkPolicy` objects that apply universally across all namespaces.

#### Step 1: Apply Calico `GlobalNetworkPolicy` for Tenant Isolation
```bash
cat <<EOF | kubectl apply -f -
apiVersion: crd.projectcalico.org/v1
kind: GlobalNetworkPolicy
metadata:
  name: tenant-isolation
spec:
  selector: 'projectcalico.org/namespace in {"tenant-a","tenant-b"}'
  types:
  - Ingress
  - Egress
  ingress:
  - action: Deny
  egress:
  - action: Deny
EOF
```

<img width="624" height="217" alt="image" src="https://github.com/user-attachments/assets/4d5f53be-eec6-4516-b183-2ac780cf8f4c" />



#### Step 2: Inspect Global Network Policy List
```bash
kubectl get globalnetworkpolicies
```

<img width="624" height="57" alt="image" src="https://github.com/user-attachments/assets/34246341-7479-4d78-8a3a-c9e22025a5cd" />



#### Step 3: Describe Global Network Policy Specification
```bash
kubectl describe globalnetworkpolicy tenant-isolation
```

<img width="624" height="288" alt="image" src="https://github.com/user-attachments/assets/ee1d83a7-9574-436c-9f6e-2b0458b849d2" />



#### Terminal Output:
```text
Name:         tenant-isolation
API Version:  crd.projectcalico.org/v1
Kind:         GlobalNetworkPolicy
Spec:
  Egress:
    Action:  Deny
  Ingress:
    Action:  Deny
  Selector:  projectcalico.org/namespace in {"tenant-a","tenant-b"}
  Types:
    Ingress
    Egress
```

> [!IMPORTANT]
> **Global Policy Value:**  
> Calico `GlobalNetworkPolicy` enables centralized security governance across all current and future tenant namespaces without requiring developer teams to write repetitive per-namespace NetworkPolicies.

---

## 8. Conclusion

This laboratory exercise successfully demonstrated the implementation and verification of secure multi-tenancy controls in Docker and Kubernetes:
1. **Compute Isolation:** Achieved via Kubernetes namespaces and `ResourceQuota` definitions (preventing noisy neighbour CPU/memory exhaustion).
2. **Network Isolation:** Transitioned from a default-open risk (proven via cross-tenant `HTTP 200` probe responses) to default-deny zero-trust network segmentation using Calico `NetworkPolicy` and `GlobalNetworkPolicy` enforcement.
3. **Storage Isolation:** Enforced using Kubernetes RBAC (ServiceAccounts, Roles, RoleBindings) for API secret access, while proving that cryptographic erasure is essential for cloud block storage sanitization against data remanence.

---
*Report compiled following UniKL MIIT Lab 2 Manual guidelines and validated against empirical CLI runtime executions.*
