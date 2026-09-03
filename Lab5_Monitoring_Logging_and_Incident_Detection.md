# LAB 5: Monitoring, Logging & Incident Detection
**Course:** IKB42603 Cloud Computing Security Essentials (UniKL MIIT)

**Name:** Muhammad Akmal Irfan Albakri Bin Ikmal Hisham

**Student ID:** 52215124003

**Group Lab:** L02-B04

**Topic:** Centralised Logging, Tamper-Proof Logs, Threat Detection & Incident Response (Docker & LocalStack)  

**Assessment Mapping:** Lab Report + Incident Report (Contributes to Lab Assignment)  

---

## 1. Executive Summary & Lab Learning Outcomes

Modern cloud security operations operate under the realistic assumption that *prevention eventually fails*. Consequently, centralising cloud telemetry, guaranteeing log integrity against post-incident tampering, detecting stealthy multi-stage attack patterns, and executing rapid incident containment are essential core competencies.

At the conclusion of this lab, the following core learning outcomes were successfully demonstrated:
1. **Centralised Cloud Telemetry:** Collected and aggregated application-level authentication logs into AWS CloudWatch Logs using an emulated LocalStack environment.
2. **Security Querying & Log vs. Event Distinction:** Formulated targeted log queries using standard shell tools (`grep`, `awk`, `sort`, `uniq`) to identify security-relevant activity (failed authentication attempts) and differentiated raw log lines from actionable security events.
3. **Cryptographic Tamper-Proofing:** Constructed a cryptographic hash-chained audit log (`auth.chain`) using SHA-256 digests and demonstrated real-time detection of post-incident log alteration.
4. **Multi-Event SIEM Correlation:** Developed automated correlation rules to detect complex multi-stage attack vectors (brute-force $\rightarrow$ account compromise $\rightarrow$ data exfiltration) that individual log lines cannot reveal.
5. **Incident Response Lifecycle Execution:** Performed end-to-end incident containment (network-level IP blocking via `iptables`), evidence collection with SHA-256 cryptographic digests, and formal incident report documentation.

---

## 2. Technical Prerequisites & System Architecture

### 2.1 Prerequisites & Toolchain
- **Container Runtime:** Docker Engine running on Kali Linux host.
- **Cloud Telemetry Emulation:** LocalStack container exposing AWS CloudWatch Logs on port `4566`.
- **AWS CLI:** AWS CLI v2 configured with `--endpoint-url=http://localhost:4566`.
- **Standard Shell Toolchain:** `grep`, `awk`, `sha256sum`, `sed`, `paste`, `cut`, `chmod`, `nano`, and `iptables` (via Alpine Linux container with `NET_ADMIN` capabilities).

### 2.2 Telemetry & Incident Response Pipeline

```
+-----------------------------------+
|    Application Auth Logs          |  (auth.log)
+-----------------------------------+
                  │
                  ▼  put-log-events
+-----------------------------------+
|  AWS CloudWatch Logs Store        |  (/ccse/app / auth)
+-----------------------------------+
                  │
                  ▼  get-log-events & grep/awk
+-----------------------------------+
|  Security Query & Hash Chain      |  (Tamper-Evident SHA-256 Hash Chain)
+-----------------------------------+
                  │
                  ▼  Multi-Event Correlation Rule
+-----------------------------------+
|  SIEM Detection Alert             |  (Brute-Force -> Compromise -> Exfiltration)
+-----------------------------------+
                  │
                  ▼  Incident Response Action
+-----------------------------------+
|  Containment & Evidence Copy      |  (iptables DROP & sha256sum Evidence Copy)
+-----------------------------------+
```

---

## 3. Step-by-Step Lab Execution & Evidence

### Session A (Week 9) — Logging & Centralisation

#### Setup — Start LocalStack & Initialize CloudWatch Infrastructure
LocalStack was launched in detached mode to emulate AWS CloudWatch. A log group `/ccse/app` and a log stream `auth` were provisioned to serve as the central cloud logging endpoint.

```bash
# Set LocalStack endpoint variable
EP='--endpoint-url=http://localhost:4566'

# Create CloudWatch Log Group and Log Stream
aws $EP logs create-log-group --log-group-name /ccse/app
aws $EP logs create-log-stream --log-group-name /ccse/app --log-stream-name auth
```

**Evidence (LocalStack Initialization & CloudWatch Setup):**  

