# A08 — Viva Master Sheet
**Track: Academic | USE THIS BEFORE EVERY EXAM**

---

## Fast-Fire Definitions

| Term | Answer in ≤15 words |
|------|---------------------|
| Cloud computing | On-demand network access to shared configurable resources with minimal management |
| IaaS | Provider gives VMs + storage + network; you manage OS upward |
| PaaS | Provider gives platform; you manage only app and data |
| SaaS | Provider gives complete app; you manage only your data |
| Hypervisor | Software creating and managing virtual machines (VMM) |
| Type-1 Hypervisor | Runs directly on bare metal, no host OS between it and hardware |
| Type-2 Hypervisor | Runs as application on top of host OS |
| Container | Isolated process using Linux namespaces + cgroups, shares host kernel |
| NIST | National Institute of Standards and Technology — defines cloud computing standard |
| Elasticity | Automatic bidirectional scaling based on demand |
| CAP Theorem | Distributed systems: pick 2 of Consistency, Availability, Partition Tolerance |
| MapReduce | Distributed batch: Map (key-value transform) → Shuffle → Reduce (aggregate) |
| HDFS | Hadoop Distributed File System: 128MB blocks, 3x replication |
| TCO | Total Cost of Ownership — all costs over asset lifetime |
| SLA | Service Level Agreement — contractual uptime guarantee with remedies |
| RPO | Recovery Point Objective — max acceptable data loss in time |
| RTO | Recovery Time Objective — max time allowed to restore service |
| IAM | Identity and Access Management — AWS permission system |
| VPC | Virtual Private Cloud — logically isolated network in AWS |
| AMI | Amazon Machine Image — OS template for launching EC2 instances |
| EBS | Elastic Block Store — persistent block storage for EC2 |
| S3 | Simple Storage Service — object storage in buckets |
| EC2 | Elastic Compute Cloud — AWS virtual machine service |
| Lambda | Serverless function execution — no server management, pay per invocation |
| Security Group | Instance-level stateful firewall — only allow rules |
| NACL | Subnet-level stateless firewall — allow and deny rules |
| IGW | Internet Gateway — enables internet access for public subnets |
| NAT Gateway | Allows private subnet outbound internet, blocks inbound |
| PUE | Power Usage Effectiveness = Total DC power ÷ IT power (lower is better) |

---

## High-Value Comparison Questions

**Parallel vs Distributed Computing:**  
Parallel = same machine, shared memory, tight coupling, speed goal.  
Distributed = multiple machines, message passing, loose coupling, scale + fault tolerance goal.

**Full Virtualization vs Para-Virtualization:**  
Full = guest OS unmodified, trap-and-emulate overhead, any OS.  
Para = guest OS modified with hypercalls, faster, requires OS modification.

**VM vs Container:**  
VM = GB size, seconds startup, strong isolation, own kernel.  
Container = MB size, milliseconds startup, process isolation, shared kernel.

**Security Group vs NACL:**  
SG = instance-level, stateful, only allow rules.  
NACL = subnet-level, stateless, allow AND deny rules.

**IGW vs NAT Gateway:**  
IGW = bidirectional internet for public subnets.  
NAT GW = outbound-only internet for private subnets.

**Stop vs Terminate EC2:**  
Stop = paused, EBS preserved, no compute charge.  
Terminate = deleted, root EBS deleted, gone.

**SSE-S3 vs SSE-KMS:**  
SSE-S3 = AWS manages key, simpler, no per-decrypt audit.  
SSE-KMS = you control key, CloudTrail logs every decrypt, compliance-grade.

**RTO vs RPO:**  
RTO = how fast you recover (time).  
RPO = how much data you can lose (expressed as time = backup interval).

**MapReduce vs Spark:**  
MapReduce = disk-based intermediate, slow for iteration.  
Spark = in-memory, 10–100x faster for iterative algorithms.

**CapEx vs OpEx:**  
CapEx = upfront hardware buy, asset on balance sheet.  
OpEx = monthly cloud bill, expense.

---

## Trap Questions Your Teacher Will Ask

**Q: Is KVM Type-1 or Type-2?**  
A: Type-1. It's a kernel module that turns Linux kernel into a hypervisor. The kernel manages hardware directly.

**Q: Serverless means no servers. True or False?**  
A: False. Servers exist; you don't manage or pay for idle time. "Serverless" = server-management-free.

**Q: Cloud is always cheaper than on-premise. True or False?**  
A: False. For stable 24/7 workloads with predictable load, on-premise can be cheaper long-term. Cloud wins for variable load, rapid scaling, no upfront capital.

**Q: Containers provide the same isolation as VMs. True or False?**  
A: False. VMs have stronger isolation (separate kernel, hypervisor boundary). Containers share the host kernel — a kernel exploit affects all containers.

**Q: If an S3 bucket has Block Public Access disabled and no bucket policy, is it public?**  
A: No. Disabling Block Public Access removes the safeguard but doesn't grant access. You still need a bucket policy with `"Principal": "*"` to make it public.

**Q: What happens when a CloudWatch Alarm fires?**  
A: Alarm state transitions OK → ALARM. Configured actions execute: SNS notification, Auto Scaling policy, EC2 action. These actions are independent and configurable.

**Q: What is data locality in MapReduce?**  
A: The scheduler assigns Map tasks to the nodes that physically store the data being processed. Avoids network transfer of potentially terabytes of input data.

---

## Architecture Questions (Deep Prep)

**Q: Design a highly available 3-tier web app on AWS.**

```
Internet → Route 53 → CloudFront CDN
→ ALB (multi-AZ, public subnets)
→ EC2 ASG (private subnets, 2+ AZs, min:2 max:10)
→ RDS PostgreSQL Multi-AZ (isolated private subnets)
Supporting: NAT GW, S3 for static, Secrets Manager for DB creds,
CloudWatch alarms, CloudTrail logging, IAM roles on EC2
```

**Q: Walk through what happens when a user visits your website.**  
A: DNS query → Route 53 → ALB IP → TCP handshake → TLS handshake → HTTP request → ALB routing rule → EC2 target → App code → DB query → Response → TLS encrypt → back to browser.
