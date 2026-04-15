# D02 — AWS Infrastructure Internals
**Track: Deep Dive | How EC2 actually works inside Amazon**

---

## 1. The Xen Era (2006–2017) — How AWS Was Originally Built

AWS launched EC2 in 2006 using the **Xen hypervisor** in para-virtualized mode.

```mermaid
graph TB
    subgraph Xen_Architecture["AWS Xen Architecture (Legacy)"]
        subgraph Dom0["Domain 0 (Privileged VM — runs on host)"]
            XenStore["XenStore<br/>Config + event channel"]
            NetFront["Netback driver<br/>Handles all guest networking"]
            BlkFront["Blkback driver<br/>Handles all guest disk I/O"]
            Dom0Kernel["Dom0 Linux Kernel<br/>Full hardware access"]
        end

        subgraph DomU1["Domain U (Guest VM — Customer 1)"]
            G1App["Customer App"]
            G1OS["Guest OS (modified)"]
            G1Net["netfront driver"]
            G1Blk["blkfront driver"]
        end

        subgraph DomU2["Domain U (Guest VM — Customer 2)"]
            G2App["Customer App"]
            G2OS["Guest OS (modified)"]
        end

        XenHV["Xen Hypervisor<br/>(bare metal, tiny ~150KB)"]
        Hardware["Physical Hardware"]
    end

    G1Net <-->|event channels + grant tables| NetFront
    G1Blk <-->|event channels + grant tables| BlkFront
    Dom0Kernel --> Hardware
    XenHV --> Hardware
    DomU1 & DomU2 & Dom0 --> XenHV
```

**The Dom0 problem:** Dom0 was a full Linux VM with hardware access. It was a large privileged attack surface. If Dom0 was compromised, all customer VMs on that host were compromised. Also, Dom0 consumed significant CPU and memory resources just for I/O handling.

---

## 2. The Nitro System (2017–present) — AWS's Custom Hypervisor

AWS built Nitro to fix Xen's Dom0 problem: move device emulation and security functions off the main CPU and onto dedicated hardware.

```mermaid
graph TB
    subgraph Nitro_System["AWS Nitro System"]
        subgraph EC2_Instances["Customer EC2 Instances"]
            VM1["Customer VM 1<br/>Gets 100% of CPU<br/>No hypervisor tax"]
            VM2["Customer VM 2"]
        end

        subgraph Nitro_Card["Nitro Card (Dedicated Hardware)"]
            NitroHV["Nitro Hypervisor<br/>Tiny KVM-based<br/>Only CPU/memory virtualization"]
            NitroNW["Nitro Network Card<br/>VPC networking in hardware<br/>Encapsulation/decapsulation"]
            NitroStorage["Nitro Storage Card<br/>EBS I/O in hardware<br/>NVMe over network"]
            NitroSec["Nitro Security Chip<br/>Hardware root of trust<br/>Instance identity"]
        end

        subgraph Physical["Physical Host"]
            CPU["CPU (100% to VMs)"]
            RAM["RAM (100% to VMs)"]
        end
    end

    VM1 & VM2 --> NitroHV
    NitroNW --> VPC_Net["VPC Network Fabric"]
    NitroStorage --> EBS["EBS Storage Network"]
    NitroHV --> CPU & RAM
```

### Before and After Nitro

| Aspect | Xen (Pre-Nitro) | Nitro |
|--------|----------------|-------|
| Hypervisor size | Full Xen + Dom0 | Tiny KVM-based core |
| CPU overhead | ~10–15% (Dom0 + Xen) | < 1% (hardware offload) |
| I/O handling | Dom0 CPU | Dedicated Nitro card |
| Networking | Dom0 Linux software | Nitro NW card (hardware) |
| Attack surface | Large (Dom0 = full Linux) | Minimal (tiny firmware) |
| Bare metal support | No | Yes (Nitro with no VM layer) |