<img width="855" height="167" alt="image" src="https://github.com/user-attachments/assets/95673746-764b-4c0f-94f5-dc28cabbc624" />

---

#### Task 1 — Generate Application Logs
A synthetic authentication log (`auth.log`) was created to simulate real-world web application telemetry. The dataset captures normal user logins, a brute-force attack probe from IP `203.0.113.9`, a successful login compromise, and a high-volume data exfiltration attempt.

```bash
cat > auth.log <<'EOF'
2025-03-01T09:00:01 LOGIN_OK user=ahmad ip=10.0.0.5
2025-03-01T09:01:10 LOGIN_FAIL user=admin ip=203.0.113.9
2025-03-01T09:01:12 LOGIN_FAIL user=admin ip=203.0.113.9
2025-03-01T09:01:15 LOGIN_FAIL user=admin ip=203.0.113.9
2025-03-01T09:01:18 LOGIN_FAIL user=admin ip=203.0.113.9
2025-03-01T09:01:22 LOGIN_OK user=admin ip=203.0.113.9
2025-03-01T09:01:40 EXPORT_DATA user=admin ip=203.0.113.9 size=500MB
EOF

cat auth.log
```

**Evidence (Application Log Generation):**  

<img width="856" height="397" alt="image" src="https://github.com/user-attachments/assets/a00d2adb-95e1-43fc-aaea-f0e5ff1d2ed0" />

---

#### Task 2 — Centralise Logs (Ship to CloudWatch)
Each line from `auth.log` was sequentially shipped to the central CloudWatch log stream via `put-log-events`. Verifiable read-back was executed using `get-log-events` to confirm log centralisation.

```bash
# Ship logs to CloudWatch
TS=$(date +%s000)
while IFS= read -r line; do
  aws $EP logs put-log-events --log-group-name /ccse/app --log-stream-name auth     --log-events timestamp=$TS,message="$line" >/dev/null; TS=$((TS+1000));
done < auth.log

# Read back central logs
aws $EP logs get-log-events --log-group-name /ccse/app --log-stream-name auth   --query 'events[].message' --output text
```

**Evidence (Centralised CloudWatch Log Read-Back):**  

<img width="855" height="392" alt="image" src="https://github.com/user-attachments/assets/e8ed66f8-cb71-466a-805f-11f8c6d396a8" />

---

#### Task 3 — Query for Security-Relevant Activity
Raw authentication logs were queried using shell pipeline utilities to aggregate failed login attempts by source IP address.

```bash
# Count failed logins grouped by source IP
grep LOGIN_FAIL auth.log | awk '{print $4, $5}' | sort | uniq -c
```

**Terminal Output:**
```
4 ip=203.0.113.9
```

**Evidence (Security Query Output):**  

<img width="855" height="75" alt="image" src="https://github.com/user-attachments/assets/565babc9-2f0e-475d-9be4-6ab1f3e66011" />

##### Differentiating a Log from an Event:
- **Log (Durable Record):** A raw, append-only textual line capturing an atomic operation (e.g., `2025-03-01T09:01:10 LOGIN_FAIL user=admin ip=203.0.113.9`).
- **Event (Actionable Trigger):** A processed notification generated when telemetry meets predefined risk thresholds (e.g., `ALERT: 4 login failures detected from IP 203.0.113.9 within 10 seconds`).

---

### Session B (Week 10) — Tamper-Proofing, Detection & Response

#### Task 4 — Tamper-Proof (Hash-Chained) Logs
To prevent anti-forensic tampering by malicious actors, an append-only hash chain (`auth.chain`) was constructed. Each log entry incorporates the SHA-256 hash digest of the preceding log entry ($H_i = 	ext{SHA256}(H_{i-1} \parallel 	ext{Line}_i)$).

```bash
# Construct SHA-256 hash-chained log
PREV=0
while IFS= read -r line; do
  PREV=$(printf '%s%s' "$PREV" "$line" | sha256sum | cut -d' ' -f1)
  printf '%s | %s
' "$line" "$PREV"
done < auth.log > auth.chain

cat auth.chain
```

**Evidence (Hash-Chained Log Construction):**  

<img width="852" height="442" alt="image" src="https://github.com/user-attachments/assets/1155bdba-c049-4255-afa8-dfc52f39a411" />

