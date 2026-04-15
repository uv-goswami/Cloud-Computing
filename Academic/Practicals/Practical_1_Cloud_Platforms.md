# Practical 1 — Introduction to Cloud Platforms
**Objective:** Familiarize with cloud platform consoles and identify key services.

---

## All Terms You Will Encounter (Examiner Will Ask Every One)

| Term | Definition |
|------|-----------|
| **AWS Management Console** | Web-based GUI for managing all AWS services. Accessed at console.aws.amazon.com |
| **Free Tier** | AWS's offer: certain services free for 12 months (or always-free) within usage limits |
| **Region** | Geographic area containing multiple data centers (e.g., ap-south-1 = Mumbai) |
| **Service** | An AWS product (EC2, S3, IAM, VPC, Lambda, RDS, etc.) |
| **Pricing Calculator** | Tool to estimate monthly AWS costs before committing |
| **GCP Console** | Google Cloud's management dashboard at console.cloud.google.com |
| **Azure Portal** | Microsoft Azure's management dashboard at portal.azure.com |
| **Compute** | Services providing processing power (EC2/VMs/GCE) |
| **Storage** | Services storing data (S3, EBS, Blob Storage) |
| **Networking** | Services managing network (VPC, Route 53, Load Balancers) |

---

## Architecture Diagram: Three Major Cloud Platforms

```
┌──────────────────────────────────────────────────────────────────────┐
│                    CLOUD PROVIDER LANDSCAPE                          │
│                                                                      │
│  ┌────────────────────┐  ┌────────────────────┐  ┌────────────────┐ │
│  │   AWS (Amazon)     │  │  Microsoft Azure   │  │  GCP (Google)  │ │
│  │                    │  │                    │  │                │ │
│  │  COMPUTE: EC2      │  │  COMPUTE: VMs      │  │  COMPUTE: GCE  │ │
│  │  STORAGE: S3, EBS  │  │  STORAGE: Blob     │  │  STORAGE: GCS  │ │
│  │  NETWORK: VPC      │  │  NETWORK: VNet     │  │  NETWORK: VPC  │ │
│  │  DB:      RDS      │  │  DB: Azure SQL     │  │  DB: Cloud SQL │ │
│  │  SERVLSS: Lambda   │  │  SERVLSS: Functions│  │  SERVLSS: Funcs│ │
│  │  MONITOR: CloudWtch│  │  MONITOR: Monitor  │  │  MONITOR: Cloud│ │
│  │                    │  │                    │  │  Monitoring    │ │
│  │  200+ services     │  │  200+ services     │  │  100+ services │ │
│  │  33% market share  │  │  21% market share  │  │  10% mkt share │ │
│  └────────────────────┘  └────────────────────┘  └────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Step-by-Step: AWS Account Exploration

### Step 1: Log Into AWS Console
1. Go to https://console.aws.amazon.com
2. Enter your email and password
3. **Important:** After login, check top-right — ensure region shows `Asia Pacific (Mumbai) ap-south-1` (or your preferred region)

**Why region matters:** Resources you create are region-specific. An EC2 in Mumbai is not visible from the North Virginia console view.

### Step 2: Explore the Services Menu
1. Click "Services" in top navigation bar
2. Identify these key categories:

```
┌──────────────────────────────────────────────────────────────────┐
│              AWS SERVICE CATEGORIES                              │
│                                                                  │
│  COMPUTE:                                                        │
│  • EC2 — Virtual machines (your most-used service)              │
│  • Lambda — Serverless functions                                 │
│  • Elastic Beanstalk — PaaS for web apps                       │
│  • ECS/EKS — Container services                                 │
│                                                                  │
│  STORAGE:                                                        │
│  • S3 — Object storage (files, images, backups)                 │
│  • EBS — Block storage (EC2 disks)                              │
│  • EFS — Elastic File System (shared NFS)                       │
│  • Glacier — Archival storage (cheap, slow)                    │
│                                                                  │
│  DATABASE:                                                       │
│  • RDS — Managed SQL (PostgreSQL, MySQL, Oracle)                │
│  • DynamoDB — NoSQL, serverless, key-value                      │
│  • ElastiCache — In-memory (Redis, Memcached)                  │
│                                                                  │
│  NETWORKING:                                                     │
│  • VPC — Your private network                                   │
│  • Route 53 — DNS service                                       │
│  • CloudFront — CDN (content delivery network)                 │
│  • API Gateway — Managed API endpoint                           │
│                                                                  │
│  SECURITY:                                                       │
│  • IAM — Users, roles, permissions                              │
│  • KMS — Encryption key management                              │
│  • CloudTrail — API audit logging                               │
│  • WAF — Web Application Firewall                               │
│                                                                  │
│  MONITORING:                                                     │
│  • CloudWatch — Metrics, alarms, logs                           │
└──────────────────────────────────────────────────────────────────┘
```

### Step 3: Use the Pricing Calculator
1. Go to https://calculator.aws.amazon.com
2. Click "Create estimate"
3. Add service: EC2 → t2.micro → 1 instance → 730 hours/month
4. Observe: Free tier shows $0.00/month for first 12 months
5. Add another: t3.medium → 730 hours → approximately $35/month

**Why this matters for exam:** Examiner may ask about cloud pricing models. You can explain: On-Demand (per hour), Reserved (1-3 year discount), Spot (cheapest, interruptible).

---

## Expected Output
- You can navigate AWS console without getting lost
- You can identify which service category (compute/storage/network/db) a service belongs to
- You can estimate monthly cost using the calculator

---

## Common Mistakes
| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| Creating resources in wrong region | Resources in us-east-1 billed normally; free tier is region-agnostic but pay attention | Always check region indicator top-right |
| Accidentally enabling paid services | Some services not in free tier | Check pricing page before enabling |
| Not enabling MFA | Account security risk | Enable MFA immediately after account creation |

---

## Viva Questions — Practical 1

1. **What is AWS Free Tier? How long does it last?**  
   A: Free tier offers certain services free for 12 months after account creation (e.g., EC2 t2.micro 750hrs/month, S3 5GB). Some services are always free (Lambda 1M requests/month, DynamoDB 25GB).

2. **Name 5 AWS compute services.**  
   A: EC2, Lambda, Elastic Beanstalk, ECS (containers), EKS (Kubernetes), Fargate (serverless containers).

3. **What is the difference between S3 and EBS?**  
   A: S3 = object storage, accessible via HTTP, unlimited size, not mounted as drive. EBS = block storage, attached to EC2 like hard drive, mounted as filesystem, supports random I/O.

4. **What does the AWS Management Console actually connect to?**  
   A: AWS APIs. The console is a web frontend that makes API calls to AWS services on your behalf. Everything you can do in the console can also be done via AWS CLI or SDK.

5. **Why are there different AWS regions? Why not just one?**  
   A: Latency (users in India get low latency from Mumbai region), data sovereignty (some countries require data to stay within borders), redundancy (a disaster in one region doesn't affect others), pricing (some regions are cheaper).

6. **What is an IAM user? Should you use root to create EC2?**  
   A: IAM user = sub-account with specific permissions. Never use root for daily work. Root has unlimited power — compromise means total account loss. Create an IAM admin user and use that.

7. **What is the GCP equivalent of EC2?**  
   A: Google Compute Engine (GCE).

8. **What is the Azure equivalent of S3?**  
   A: Azure Blob Storage.

---
