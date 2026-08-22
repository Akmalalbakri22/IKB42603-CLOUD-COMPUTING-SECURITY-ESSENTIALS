# Lab 3: Data Protection: Encryption & Key Management

**Course**: IKB42603 Cloud Computing Security Essentials  

**Institution**: Universiti Kuala Lumpur Malaysian Institute of Information Technology (UniKL MIIT)

**Name**: Muhammad Akmal Irfan Albakri Bin Ikmal Hisham

**Student ID:** 52215124003

**Group:** L02-B04


**Lab Focus**: At-Rest & In-Transit Encryption, Envelope Encryption, Cryptographic Erasure & Integrity (OpenSSL & LocalStack KMS)  

---

## Executive Summary & Learning Outcomes

This report documents the step-by-step implementation, empirical execution, and theoretical analysis for **Lab 3: Data Protection: Encryption & Key Management**. The primary objective is to evaluate data protection mechanisms across cloud environments at rest, in transit, and during key lifecycle transitions.

### Key Competencies Demonstrated
1. **Symmetric & Asymmetric Cryptography**: Encrypting sensitive data at rest using AES-256-CBC with PBKDF2 salt, and performing RSA-2048 keypair generation, public key encryption, and digital signatures.
2. **Data in Transit Protection**: Provisioning X.509 certificates and serving secure HTTPS traffic via Nginx in Docker to mitigate eavesdropping.
3. **Key Management & Envelope Encryption**: Utilizing LocalStack AWS KMS to create Customer Master Keys (CMKs), generate data key pairs, encrypt payloads locally with plaintext data keys, and destroy unencrypted key material.
4. **Per-Tenant Separation & Cryptographic Erasure**: Implementing multi-tenant key isolation, key deletion scheduling/disabling, and verifying provable deletion (crypto-shredding).
5. **Data Integrity & Tamper-Evidence**: Hashing sensitive records with SHA-256, detecting bit-level modifications, and building a tamper-evident log via hash chaining.

---

## Technical Environment & Prerequisites

| Requirement | Implementation Detail | Status / Notes |
| :--- | :--- | :--- |
| **Operating System** | Linux / Kali Linux (`anonym22@kali`) | Shell execution environment |
| **Containers** | Docker Engine & Docker CLI | Used for TLS Nginx & LocalStack KMS |
| **Cryptography Tools** | OpenSSL v3.0+ | `enc`, `genrsa`, `rsa`, `pkeyutl`, `req`, `dgst` |
| **AWS CLI** | AWS CLI v2 | Pointed to LocalStack (`--endpoint-url=http://localhost:4566`) |
| **KMS Emulator** | LocalStack KMS Engine | Emulates AWS KMS master keys & data key APIs |

---

# Session A (Week 5) — Encryption Fundamentals

---

## Task 1 — Symmetric Encryption (Data at Rest)

### Objective
Create a sensitive patient record, encrypt it using AES-256-CBC with PBKDF2 key derivation, inspect the raw ciphertext, decrypt it back, and verify string match integrity.

### Step-by-Step Execution

#### Step 1.1: Create Sample Sensitive File
```bash
echo 'Patient: Badrol, Diagnosis: confidential' > record.txt
```

#### Step 1.2: Symmetric AES-256 Encryption
Encrypt `record.txt` into `record.enc` using AES-256 in Cipher Block Chaining (CBC) mode with Password-Based Key Derivation Function 2 (PBKDF2) and salt:
```bash
openssl enc -aes-256-cbc -pbkdf2 -salt -in record.txt -out record.enc
```
*Prompt*: Enter AES-256-CBC encryption password.

#### Step 1.3: Inspect Ciphertext
```bash
cat record.enc
```
*Observation*: The file contents are unreadable binary garbage prefixed with `Salted__`, proving that plaintext data at rest is protected against direct read access.

#### Step 1.4: Decrypt File & Verify Match
```bash
openssl enc -d -aes-256-cbc -pbkdf2 -in record.enc -out record.dec.txt
diff record.txt record.dec.txt && echo 'MATCH: decryption successful'
```
*Output*: `MATCH: decryption successful`

### Visual Evidence (Task 1)

<img width="624" height="416" alt="image" src="https://github.com/user-attachments/assets/0f059a26-04ee-43b4-b563-b3beab6f965a" />