**Result:** On Nitro, the hypervisor overhead is so small that AWS introduced **bare metal instances** — EC2 instances with NO hypervisor overhead at all (you get raw hardware, used for licensing requirements, custom hypervisors, nested virtualization).

---

## 3. EC2 Instance Lifecycle — Full Internal Flow

```mermaid
sequenceDiagram
    participant API as EC2 API (RunInstances)
    participant Scheduler as EC2 Scheduler
    participant Host as Physical Host
    participant Nitro as Nitro Hypervisor
    participant EBS as EBS Network
    participant VPC as VPC Network

    API->>Scheduler: RunInstances(type=t3.medium, AMI=ami-xxx)
    Note over Scheduler: Find host with: available capacity,<br/>correct instance family,<br/>requested AZ, placement group rules

    Scheduler->>Host: Place instance here
    Host->>Nitro: Create VM: 2 vCPU, 4GB, networking config
    Nitro->>EBS: Attach root volume (via NVMe over network)
    EBS->>Nitro: Volume attached, device visible to VM
    Nitro->>VPC: Provision ENI (Elastic Network Interface)
    VPC->>Nitro: MAC address, IP, VPC routing configured
    Nitro->>Nitro: VMLAUNCH — boot from EBS root volume
    Note over Nitro: VM boots, runs cloud-init, executes User Data
    Nitro->>API: Instance state: running
    API->>You: InstanceId, PublicIP, PrivateIP
```

---

## 4. EC2 Instance Types — Hardware Mapping

```mermaid
graph LR
    subgraph Families["EC2 Instance Families → Physical Hardware"]
        T["t3/t4g — Burstable<br/>Shared physical CPU<br/>CPU credit model<br/>Intel/Graviton"]
        M["m5/m6i — General Purpose<br/>Balanced CPU:RAM<br/>Intel/AMD/Graviton"]
        C["c5/c6i — Compute<br/>High vCPU:RAM ratio<br/>Intel/AMD/Graviton"]
        R["r5/r6i — Memory<br/>High RAM:CPU ratio<br/>Up to 768GB RAM"]
        P["p3/p4d — GPU<br/>NVIDIA V100/A100<br/>SR-IOV passthrough"]
        I["i3/i4i — Storage<br/>NVMe SSD direct attach<br/>Not EBS — local SSD"]
        Inf["inf1/inf2 — Inferentia<br/>AWS custom ML chip<br/>LLM inference"]
    end
```

### CPU Oversubscription in t-family Instances

t2/t3 instances use CPU credits. The physical host is oversubscribed — more vCPUs sold than physical cores available.

```
Physical host: 48 physical cores
t3.micro VMs: 2 vCPUs each → can sell 200+ t3.micro instances

When CPU is idle: vCPUs run at full speed, earn credits
When CPU spikes: credits consumed, eventually throttled to "baseline"
t3.micro baseline: 10% of 1 vCPU
```

For sustained CPU workloads: use m/c/r family. For burst workloads: t family is cheap and sufficient.

---

## 5. EBS Architecture — How EC2 Disk I/O Actually Works

EBS is NOT a local disk. It is a **network-attached block device** delivered over AWS's internal network.

```mermaid
graph LR
    subgraph EC2_Host["EC2 Physical Host"]
        VM["Your EC2 VM"]
        NitroStorage["Nitro Storage Controller<br/>NVMe interface to VM"]
    end

    subgraph EBS_Network["EBS Internal Network (25Gbps+)"]
        Net["Dedicated storage network<br/>Separate from EC2 data network"]
    end

    subgraph EBS_Fleet["EBS Storage Fleet"]
        EBSNode1["EBS Storage Node<br/>SSDs/HDDs"]
        EBSNode2["EBS Storage Node<br/>Replicated copy"]
    end

    VM -->|NVMe commands| NitroStorage
    NitroStorage -->|NVMe over Fabric (NVMeoF)| Net
    Net --> EBSNode1 & EBSNode2
```

