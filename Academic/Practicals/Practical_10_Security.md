# Practical 10 — Cloud Security
**Objective:** Implement IAM roles/policies, configure S3 encryption, and set up firewall rules with least-privilege principles.

---

## All Terms You Will Encounter

| Term | Definition |
|------|-----------|
| **IAM** | Identity and Access Management — AWS's permission system |
| **IAM User** | Permanent identity for a person. Has username/password + optional access keys |
| **IAM Role** | Temporary identity assumed by services, applications, or users |
| **IAM Policy** | JSON document defining allowed/denied actions on resources |
| **ARN** | Amazon Resource Name — unique identifier for any AWS resource |
| **Least Privilege** | Grant only minimum permissions required for the task |
| **Managed Policy** | Reusable policy maintained by AWS or by you |
| **Inline Policy** | Policy embedded directly in one user/group/role (not reusable) |
| **Trust Policy** | Document defining who CAN assume a role |
| **Permission Policy** | Document defining what a role CAN DO |
| **Access Key** | Programmatic access credentials (Access Key ID + Secret Access Key) |
| **MFA** | Multi-Factor Authentication — second factor (phone app/hardware token) |
| **SSE-S3** | Server-Side Encryption with S3-managed keys |
| **SSE-KMS** | Server-Side Encryption with KMS-managed keys |
| **KMS** | Key Management Service — create and manage encryption keys |
| **CMK** | Customer Master Key — encryption key you create in KMS |
| **Security Group** | Instance-level stateful firewall (AWS VPC) |
| **WAF** | Web Application Firewall — blocks HTTP attacks (SQLi, XSS) |
| **CloudTrail** | Logs every API call made in your AWS account |
| **Principle of Least Privilege** | Users/services get minimum permissions needed, nothing more |

---