> [!NOTE]
> **Key Distribution Problem in Symmetric Encryption**:  
> Symmetric encryption relies on a single shared key for both encryption and decryption. In cloud environments, transmitting or sharing this single key between distributed microservices, application servers, and multi-tenant clients creates a critical vulnerability. If the key is intercepted in transit or compromised at either endpoint, all historical and future encrypted data is exposed. Thus, symmetric encryption requires out-of-band secure key management (such as asymmetric key exchange or KMS).

---

## Task 2 — Asymmetric Encryption & Digital Signatures

### Objective
Generate an RSA-2048 keypair. Demonstrate confidentiality by encrypting with the public key and decrypting with the private key. Demonstrate integrity and non-repudiation by signing with the private key and verifying with the public key.

### Step-by-Step Execution

#### Step 2.1: Generate RSA 2048-Bit Key Pair
```bash
openssl genrsa -out private.pem 2048
openssl rsa -in private.pem -pubout -out public.pem
```

#### Step 2.2: Asymmetric Confidentiality (Encrypt with Public Key, Decrypt with Private Key)
```bash
# Encrypt with Public Key
openssl pkeyutl -encrypt -pubin -inkey public.pem -in record.txt -out record.rsa

# Decrypt with Private Key
openssl pkeyutl -decrypt -inkey private.pem -in record.rsa -out record.rsa.txt
```

#### Step 2.3: Digital Signature & Verification (Sign with Private Key, Verify with Public Key)
```bash
# Sign digest with Private Key
openssl dgst -sha256 -sign private.pem -out record.sig record.txt

# Verify signature with Public Key
openssl dgst -sha256 -verify public.pem -signature record.sig record.txt
```
*Output*: `Verified OK`

### Visual Evidence (Task 2)

<img width="624" height="287" alt="image" src="https://github.com/user-attachments/assets/8b3fa553-4fda-4fed-893b-e1f8e7a90dbe" />


> [!IMPORTANT]
> **Role Reversal in Public Key Cryptography**:  
> - **Encryption**: Uses the **Public Key** to lock data (anyone can encrypt), and the **Private Key** to unlock data (only the key owner can decrypt).  
> - **Digital Signature**: Uses the **Private Key** to sign a digest (only the key owner can produce the signature), and the **Public Key** to verify (anyone can confirm authenticity and data integrity).

---

## Task 3 — Encryption in Transit (TLS)

### Objective
Generate a self-signed X.509 TLS certificate, spin up an Nginx container serving HTTPS on port `8443`, and fetch the sensitive file securely over TLS using `curl`.

### Step-by-Step Execution

#### Step 3.1: Generate Self-Signed Certificate
```bash
openssl req -x509 -newkey rsa:2048 -keyout key.pem -out cert.pem \
 -days 7 -nodes -subj '/CN=localhost'
```

#### Step 3.2: Launch Nginx HTTPS Container
```bash
docker run --rm -d --name tls -p 8443:443 \
 -v $(pwd)/cert.pem:/etc/nginx/cert.pem -v $(pwd)/key.pem:/etc/nginx/key.pem \
 -v $(pwd)/record.txt:/usr/share/nginx/html/record.txt nginx
```
*Output*: Container ID (e.g. `338...`).

#### Step 3.3: Verify HTTPS Channel over TLS
```bash
curl -k https://localhost:8443/record.txt
```
*Output*: `Patient: Badrol, Diagnosis: confidential`

### Visual Evidence (Task 3)

<img width="624" height="377" alt="image" src="https://github.com/user-attachments/assets/8438e40c-d1a6-4d50-8901-f4adddd30317" />


*Figure 3.1: X.509 Certificate & Private Key Creation*

<img width="852" height="108" alt="image" src="https://github.com/user-attachments/assets/a80f4f19-fdbb-4c89-a638-af6b46afa60a" />



*Figure 3.2: Docker Container Execution for Port 8443*


<img width="856" height="100" alt="image" src="https://github.com/user-attachments/assets/2d568890-8537-498c-831c-b6612be57085" />


*Figure 3.3: Fetching Encrypted Record over TLS*

> [!TIP]
> **Security Comparison (HTTP vs. HTTPS/TLS)**:  
> Plain HTTP transmits network packets unencrypted as clear text. Any adversary on the network path (e.g., rogue Wi-Fi access points, compromised routers, or ISP tap) can perform packet sniffing to read sensitive data. Transport Layer Security (TLS) encapsulates HTTP within an encrypted channel using symmetric session keys established via asymmetric handshakes, rendering intercepted network traffic unreadable.

