# Cloud Computing Complete — Two-Track Master Pack
### BSc (H) Computer Science | NEP UGCF 2022 | DSC18/GE7d/DSE8e

---

## Two Tracks. One Purpose.

This pack is built in two completely separate tracks. Use both.

```
┌─────────────────────────────────┬────────────────────────────────────┐
│  ACADEMIC TRACK                 │  DEEP DIVE TRACK                   │
│  Exam + Viva Ready              │  Industry + Systems Internals       │
├─────────────────────────────────┼────────────────────────────────────┤
│  Clean definitions              │  How systems actually work          │
│  Comparison tables              │  Internal workflows + data flows    │
│  Syllabus-exact coverage        │  Failure scenarios + trade-offs     │
│  10 complete practicals         │  Platform internals (AWS, KVM)      │
│  150+ viva questions            │  Real architecture reasoning        │
└─────────────────────────────────┴────────────────────────────────────┘
```

---

## File Structure

```
Cloud-Computing-Complete/
│
├── README.md                              ← This file
│
├── Academic/
│   ├── Theory/
│   │   ├── A01_Fundamentals.md            Unit 1 — History, NIST, platforms
│   │   ├── A02_Architecture.md            Unit 4 — IaaS/PaaS/SaaS, deployment
│   │   ├── A03_Virtualization.md          Unit 3 — Types, hypervisors, containers
│   │   ├── A04_Distributed_Computing.md   Unit 2 — Parallel vs Distributed, CAP
│   │   ├── A05_MapReduce.md               Unit 5 — MapReduce, Hadoop, HDFS
│   │   ├── A06_Management.md              Unit 6 — TCO, SLA, CapEx/OpEx
│   │   ├── A07_Security.md                Unit 7 — IAM, encryption, risks
│   │   └── A08_Viva_MasterSheet.md        150+ Q&A — ruthless examiner prep
│   │
│   └── Practicals/
│       ├── P01_Cloud_Platforms.md
│       ├── P02_EC2.md
│       ├── P03_VPC.md
│       ├── P04_AutoScaling_ALB.md
│       ├── P05_Static_Website.md
│       ├── P06_CloudWatch.md
│       ├── P07_OpenStack_Install.md
│       ├── P08_OpenStack_VM.md
│       ├── P09_OpenStack_Networking.md
│       └── P10_Security.md
│
├── Deep_Dive/
│   ├── D01_Virtualization_Internals.md    KVM, QEMU, hypervisor traps, VMCS
│   ├── D02_AWS_Infrastructure.md          Nitro system, Xen→Nitro, hardware
│   ├── D03_Networking_Internals.md        VPC packet path, OVS, VXLAN, iptables
│   ├── D04_Storage_Internals.md           EBS architecture, S3 internals, HDFS
│   ├── D05_Distributed_Systems.md         Consensus, Raft, Paxos, CAP in practice
│   ├── D06_MapReduce_Internals.md         Scheduler, shuffle, sort, Spark vs MR
│   └── D07_Security_Internals.md          IAM eval engine, KMS HSM, TLS internals
│
└── Diagrams/
    ├── cloud_service_models.mmd
    ├── virtualization_layers.mmd
    ├── vpc_packet_flow.mmd
    ├── kvm_architecture.mmd
    ├── aws_nitro.mmd
    ├── mapreduce_flow.mmd
    ├── three_tier_arch.mmd
    ├── iam_evaluation.mmd
    └── hdfs_write_path.mmd
```

---

## What Beats Your Classmates' Work

| Criteria | Kshitiz's ZIP | Riya's ZIP | Cloud-Computing-main | **This Pack** |
|----------|--------------|------------|----------------------|---------------|
| Practicals | ✅ Good steps | ❌ None | ❌ None | ✅ Full + deeper |
| Theory depth | 🔸 Surface | ❌ Keywords only | ✅ Virtualization | ✅ Both tracks |
| Industry internals | ❌ | ❌ | 🔸 Partial | ✅ Full deep dive |
| Mermaid diagrams | ❌ ASCII only | ❌ None | ❌ ASCII only | ✅ All Mermaid |
| Viva preparation | 🔸 Per practical | ❌ None | ❌ None | ✅ 150+ master Q&A |
| Failure scenarios | ❌ | ❌ | ❌ | ✅ Every topic |
| AWS internals | ❌ | ❌ | ❌ | ✅ Nitro, KVM, VPC |

---

## Quick Reference — Every Term Your Teacher Will Ask

| Term | One Line | Track |
|------|---------|-------|
| Hypervisor | Software layer creating/managing VMs | Academic + Deep |
| KVM | Linux kernel module = Type-1 hypervisor | Deep |
| VMCS | VM Control Structure — Intel hardware for trap control | Deep |
| VXLAN | Layer-2 overlay over Layer-3 for tenant networks | Deep |
| Nitro | AWS's custom hypervisor replacing Xen since 2017 | Deep |
| IaaS | You manage OS and above; provider manages hardware | Academic |
| PaaS | You manage app+data; provider manages platform | Academic |
| SaaS | You manage data only | Academic |
| NIST | Defines 5 CC characteristics, 3 service models, 4 deployments | Academic |
| CAP | Consistency/Availability/Partition — pick 2 in distributed systems | Both |
| MapReduce | Distributed batch: Map (transform) → Shuffle → Reduce (aggregate) | Academic |
| HDFS | Hadoop's distributed FS: 128MB blocks, 3x replication | Academic |
| TCO | Total Cost of Ownership — all costs, not just hardware | Academic |
| SLA | Service Level Agreement — contractual uptime guarantees | Academic |
| IAM eval | Default deny → explicit allow → explicit deny wins | Deep |
| KMS + HSM | Key Management Service backed by Hardware Security Modules | Deep |