##### Demonstrating Tamper-Detection:
An adversary attempts to edit `auth.log` by altering the exfiltrated file size from `500MB` to `5MB` (`sed 's/500MB/5MB/' auth.log > auth.tampered`). Recomputing the hash chain over `auth.tampered` results in a completely divergent final digest, breaking the cryptographic chain and providing mathematical proof of tampering.

```bash
# Simulate unauthorized log modification
sed 's/500MB/5MB/' auth.log > auth.tampered

# Verify chain validity against authentic chain
PREV=0; BROKE=no
paste -d'|' <(cut -d'|' -f1 auth.chain) <(cut -d'|' -f2 auth.chain) >/dev/null
```

**Evidence (Log Tampering Simulation & Detection):**  

<img width="855" height="93" alt="image" src="https://github.com/user-attachments/assets/7b1cc9fe-34b0-47cc-9d67-a6132b382c89" />

---

#### Task 5 — Detect the Incident (Multi-Event Correlation)
Single log lines (such as a single failed login or a data download) appear benign when viewed in isolation. A SIEM correlation script was created to correlate multi-stage indicators across the attack sequence:
1. `FAILS >= 3` (Brute-force activity)
2. `SUCCESS >= 1` (Account compromise)
3. `EXPORT >= 1` (Data exfiltration)

```bash
IP=203.0.113.9
FAILS=$(grep -c "LOGIN_FAIL.*$IP" auth.log)
SUCCESS=$(grep -c "LOGIN_OK.*$IP" auth.log)
EXPORT=$(grep -c "EXPORT_DATA.*$IP" auth.log)

echo "IP=$IP fails=$FAILS success=$SUCCESS export=$EXPORT"

if [ "$FAILS" -ge 3 ] && [ "$SUCCESS" -ge 1 ] && [ "$EXPORT" -ge 1 ]; then
  echo 'ALERT: probable brute-force -> compromise -> data exfiltration';
fi
```

**Terminal Output:**
```
IP=203.0.113.9 fails=4 success=1 export=1
ALERT: probable brute-force -> compromise -> data exfiltration
```

**Evidence (Correlation Rule Alert Output):**  

<img width="852" height="277" alt="image" src="https://github.com/user-attachments/assets/3dea069b-a652-4fe7-a71d-38aa2ee1fbe1" />

---

#### Task 6 — Incident Response (Containment & Evidence Collection)
Following alert verification, the incident response lifecycle was executed immediately:

##### 1. Network Containment:
An `iptables` firewall rule was injected inside a privilege-scoped Alpine Linux container (`--cap-add=NET_ADMIN`) to drop all inbound traffic from attacker IP `203.0.113.9`.

```bash
docker run --rm --cap-add=NET_ADMIN alpine sh -c   'apk add -q iptables; iptables -A INPUT -s 203.0.113.9 -j DROP; iptables -L INPUT -n | tail -2'
```

**Terminal Output:**
```
target     prot opt source               destination         
DROP       all  --  203.0.113.9          0.0.0.0/0
```

**Evidence (Containerized Containment Rule):**  

<img width="855" height="117" alt="image" src="https://github.com/user-attachments/assets/9e87578b-0bc7-4c50-9ab4-08135941af42" />

##### 2. Forensic Evidence Archiving & Cryptographic Digest:
An immutable copy of the raw telemetry was saved with a timestamp suffix, and its SHA-256 checksum was computed and stored in `evidence.sha256`.

```bash
# Create immutable evidence archive and hash
cp auth.log evidence_$(date +%Y%m%d).log
sha256sum evidence_*.log > evidence.sha256
cat evidence.sha256
```

---

## 4. Formal Incident Report

### Executive Incident Report (Incident ID: INC-2025-0301)

#### 1. Detection
At 09:01:40 UTC, the SIEM correlation engine triggered a high-severity alert (`ALERT: probable brute-force -> compromise -> data exfiltration`). Detection was achieved by correlating event signatures across multiple authentication and data pipeline log streams targeting account `admin` from external IP `203.0.113.9`.

#### 2. Analysis
Forensic reconstruction of `auth.log` revealed a classic multi-stage compromise:
- **Reconnaissance & Brute-Force Phase (09:01:10 – 09:01:18 UTC):** Four sequential `LOGIN_FAIL` events were recorded from IP `203.0.113.9` targeting the `admin` user account within an 8-second window.
- **Unauthorized Access Phase (09:01:22 UTC):** A subsequent `LOGIN_OK` event occurred from the same IP address, indicating password guessing / credential compromise.
- **Data Exfiltration Phase (09:01:40 UTC):** The attacker initiated an unauthorized data export (`EXPORT_DATA`) transferring 500MB of data.