---

# Session B (Week 6) — Key Management, Envelope Encryption & Erasure

---

## Task 4 — Create and Use a KMS Master Key

### Objective
Configure AWS CLI to communicate with LocalStack KMS (`http://localhost:4566`), create a Customer Master Key (CMK) for Tenant A, and perform direct encryption of small secret data.

### Step-by-Step Execution

#### Step 4.1: Define LocalStack Endpoint & Create Master Key (KEY_A)
```bash
EP='--endpoint-url=http://localhost:4566'

# Create Customer Master Key for Tenant A
aws $EP kms create-key --description 'CCSE tenant-A master key'
```
*Output JSON*:
```json
{
  "KeyMetadata": {
    "AWSAccountId": "000000000000",
    "KeyId": "KEY_ID_A",
    "Arn": "arn:aws:kms:us-east-1:000000000000:key/KEY_ID_A",
    "CreationDate": "2026-08-22T06:58:47.497358-07:00",
    "Enabled": true,
    "Description": "CCSE tenant-A master key",
    "KeyUsage": "ENCRYPT_DECRYPT",
    "KeyState": "Enabled",
    "Origin": "AWS_KMS",
    "KeyManager": "CUSTOMER",
    "CustomerMasterKeySpec": "SYMMETRIC_DEFAULT",
    "KeySpec": "SYMMETRIC_DEFAULT",
    "EncryptionAlgorithms": ["SYMMETRIC_DEFAULT"]
  }
}
```

#### Step 4.2: Store Key ID & Encrypt Small Secret
```bash
KEY_A="KEY_ID_A"

aws $EP kms encrypt --key-id $KEY_A --plaintext "$(echo -n 'hello' | base64)" \
 --query CiphertextBlob --output text
```
*Output*: Encrypted Base64 string representing the `CiphertextBlob`.

### Visual Evidence (Task 4)

<img width="855" height="407" alt="image" src="https://github.com/user-attachments/assets/2067c49e-b9b7-495c-997b-99aa1d32550c" />

*Figure 4.1: Creation of Tenant A CMK in KMS*

<img width="855" height="203" alt="image" src="https://github.com/user-attachments/assets/44818d2f-2f51-4c9e-a114-c35500152a83" />


*Figure 4.2: Direct KMS Data Encryption*

---

## Task 5 — Envelope Encryption

### Objective
Demonstrate envelope encryption for bulk payload protection. Request a data key pair from KMS, use the plaintext data key to encrypt `record.txt` locally, destroy the plaintext data key from local storage, and preserve only the KMS-encrypted data key (`datakey.enc`).

### Step-by-Step Execution

#### Step 5.1: Request Data Key Pair from KMS
```bash
aws $EP kms generate-data-key --key-id "$KEY_A" --key-spec AES_256 \
 --query '[Plaintext,CiphertextBlob]' --output text > key_output.txt

# Extract Column 1 (Plaintext Data Key b64) and Column 2 (Wrapped Data Key b64)
awk '{print $1}' key_output.txt > datakey.b64
awk '{print $2}' key_output.txt > datakey.enc
```

#### Step 5.2: Encrypt Local Data with Plaintext Data Key
```bash
# Decode base64 plaintext key into raw binary key
base64 -d datakey.b64 > datakey.bin

# Encrypt local payload with plaintext binary key
openssl enc -aes-256-cbc -pbkdf2 -in record.txt -out record.env.enc \
 -pass file:./datakey.bin
```

#### Step 5.3: Destroy Plaintext Data Key Material
```bash
# Delete unencrypted data key from local storage
rm datakey.bin datakey.b64

# Verify remaining key files
ls -l datakey*
```
*Output*: Only `datakey.enc` remains.

### Visual Evidence (Task 5)

<img width="852" height="172" alt="image" src="https://github.com/user-attachments/assets/9870ad82-3233-4083-834a-658b2fdf699d" />

*Figure 5.1: KMS Data Key Generation Output*

<img width="855" height="257" alt="image" src="https://github.com/user-attachments/assets/b010b812-a167-45e0-b1ab-7b6f80d193ab" />

*Figure 5.2: Splitting Plaintext (datakey.b64) and Wrapped (datakey.enc) Keys*

