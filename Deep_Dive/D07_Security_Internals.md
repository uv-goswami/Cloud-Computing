# D07 — Security Internals
**Track: Deep Dive | IAM evaluation engine, KMS, TLS handshake**

---

## 1. IAM Policy Evaluation Engine — Complete Logic

```mermaid
flowchart TD
    Request["API Request:<br/>Principal: arn:aws:iam::123:user/alice<br/>Action: s3:GetObject<br/>Resource: arn:aws:s3:::bucket/file.txt"]

    Step1["1. Is there an explicit DENY<br/>in any applicable policy?"]
    Step1 -->|Yes| Deny["DENY — Access Blocked<br/>Explicit DENY cannot be overridden"]
    Step1 -->|No| Step2

    Step2["2. Is there an explicit ALLOW<br/>in an identity-based policy?"]
    Step2 -->|No| Step3
    Step2 -->|Yes| Step4

    Step3["3. Is there an explicit ALLOW<br/>in a resource-based policy?<br/>(S3 bucket policy, KMS key policy)"]
    Step3 -->|No| ImplicitDeny["IMPLICIT DENY<br/>Default: deny all"]
    Step3 -->|Yes| Allow["ALLOW — Access Granted"]

    Step4["4. Does a permission boundary<br/>also allow this action?"]
    Step4 -->|No| Deny2["DENY — Boundary blocks it"]
    Step4 -->|Yes| Allow
```

**Policy types evaluated (in order):**
1. SCP (Service Control Policy) — AWS Organizations limit
2. Permission Boundary — limits max permissions for IAM entity
3. Session Policy — temporary credentials scope
4. Identity Policy — user/role/group policies
5. Resource Policy — S3 bucket policy, KMS key policy

---

## 2. AWS KMS — How Encryption Keys Are Protected

```mermaid
graph TB
    subgraph KMS_Architecture["AWS KMS Architecture"]
        API["KMS API<br/>Encrypt / Decrypt / GenerateDataKey"]

        subgraph HSM_Cluster["HSM Cluster (Hardware Security Modules)"]
            HSM1["HSM 1 (AZ-a)<br/>FIPS 140-2 Level 3<br/>CMK never leaves HSM<br/>Tamper-evident hardware"]
            HSM2["HSM 2 (AZ-b)<br/>Synchronous replication"]
            HSM3["HSM 3 (AZ-c)"]
        end

        Domain_State["Domain State<br/>Encrypted with HSM cluster key<br/>Loaded at HSM init"]
    end

    App["Your Application"] -->|"Encrypt(plaintext, keyId)"| API
    API --> HSM1
    HSM1 --> API
    API -->|"Ciphertext (CMK never exposed)"| App
```

**Envelope encryption (how S3 SSE-KMS works):**

```
1. S3 calls KMS: GenerateDataKey(CMK_id)
2. KMS returns: (plaintext_data_key, encrypted_data_key)
   - plaintext_data_key: random 256-bit AES key in memory
   - encrypted_data_key: plaintext_data_key encrypted with your CMK (stored in S3 metadata)
3. S3 encrypts the object with plaintext_data_key (fast AES encryption)
4. S3 discards plaintext_data_key from memory
5. Stores encrypted_data_key alongside the object

To decrypt:
1. S3 retrieves encrypted_data_key from metadata
2. S3 calls KMS: Decrypt(encrypted_data_key) — requires access to CMK
3. KMS returns plaintext_data_key (only if IAM allows)
4. S3 decrypts object, discards plaintext_data_key
```

**Why envelope encryption?** KMS only encrypts small payloads (up to 4KB). Encrypting a 5GB object directly in KMS is impossible and would be enormously expensive. The data key handles the bulk data; KMS only handles the small key.

---

## 3. TLS Handshake — What HTTPS Actually Does

```mermaid
sequenceDiagram
    participant Browser
    participant Server as AWS ALB / EC2

    Browser->>Server: ClientHello<br/>TLS version: 1.3<br/>Cipher suites: [AES-256-GCM, ChaCha20]<br/>Random nonce: client_random

    Server->>Browser: ServerHello<br/>Selected cipher: AES-256-GCM<br/>Random nonce: server_random<br/>Certificate: [domain.com, public_key, signed_by_CA]

    Browser->>Browser: Verify certificate:<br/>1. Signed by trusted CA?<br/>2. Domain matches?<br/>3. Not expired?<br/>4. Not revoked? (OCSP)

    Note over Browser,Server: TLS 1.3: Key exchange integrated into handshake
    Browser->>Server: Key Exchange<br/>Derive session keys from:<br/>client_random + server_random + DH key exchange<br/>Both sides compute same session key independently

    Browser->>Server: Application Data (encrypted with session key)
    Server->>Browser: Application Data (encrypted with session key)
```

**Perfect Forward Secrecy (TLS 1.3):** Each TLS session uses a new ephemeral Diffie-Hellman key pair. If the server's private key is compromised in the future, past sessions cannot be decrypted (the session key was never transmitted, only derived). This is why TLS 1.3 is mandatory for compliance-heavy workloads.

---

## 4. CloudTrail — Audit Architecture

```mermaid
graph TB
    subgraph Event_Sources["Event Sources"]
        IAM_API["IAM API calls<br/>CreateUser, AttachPolicy"]
        EC2_API["EC2 API calls<br/>RunInstances, TerminateInstances"]
        S3_API["S3 data events<br/>GetObject, PutObject"]
        Console["Console actions<br/>(all become API calls)"]
    end

    subgraph CloudTrail["CloudTrail Processing"]
        CT_Service["CloudTrail Service<br/>Receives all control plane events<br/>+ configured data plane events"]
        JSON["JSON event format:<br/>eventTime, userIdentity,<br/>eventName, requestParameters,<br/>sourceIPAddress, errorCode"]
    end

    subgraph Storage["Audit Log Storage"]
        S3_Logs["S3 Bucket<br/>Encrypted with KMS<br/>Log file integrity validation<br/>(SHA-256 digest files)"]
        CW_Logs["CloudWatch Logs<br/>For real-time alerting"]
        Athena["Athena<br/>SQL queries over S3 logs<br/>Who deleted this object?"]
    end

    Event_Sources --> CT_Service --> JSON --> S3_Logs & CW_Logs
    S3_Logs --> Athena
```

**Log file integrity validation:** CloudTrail generates a SHA-256 digest file every hour referencing all log files delivered that hour. The digest file itself is signed with CloudTrail's private key. You can verify no log files were tampered with or deleted by checking the digest chain.

---

## 5. Security Failure Scenarios

| Scenario | Attack | What Happens | Mitigation |
|----------|--------|-------------|-----------|
| Access keys in GitHub | Attacker finds keys via automated scanning | Full API access within minutes | Use IAM roles, GitHub secret scanning |
| Overly permissive IAM | Compromised Lambda has s3:* | Can read/modify all buckets | Least privilege — specific ARN + specific actions |
| Public S3 bucket | Attacker finds bucket | Data breach, public read access | Block Public Access + S3 Access Analyzer |
| No MFA on root | Phishing attack | Total account takeover | MFA mandatory on root |
| CloudTrail disabled | Attacker disables logging | No forensic record | Alert on CloudTrail stop, SCPs preventing disable |
| Insecure user data | EC2 user data contains secrets | Instance metadata leaks secrets | Secrets Manager + IAM roles |