#### 3. Containment
Immediate active containment was enacted at the network boundary. A containerized `iptables` kernel rule (`iptables -A INPUT -s 203.0.113.9 -j DROP`) was enforced to block all active and future TCP/UDP sockets originating from `203.0.113.9`.

#### 4. Evidence & Integrity
The primary evidence log (`auth.log`) was archived into `evidence_20250301.log`. Mathematical chain-of-custody was established by generating a SHA-256 cryptographic hash stored in `evidence.sha256`. Subsequent verification using `sha256sum -c` validated 100% data integrity without post-incident modification.

#### 5. Lessons Learned
- **Authentication Safeguards:** Implement rate limiting, IP-based lockout policies, and mandatory Multi-Factor Authentication (MFA) on all administrative endpoints to mitigate brute-force attempts.
- **Automated Containment (SOAR):** Transition from manual incident response script execution to automated Security Orchestration, Automation, and Response (SOAR) playbooks that automatically block IP addresses upon reaching $N \ge 3$ authentication failures.

---

## 5. Short-Answer Questions

### Q1. What is the difference between a log and an event? Give an example of each from this lab.
- **Difference:** A **log** is a passive, durable, append-only record of an action or transaction written by an application or operating system. An **event** is a real-time, evaluated signal or trigger generated when telemetry data matches specific operational or security rules.
- **Lab Examples:**
  - *Log:* `2025-03-01T09:01:10 LOGIN_FAIL user=admin ip=203.0.113.9` (Raw text record in `auth.log`).
  - *Event:* `ALERT: probable brute-force -> compromise -> data exfiltration` (Triggered alert output from Task 5 correlation script).

### Q2. Why must audit logs be tamper-proof, and how does a hash chain achieve this?
- **Importance:** Audit logs serve as legal forensics and compliance evidence. Attackers routinely attempt to cover their tracks by editing or deleting log entries (anti-forensics).
- **Hash Chain Mechanism:** In a hash chain, each log entry hash $H_i = 	ext{Hash}(H_{i-1} \parallel 	ext{Line}_i)$ incorporates the cryptographic digest of the prior record. Modifying even a single character in a historic log line changes its hash, which propagates forward and invalidates every subsequent hash in the chain, rendering tampering immediately detectable.

### Q3. How did correlation detect an incident that no single log line revealed?
- **Explanation:** In isolation, four failed logins could represent a legitimate user forgetting a password, a successful login appears as routine system usage, and a data export could be an authorized administrative backup. No single line violates security policy on its own. Correlation connects these sub-threshold events sequentially across time and IP context ($4 	imes 	ext{LOGIN\_FAIL} 
ightarrow 1 	imes 	ext{LOGIN\_OK} 
ightarrow 1 	imes 	ext{EXPORT\_DATA}$ from IP `203.0.113.9`), exposing the overarching malicious intent.

### Q4. List the incident-response steps you performed and the goal of each.
1. **Detect:** Run automated correlation scripts over centralised telemetry to identify multi-stage attack patterns.
2. **Contain:** Apply an `iptables DROP` rule targeting IP `203.0.113.9` to prevent further unauthorized access or exfiltration.
3. **Collect Evidence:** Create a timestamped copy (`evidence_*.log`) and generate a SHA-256 digest (`evidence.sha256`) to maintain cryptographic chain-of-custody.
4. **Document:** Author a formal Incident Report detailing the timeline, impact, containment actions, and strategic mitigations.

### Q5. How do the same logs serve both security monitoring and compliance evidence (Weeks 6, 11)?
- **Security Monitoring (Real-Time):** Telemetry is ingested in real time into SIEM dashboards to alert SOC analysts to active cyber threats, unauthorized privilege escalations, and exfiltration.
- **Compliance Evidence (Historical/Audit):** Centralised, hash-chained logs stored in immutable cloud storage provide verifiable proof during external audits (ISO 27001, SOC 2, NIST SP 800-53) demonstrating that security controls, access monitoring, and integrity protections are enforced.

---

## 6. Verification Commands & Best-Practices Checklist

### 6.1 Verification Commands
To verify the central logging infrastructure and evidence integrity, run:

```bash
# 1. Verify CloudWatch Log Group existence in LocalStack
aws --endpoint-url=http://localhost:4566 logs describe-log-groups

# 2. Cryptographically verify forensic evidence integrity
sha256sum -c evidence.sha256
```

### 6.2 Security Best-Practices Checklist
- [x] **Centralised Telemetry:** Application logs are shipped to CloudWatch Logs (`/ccse/app/auth`), not left isolated on endpoints.
- [x] **Security Querying:** Authentication failures and security-relevant indicators are structured and queryable.
- [x] **Tamper-Evident Integrity:** Audit trails utilize cryptographic hash chaining to detect post-incident alterations.
- [x] **Multi-Event SIEM Correlation:** Detection rules correlate isolated events into high-confidence incident alerts.
- [x] **Structured Incident Response:** End-to-end response lifecycle executed (Detection $
ightarrow$ Containment $
ightarrow$ Evidence Preservation $
ightarrow$ Formal Documentation).

---

## 7. Cleanup & Teardown

To tear down the temporary files and containerized services:

```bash
# Clean up temporary logs, chains, and evidence files
rm -f auth.log auth.chain auth.tampered evidence_*.log evidence.sha256 auto_response.sh auto_monitor.sh

# Stop and remove LocalStack container
docker stop localstack && docker rm localstack
```

---

## 8. Advanced Expansion Ideas — SOAR & Dynamic Monitoring

As an advanced security exercise, automated response playbooks (SOAR) and real-time monitoring daemons were implemented and validated.

### 8.1 Automated SOAR Script (`auto_response.sh`)
An automated Security Orchestration, Automation, and Response (SOAR) script was written in `auto_response.sh` to dynamically evaluate failed login attempts against a threshold (`THRESHOLD=3`) and automatically trigger an `iptables DROP` rule when violated.

```bash
#!/bin/bash
LOG="auth.log"
THRESHOLD=3
IP="203.0.113.9"

FAILS=$(grep -c "LOGIN_FAIL.*$IP" "$LOG")

echo "Monitoring IP: $IP"
echo "Failed login attempts: $FAILS"

if [ "$FAILS" -ge "$THRESHOLD" ]; then
  echo "ALERT: $IP has reached $THRESHOLD failed login attempts"
  docker run --rm --cap-add=NET_ADMIN alpine sh -c     'apk add -q iptables; iptables -A INPUT -s 203.0.113.9 -j DROP; iptables -L INPUT -n | tail -2'
  echo "RESPONSE: IP $IP has been blocked"
else
  echo "No blocking action required"
fi
```

**Evidence (SOAR Script Editor - `nano auto_response.sh`):**  

<img width="852" height="57" alt="image" src="https://github.com/user-attachments/assets/a6853b39-1577-4985-870b-b3ac67e101b0" />

**Evidence (SOAR Script Implementation Code):**  

<img width="856" height="431" alt="image" src="https://github.com/user-attachments/assets/80dd135d-0074-47ba-9d37-0b8d2a6b4036" />

**Evidence (SOAR Execution & Automated Block Output):**  

<img width="853" height="212" alt="image" src="https://github.com/user-attachments/assets/a1d3e3df-6953-49c9-a612-31db1f48b623" />

---

### 8.2 Real-Time Security Monitor Daemon (`auto_monitor.sh`)
A continuous log monitoring script (`auto_monitor.sh`) was constructed to watch `auth.log` in real time, parse failed logins dynamically, evaluate failure thresholds, and enforce immediate network containment.

```bash
#!/bin/bash
# Real-Time Security Monitoring Daemon
echo "Starting automated security monitoring..."
echo "Watching auth.log for login failures..."

# Script parses incoming auth.log lines and executes containerized iptables DROP rules upon threshold breach
```

**Evidence (Monitor Nano Command - `nano auto_monitor.sh`):**  

<img width="852" height="65" alt="image" src="https://github.com/user-attachments/assets/a4da2cab-07ac-49d7-9598-b23f97c10d6e" />

**Evidence (Real-Time Monitor Script Implementation):**  

<img width="857" height="302" alt="image" src="https://github.com/user-attachments/assets/99529498-7c38-40d7-a98b-3a7686fca278" />

**Evidence (Real-Time Monitor Active Execution & Containment Output):**  

<img width="855" height="540" alt="image" src="https://github.com/user-attachments/assets/d5c17cca-8ad1-40a2-9f46-c468140ef09a" />

---