<img width="855" height="63" alt="image" src="https://github.com/user-attachments/assets/65296db5-49f3-46da-8e46-b716a32c6a85" />

*Figure 5.3: OpenSSL Payload Encryption with datakey.bin*

<img width="855" height="245" alt="image" src="https://github.com/user-attachments/assets/28d36b19-ef30-47c1-9560-be3fec0d8d45" />

 *Figure 5.4: Disk Cleanup Verification Leaving Only datakey.enc*

> [!NOTE]
> **Workflow of Envelope Encryption**:  
> 1. **Data Key Request**: Client asks KMS for a Data Encryption Key (DEK). KMS generates a new symmetric key and returns both a **Plaintext DEK** and a **Ciphertext DEK** (wrapped by CMK).  
> 2. **Local Encryption**: Client encrypts the large file locally using the **Plaintext DEK**.  
> 3. **Purge Plaintext DEK**: Client immediately deletes the **Plaintext DEK** from disk/memory.  
> 4. **Storage**: The encrypted file (`record.env.enc`) is stored alongside the **Wrapped DEK** (`datakey.enc`).  
> 5. **Decryption**: To read the file later, the client sends `datakey.enc` to KMS (`kms decrypt`). KMS decrypts the DEK using the CMK and returns the plaintext key to memory, allowing payload decryption.

---

## Task 6 — Per-Tenant Keys & Cryptographic Erasure

### Objective
Create a separate master key for Tenant B (`KEY_B`). Simulate cryptographic erasure (crypto-shredding) of Tenant A by scheduling deletion and disabling Tenant A's CMK (`KEY_A`), then verifying that data decryption attempts fail deterministically.

### Step-by-Step Execution

#### Step 6.1: Create CMK for Tenant B
```bash
aws $EP kms create-key --description 'CCSE tenant-B master key'
```
*Output KeyId*: `KEY_B="df56c... "`

```bash
KEY_B="df56c802-53d7-4c7b-94df-7f9a888c3a91"
```

#### Step 6.2: Schedule Key Deletion for Tenant A
```bash
aws $EP kms schedule-key-deletion --key-id $KEY_A --pending-window-in-days 7
```
*Output JSON*: `KeyState: "PendingDeletion"`, `DeletionDate: "2026-08-29T07:30:13-07:00"`.

#### Step 6.3: Disable Key A Immediately
```bash
aws $EP kms disable-key --key-id $KEY_A
```
*Output Error*: `aws: [ERROR]: An error occurred (KMSInvalidStateException) when calling the DisableKey operation: ... is pending deletion.`

#### Step 6.4: Attempt Decryption of Tenant A Data Key (Crypto-Erasure Verification)
```bash
aws $EP kms decrypt --ciphertext-blob fileb://datakey.enc 2>&1 | head -3
```
*Output Error*:
```text
aws: [ERROR]: An error occurred (NotFoundException) when calling the Decrypt operation: Invalid keyId '0e599e69-0268-45d4-8dfd-0be3c2004245'
```

### Visual Evidence (Task 6)

<img width="855" height="412" alt="image" src="https://github.com/user-attachments/assets/8dd18ff1-14ab-4865-a86c-4b7c5ee55b4b" />

*Figure 6.1: Tenant B CMK Generation*

<img width="855" height="75" alt="image" src="https://github.com/user-attachments/assets/46f72d97-034a-4a09-8145-270356b7d487" />

*Figure 6.2: Environment Variable Assignment for KEY_B*

<img width="855" height="147" alt="image" src="https://github.com/user-attachments/assets/04b12723-64e1-4588-b156-cc27a7decd75" />

*Figure 6.3: Scheduling Tenant A CMK Deletion in KMS*

<img width="856" height="102" alt="image" src="https://github.com/user-attachments/assets/7b378c2a-bd0d-4cb0-a6ad-f75bb5021331" />

*Figure 6.4: State Conflict on Disable Key while Pending Deletion*

<img width="853" height="100" alt="image" src="https://github.com/user-attachments/assets/59876f42-2380-49c5-8fde-1e38ab97575b" />

*Figure 6.5: Deterministic Decryption Failure after Key Deletion*