**Implications:**
- EBS latency: ~0.5–2ms (network round trip)
- Local instance store (i3/i4i): ~0.01ms (physically on host)
- EBS survives EC2 instance death (data on separate fleet)
- io2 Block Express: ~500K IOPS, ~0.1ms latency (dedicated path)

---

## 6. AWS Global Network — How Traffic Gets to Your EC2

```mermaid
graph TB
    User["User Browser<br/>Anywhere in world"]

    subgraph AWS_Edge["AWS Edge (100+ Points of Presence)"]
        CloudFront["CloudFront CDN<br/>Static content cached here"]
        Shield["AWS Shield DDoS<br/>Traffic scrubbing"]
    end

    subgraph AWS_Backbone["AWS Global Backbone<br/>(Private fiber network)"]
        Backbone["AWS private fiber<br/>Multiple 100Gbps paths<br/>Never routes over public internet"]
    end

    subgraph Region["AWS Region ap-south-1"]
        AZ_A["AZ ap-south-1a<br/>Data Center"]
        AZ_B["AZ ap-south-1b<br/>Data Center"]
    end

    User --> CloudFront
    CloudFront --> Shield
    Shield --> Backbone
    Backbone --> AZ_A & AZ_B
```

**Key facts for exams:**
- Traffic between AWS services in the same region stays on AWS backbone (not public internet)
- Cross-region traffic also uses AWS backbone (not public internet)
- AWS has 100+ edge PoPs (Points of Presence) for CloudFront
- Transfer within same AZ: free. Cross-AZ: $0.01/GB. Cross-region: $0.02–0.09/GB.

---

## 7. The Graviton Processor — AWS Custom ARM CPU

AWS designed its own CPU for EC2: **AWS Graviton (ARM-based)**.

```
Graviton1 (2018): 16-core ARM Cortex-A72
Graviton2 (2020): 64-core Neoverse N1 — 7nm, 40% better price-performance vs x86
Graviton3 (2022): 64-core Neoverse V1 — 25% better than Graviton2
Graviton4 (2024): 96-core — 30% better than Graviton3
```

**Why AWS built its own CPU:**
- x86 licensing: Intel/AMD charge per socket — expensive at cloud scale
- Custom features: hardware-specific instructions optimized for cloud workloads
- Vertical integration: AWS controls the full stack (chip → hypervisor → EC2)
- Cost: ~40% cheaper than equivalent x86 EC2 instances

**Trade-off:** ARM architecture. Applications compiled for x86 must be recompiled. Python, Java, Node.js = works fine (JIT or interpreted). Native binaries = must recompile.

---

## 8. Failure Analysis: EC2 Availability Zone Design

```mermaid
graph TB
    subgraph Region["Region: ap-south-1"]
        subgraph AZ_A["Availability Zone A<br/>(Independent power, cooling, networking)"]
            Host_A1["Physical Host 1"]
            Host_A2["Physical Host 2"]
        end
        subgraph AZ_B["Availability Zone B<br/>(Independent power, cooling, networking)"]
            Host_B1["Physical Host 3"]
            Host_B2["Physical Host 4"]
        end
        subgraph AZ_C["Availability Zone C<br/>(Independent power, cooling, networking)"]
            Host_C1["Physical Host 5"]
        end

        InterAZ["Inter-AZ network<br/>< 2ms latency<br/>25Gbps+"]
    end

    AZ_A <--> InterAZ <--> AZ_B <--> InterAZ
```

**Failure model:**
- Single instance failure: ~1 in 10,000 instance-hours (hardware defect, hypervisor crash)
- Single AZ failure: Rare but happens (power: us-east-1 2012, cooling: us-east-1 2023)
- Single region failure: Extremely rare (us-east-1 2021 was partial)
- Multi-region failure: Has not happened for major AWS regions

**Design implication:** For 99.99% availability, deploy across 2+ AZs with ALB and health checks. A single AZ gives ~99.9% at best.