## IAM Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                      IAM PERMISSION MODEL                            │
│                                                                      │
│  WHO (Principal)          CAN DO (Action)        ON WHAT (Resource)  │
│  ┌────────────────┐       ┌──────────────────┐   ┌────────────────┐  │
│  │ IAM User       │       │ s3:GetObject     │   │ my-bucket/*   │  │
│  │ IAM Role       │  ───► │ ec2:DescribeInst │ ─►│ EC2 instances │  │
│  │ AWS Service    │       │ rds:Connect      │   │ RDS databases │  │
│  │ External       │       │ ...              │   │ ...           │  │
│  └────────────────┘       └──────────────────┘   └────────────────┘  │
│                                                                      │
│  EVALUATION LOGIC:                                                   │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  Default: DENY everything                                      │  │
│  │  Explicit ALLOW: grants access (from identity or resource pol.)│  │
│  │  Explicit DENY: overrides any ALLOW (cannot be overridden)    │  │
│  │                                                                │  │
│  │  Decision: ALLOW only if explicitly allowed AND not denied    │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Step-by-Step

### Part A: Create Least-Privilege IAM Users and Roles

#### Step A1: Create IAM User with Read-Only S3 Access

```
IAM Console → Users → Create User
Name: s3-readonly-user
Access type: AWS Management Console access + Programmatic access
Password: (set strong password)
```

**Create custom policy (NOT AmazonS3FullAccess — that's too broad):**

1. IAM → Policies → **Create Policy** → JSON editor:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowS3ReadOnly",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::my-website-bucket",
                "arn:aws:s3:::my-website-bucket/*"
            ]
        }
    ]
}
```

2. Policy name: `S3ReadOnlyMyBucket`
3. Attach this policy to `s3-readonly-user`

**Why not `s3:*`?** Least privilege. This user should read, not delete, overwrite, or change permissions. If compromised, attacker can read but not destroy data.

---

#### Step A2: Create IAM Role for EC2 to Access S3

Instead of storing access keys on EC2 (dangerous), use an IAM Role that EC2 instances assume automatically.

```
IAM → Roles → Create Role
Trusted entity: AWS Service → EC2
Permissions: Attach S3ReadOnlyMyBucket policy
Role name: EC2-S3-ReadOnly-Role
```

**Attach to EC2:**
- EC2 Console → Instance → Actions → Security → Modify IAM Role
- Select: `EC2-S3-ReadOnly-Role`

**Test on EC2 instance:**
```bash
# SSH into EC2
ssh -i key.pem ec2-user@EC2_IP

# No access keys needed! Role is assumed automatically
aws s3 ls s3://my-website-bucket
# Works — instance has permission via role

aws s3 rm s3://my-website-bucket/index.html
# Fails — only GetObject and ListBucket allowed
```

---

#### Step A3: Enable MFA on Root Account

1. AWS Console → Account (top right) → Security Credentials
2. Multi-Factor Authentication → **Assign MFA device**
3. Choose: Authenticator app (Google Authenticator / Authy)
4. Scan QR code → Enter two consecutive codes
5. MFA enabled

**Test:** Log out → Log in → After password, asked for MFA code → account now secure even if password stolen.

---

### Part B: S3 Encryption

#### Step B1: Enable Default Bucket Encryption

1. S3 → Your bucket → Properties
2. Default encryption → **Edit**
3. Encryption type: **SSE-S3** (simplest — AWS manages keys)
4. Save

From now on: every new object uploaded is automatically encrypted at rest.

#### Step B2: Use SSE-KMS (More Control)

1. KMS → Customer Managed Keys → **Create Key**
2. Key type: Symmetric (AES-256 GCM)
3. Key usage: Encrypt and decrypt
4. Alias: `my-s3-key`
5. Key administrators: your admin IAM user
6. Key users: `EC2-S3-ReadOnly-Role`
7. Create

8. S3 → Bucket → Properties → Default encryption:
   - SSE-KMS
   - AWS KMS Key: `my-s3-key`

**The difference this makes:**
- SSE-S3: AWS manages the key. You can't audit key usage. Anyone with S3 access gets the data.
- SSE-KMS: You control the key. CloudTrail logs every decrypt operation. Revoke key = data permanently inaccessible even to AWS staff. Required for compliance (HIPAA, PCI-DSS).

---

### Part C: Security Group as Firewall

#### Step C1: Create Restrictive Security Group

```
EC2 → Security Groups → Create Security Group
Name: production-sg
VPC: your-custom-vpc

Inbound Rules:
Type        Protocol  Port   Source
HTTPS       TCP       443    0.0.0.0/0    (public HTTPS)
SSH         TCP       22     YOUR_IP/32   (only your IP can SSH)

Outbound Rules:
Type        Protocol  Port   Destination
HTTPS       TCP       443    0.0.0.0/0    (outbound HTTPS calls)
HTTP        TCP       80     0.0.0.0/0    (updates, package downloads)
DNS         UDP       53     0.0.0.0/0    (DNS lookups)
PostgreSQL  TCP       5432   10.0.2.0/24  (DB in private subnet only)
```

**Why restrict outbound?** Data exfiltration prevention. If attacker compromises EC2, restrictive outbound rules prevent sending stolen data to external servers.

---

#### Step C2: Test Security Group Rules

```bash
# From EC2 instance, these should WORK:
curl https://api.github.com        # HTTPS outbound — allowed
dig google.com                     # DNS — allowed

# These should FAIL (intentionally blocked):
curl http://example.com            # HTTP outbound not allowed
nc -zv 8.8.8.8 9999               # Random port — blocked
```

---

#### Step C3: Enable CloudTrail Audit Logging

```
CloudTrail → Create Trail
Name: my-audit-trail
Apply to all regions: Yes
S3 bucket: Create new (cloudtrail-logs-yourname)
Log file encryption: Enabled (SSE-KMS with your key)
Enable log file validation: Yes
```

**Test — Check who created a Security Group:**
```
CloudTrail → Event History
Filter: Event name = CreateSecurityGroup
```
Shows: who did it, when, from which IP, with what parameters.

---

## Expected Output
- IAM user with only S3 read access created
- EC2 assumes IAM role (no keys in code)
- S3 bucket has default KMS encryption
- Security Group restricts both inbound and outbound
- CloudTrail logging all API calls

---

## Viva Questions — Practical 10

1. **What is the difference between an IAM User and an IAM Role?**  
   A: User = permanent identity for a person (has long-term credentials). Role = temporary identity assumed by services (EC2, Lambda) or users from other accounts. Roles issue temporary credentials valid for 1-12 hours. Use roles for services — never put access keys on EC2 instances.

2. **Explain the IAM policy evaluation logic.**  
   A: Default = deny all. Explicit ALLOW grants access. Explicit DENY overrides any ALLOW and cannot be overridden. Decision: allow only if there is an explicit ALLOW and no explicit DENY.

3. **What is the principle of least privilege? Give a specific example.**  
   A: Grant only minimum permissions required. Example: Lambda that reads from one DynamoDB table gets only `dynamodb:GetItem` on that specific table ARN — not `dynamodb:*` on all tables. This limits blast radius if compromised.

4. **What is the difference between SSE-S3 and SSE-KMS?**  
   A: SSE-S3: AWS manages encryption key internally, no audit trail per object, simpler. SSE-KMS: You or AWS creates/manages key in KMS, CloudTrail logs every decrypt, you can revoke key to make data permanently inaccessible, required for some compliance frameworks.

5. **Why should you never store AWS access keys on EC2 instances?**  
   A: If instance is compromised, attacker gets permanent credentials. If code pushed to GitHub, keys leaked publicly (AWS scans GitHub and alerts on leaked keys but damage can be done in minutes). Use IAM Roles instead — they provide temporary credentials that auto-rotate, scoped to that instance only.

6. **What does CloudTrail record?**  
   A: Every API call made to AWS: who made it (IAM user/role ARN), when (timestamp), from where (source IP), what service, what action, what parameters, and whether it succeeded. Stored as JSON files in S3. Retained 90 days in Event History for free; enable Trail for longer retention.

7. **What is an ARN? Break down this ARN: `arn:aws:s3:::my-bucket/*`**  
   A: Amazon Resource Name — unique identifier. Format: `arn:partition:service:region:account:resource`. This ARN: `arn` (fixed), `aws` (partition), `s3` (service), empty (S3 is global, no region), empty (no account), `my-bucket/*` (all objects in my-bucket). The `*` is a wildcard matching all objects.

8. **What is the difference between a Security Group and a Network ACL (from a security perspective)?**  
   A: SG = instance level, stateful, only ALLOW rules — easier for application security. NACL = subnet level, stateless, ALLOW and DENY rules — can explicitly block known malicious IPs at subnet boundary. For defense in depth: use both. NACL as outer perimeter (subnet), SG as inner fence (instance).

9. **If an EC2 instance has a role with S3 read access, how does it authenticate to S3?**  
   A: Instance Metadata Service provides temporary credentials (Access Key ID + Secret + Session Token) that are rotated automatically every few hours. The AWS SDK on the instance queries 169.254.169.254/latest/meta-data/iam/security-credentials/ROLE-NAME to get these. No hardcoded keys needed.

10. **What is MFA Delete on S3?**  
    A: Requires MFA authentication to permanently delete objects or disable versioning. Even if attacker has your access keys, they cannot delete data without physical access to your MFA device. Recommended for critical data buckets.

---