> [!CAUTION]
> **Provable Deletion via Cryptographic Erasure**:  
> In multi-tenant cloud storage, physical media cannot be individually sanitized or degaussed without destroying shared infrastructure. Cryptographic erasure (crypto-shredding) guarantees that by destroying the tenant-specific Customer Master Key (CMK), all data keys wrapped by that CMK become permanently un-decryptable. The underlying encrypted data blocks on disk are rendered mathematically indistinguishable from random noise, achieving compliance-grade deletion instantly.

---

## Task 7 — Integrity & Tamper-Evidence

### Objective
Generate a baseline SHA-256 digest of `record.txt`, tamper with a file copy (`tampered.txt`), observe hash divergence, and construct a hash chain script where each entry incorporates the hash of the preceding record.

### Step-by-Step Execution

#### Step 7.1: Baseline SHA-256 Fingerprint
```bash
sha256sum record.txt
```
*Output*: `...fa7fc0062ad9bb3cd0  record.txt`

#### Step 7.2: Tamper with Data Copy & Verify Hash Change
```bash
cp record.txt tampered.txt; echo 'x' >> tampered.txt
sha256sum record.txt tampered.txt
```
*Output*:
```text
...fa7fc0062ad9bb3cd0  record.txt
1468b6da...            tampered.txt
```

#### Step 7.3: Implement Hash Chain Log
```bash
PREV=0
for line in 'login ok' 'file read' 'export data'; do \
 PREV=$(echo -n "$PREV$line" | sha256sum | cut -d' ' -f1); \
 echo "$line | $PREV"; done
```
*Output*:
```text
login ok | ...c91d90929a820053
file read | 6c3adc6...
export data | ...c503431b32da68
```

### Visual Evidence (Task 7)

<img width="857" height="92" alt="image" src="https://github.com/user-attachments/assets/d4932b02-5001-424c-9911-1377a496d9be" />

*Figure 7.1: Baseline SHA-256 Hash of record.txt*

<img width="853" height="72" alt="image" src="https://github.com/user-attachments/assets/8986079e-6d1a-43d2-bba8-ba63735fb79f" />

*Figure 7.2: Detecting Modification via Hash Comparison*

<img width="855" height="101" alt="image" src="https://github.com/user-attachments/assets/46579bbe-303d-41f2-8ead-6b4b6d513ef8" />

*Figure 7.3: Sequential Hash Chaining Output*


## 2. Short-Answer Assessment Questions

### Q1. Compare symmetric and asymmetric encryption: speed, key distribution, and typical use.

| Criteria | Symmetric Encryption (e.g., AES-256) | Asymmetric Encryption (e.g., RSA-2048) |
| :--- | :--- | :--- |
| **Speed & Computational Overhead** | Extremely fast; minimal CPU overhead due to hardware acceleration (AES-NI). Suitable for bulk data payload encryption. | Orders of magnitude slower; heavy modular exponentiation math. Restricted to small data sizes (e.g., key transport). |
| **Key Distribution** | **High difficulty**: Both parties must share the exact same private key prior to communication without interception. | **Low difficulty**: Public keys can be distributed open to the world; private keys remain secret with the owner. |
| **Typical Use Cases** | Encrypting bulk storage at rest (disk volumes, databases, object stores) and symmetric payload transport. | Digital signatures, identity authentication, key exchange (Diffie-Hellman / RSA in TLS handshakes), envelope encryption key wrapping. |

---

### Q2. Why is key management described as the weakest link, not the algorithm?
Modern cryptographic algorithms such as AES-256 and RSA-2048 are mathematically robust against brute-force attacks given current computational capabilities. However, security fails in practice due to poor **key management lifecycle controls**:
1. **Insecure Storage**: Hardcoding keys in source repositories, leaving unencrypted keys in temporary files, or storing keys on disk alongside encrypted data.
2. **Access Control Failures**: Over-privileged IAM roles permitting unauthorized principals to invoke `kms:Decrypt`.
3. **Lack of Key Rotation**: Using static keys for prolonged periods increases exposure windows to cryptanalysis or side-channel attacks.
4. **Key Compromise**: If an attacker gains access to a key management service or memory dump containing unencrypted key material, all underlying algorithms become irrelevant.

---

### Q3. Explain envelope encryption and why only the master key needs hardware-grade protection.
**Envelope Encryption** is a hierarchical key management design where data is encrypted locally using a unique **Data Encryption Key (DEK)**, and the DEK itself is encrypted (wrapped) using a **Customer Master Key (CMK)** managed inside a Key Management Service (KMS).

- **Why only the CMK needs hardware-grade protection**:
  - Bulk data encryption using a centralized Hardware Security Module (HSM) causes high network latency and performance bottlenecks.
  - By using DEKs for local bulk encryption, high-throughput payload encryption occurs locally at memory speed.
  - The CMK inside the HSM is only invoked to wrap or unwrap the small (32-byte) DEKs.
  - As a result, hardware protection (FIPS 140-2/140-3 HSMs) is reserved for the root/master key (`CMK`), providing maximum security at scale.

---

### Q4. How does cryptographic erasure achieve provable deletion where overwriting cannot (in the cloud)?
In cloud multi-tenant architectures, physical disks are abstracted, virtualized, replicated across Availability Zones, and shared across tenants. Standard overwrite techniques (such as `shred` or DoD 5220.22-M zeroization) are infeasible because:
1. Storage hardware is shared and physically inaccessible to tenants.
2. Flash storage controllers (SSDs) use wear leveling, bad-block remapping, and copy-on-write snapshots that obscure physical block addresses.

**Cryptographic Erasure (Crypto-Shredding)** achieves provable deletion by destroying the specific Customer Master Key (CMK) assigned to a tenant or object. Without the CMK, unwrapping the Data Encryption Key (DEK) is mathematically impossible. The remaining encrypted data blocks become indistinguishable from random entropy, ensuring instant, provable, and compliance-ready data destruction across all backups and replicas.

---

### Q5. How does a hash chain make a log tamper-evident (link to tamper-proof logs)?
A **Hash Chain** constructs a cryptographic dependency graph where record $N$ incorporates the hash output of record $N-1$:

$$\text{Hash}_N = \text{SHA-256}(\text{Hash}_{N-1} \parallel \text{LogEntry}_N)$$

- **Tamper-Evidence**: If an attacker modifies an earlier log entry (e.g., altering `LogEntry_1`), its hash changes. Consequently, all downstream hashes ($\text{Hash}_2, \text{Hash}_3, \dots$) invalidate completely.
- **Tamper-Proof Audit Trails**: When combined with append-only storage or digital signatures per block, any modification, insertion, or deletion of log entries is immediately detected during automated validation routines.

---

## 3. Verification Command Output

The system environment was validated using the lab verification suite:

```bash
# 1. Verify KMS Active Keys
aws --endpoint-url=http://localhost:4566 kms list-keys

# 2. Verify RSA Digital Signature
openssl dgst -sha256 -verify public.pem -signature record.sig record.txt
```

*Verification Log*:
```text
Verified OK
```

---

## 4. Security Best-Practices Checklist

- [x] **Data encrypted at rest (AES)**: AES-256-CBC PBKDF2 encryption verified with `diff` match.
- [x] **Asymmetric keys used correctly**: Public key used for encryption, private key used for digital signatures.
- [x] **Data protected in transit with TLS**: Nginx HTTPS service operational over port 8443 verified with `curl -k`.
- [x] **Envelope encryption used**: Plaintext data keys destroyed from local disk; only wrapped `datakey.enc` stored.
- [x] **Per-tenant keys & cryptographic erasure**: Isolated master keys per tenant; key deletion/disabling demonstrated with deterministic decryption failure.
- [x] **Integrity verified**: Baseline hash fingerprinting, tamper detection, and hash-chain audit logging executed.

---

## 5. Cleanup & Teardown Commands

To restore the host environment to a clean baseline state, execute the following teardown commands:

```bash
# Stop and remove TLS Nginx container
docker stop tls 2>/dev/null

# Clean up local key and ciphertext artifacts
rm -f record.* private.pem public.pem key.pem cert.pem datakey.* tampered.txt key_output.txt

# Stop LocalStack container (if applicable)
docker stop localstack 2>/dev/null && docker rm localstack 2>/dev/null
```

---

## References

1. UniKL MIIT — *IKB42603 Cloud Computing Security Essentials Lab Manual (Lab 3)*, Prof. Dr. Shahrulniza Musa.
2. OpenSSL Project — *OpenSSL Cryptographic Tool Documentation*, [openssl.org/docs](https://www.openssl.org/docs/).
3. AWS Documentation — *AWS Key Management Service (KMS) Developer Guide*, [docs.aws.amazon.com/kms](https://docs.aws.amazon.com/kms/).
4. Cloud Security Alliance — *CSA Security Guidance v5: Data Security & Encryption*.
