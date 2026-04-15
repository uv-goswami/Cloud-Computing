# D01 — Virtualization Internals
**Track: Deep Dive | How the hypervisor actually works under the hood**

---

## 1. The Problem Virtualization Solves — At Hardware Level

Before virtualization, the x86 architecture had a fundamental problem: **the ring privilege model breaks when you try to run an OS inside another OS.**

```mermaid
graph TB
    subgraph x86_Rings["x86 Privilege Rings"]
        R0["Ring 0 — Kernel Mode<br/>CPU, Memory, I/O control<br/>Only OS runs here"]
        R1["Ring 1 — Unused in modern OS"]
        R2["Ring 2 — Unused in modern OS"]
        R3["Ring 3 — User Mode<br/>Applications run here<br/>Cannot touch hardware"]
    end

    R3 -->|syscall trap| R0
    R0 -->|return| R3
```

**The virtualization problem:** A guest OS expects to run in Ring 0. But if it's already running inside a host OS, Ring 0 is taken by the hypervisor. When the guest OS executes a privileged instruction (like `MOV CR3` to change page tables), one of three things happens:
1. CPU silently ignores it — guest OS breaks
2. CPU crashes the guest
3. CPU traps to hypervisor — hypervisor emulates — guest continues

Option 3 is the basis of **trap-and-emulate virtualization.**

---

## 2. Full Virtualization — Trap-and-Emulate Internals

```mermaid
sequenceDiagram
    participant GuestOS as Guest OS (Ring 1/3)
    participant CPU as Physical CPU
    participant VMM as Hypervisor/VMM (Ring 0)
    participant HW as Hardware

    GuestOS->>CPU: Executes privileged instruction e.g., "cli" (disable interrupts)
    Note over CPU: This is a privileged op — not allowed from Ring 1
    CPU->>VMM: TRAP — CPU exception, control transfers to VMM
    VMM->>VMM: Save guest register state
    VMM->>VMM: Identify which instruction caused trap
    VMM->>HW: Execute actual hardware operation if safe
    HW->>VMM: Result
    VMM->>VMM: Emulate what guest expected to happen
    VMM->>GuestOS: Restore guest state, return as if instruction succeeded
    Note over GuestOS: Guest never knew it was trapped
```

**Performance cost:** Each trap = thousands of CPU cycles (context save, VMM decision, emulation, context restore). On a workload that frequently executes privileged instructions, this overhead becomes significant.

---

## 3. Intel VT-x and AMD-V — Hardware-Assisted Virtualization

Intel and AMD added hardware support specifically to make virtualization efficient. This is called **Intel VT-x (Virtualization Technology for x86)** and **AMD-V (AMD Virtualization)**.

### VMCS — VM Control Structure

The core Intel VT-x data structure. Every VM has its own VMCS — a 4KB memory region that the CPU uses to track VM state.

```mermaid
graph TB
    VMCS["VMCS — VM Control Structure<br/>(4KB per VM, stored in RAM)"]

    VMCS --> GuestState["Guest State Area<br/>RIP, RSP, RFLAGS, CR0, CR3<br/>Saved on VM Exit<br/>Restored on VM Entry"]
    VMCS --> HostState["Host State Area<br/>Hypervisor RIP, RSP, CR3<br/>Where CPU jumps on VM Exit"]
    VMCS --> Control["VM Execution Controls<br/>Which events cause VM Exit?<br/>CPUID? I/O ports? MSR access?<br/>CR3 writes? RDTSC?"]
    VMCS --> ExitInfo["VM Exit Information<br/>Why did exit happen?<br/>Which instruction?<br/>What address?"]
```

### VMENTER and VMEXIT — The Two Hardware Instructions

```mermaid
sequenceDiagram
    participant HV as Hypervisor (Ring 0)
    participant CPU
    participant Guest as Guest OS

    HV->>CPU: VMLAUNCH / VMRESUME instruction
    Note over CPU: CPU loads guest state from VMCS
    CPU->>Guest: Guest OS executes at full hardware speed
    Note over Guest: Running natively — no emulation overhead

    Guest->>CPU: Executes CPUID instruction (configured to exit)
    CPU->>HV: VMEXIT — CPU saves guest state to VMCS
    Note over CPU: CPU loads host state from VMCS
    HV->>HV: Handle VM Exit reason: CPUID
    HV->>HV: Fabricate CPUID response for guest
    HV->>CPU: VMRESUME
    CPU->>Guest: Returns from CPUID with fabricated values
```

**Key insight:** With VT-x, the CPU natively switches between host and guest mode. No software emulation of privilege levels. The guest runs in a new CPU mode called **VMX non-root operation** — it has near-native performance, but certain operations still cause VMEXIT to the hypervisor.

---

## 4. KVM Architecture — Deep Internals

KVM (Kernel-based Virtual Machine) is Linux's virtualization module. Understanding it means understanding how AWS EC2 actually works.

```mermaid
graph TB
    subgraph User_Space["User Space"]
        QEMU["QEMU Process<br/>One per VM<br/>Emulates: NIC, disk, USB<br/>Manages VM memory"]
        LibVirt["libvirt daemon<br/>API for managing VMs<br/>virsh, virt-manager"]
    end

    subgraph Kernel_Space["Linux Kernel Space"]
        KVM_Module["kvm.ko module<br/>+ kvm-intel.ko or kvm-amd.ko<br/>Exposes /dev/kvm device"]
        VFS["VFS — Virtual File System"]
        Scheduler["Linux CFS Scheduler<br/>Schedules vCPUs like threads"]
        MM["Memory Manager<br/>EPT/NPT page table mgmt"]
    end

    subgraph CPU_HW["CPU Hardware"]
        VTX["Intel VT-x / AMD-V<br/>VMCS management<br/>VMLAUNCH, VMRESUME"]
    end

    LibVirt --> QEMU
    QEMU -->|ioctl /dev/kvm| KVM_Module
    KVM_Module --> VTX
    KVM_Module --> Scheduler
    KVM_Module --> MM
    VTX -->|VMEXIT events| KVM_Module
    KVM_Module -->|I/O exits| QEMU
```

### The vCPU Execution Loop

```mermaid
flowchart TD
    Start["KVM: VMRESUME<br/>Guest vCPU runs"] --> Run["Guest executes instructions<br/>at native hardware speed"]
    Run --> Exit{"VMEXIT?"}
    Exit -->|No| Run
    Exit -->|Yes| Reason{"Exit Reason"}

    Reason -->|I/O instruction<br/>e.g. disk read| IOExit["KVM: handle in kernel<br/>or pass to QEMU"]
    Reason -->|HLT instruction<br/>guest idle| Halt["vCPU thread sleeps<br/>Linux scheduler picks other work"]
    Reason -->|EPT violation<br/>page not mapped| EPT["MMU: set up EPT entry<br/>map guest physical → host physical"]
    Reason -->|CPUID| CPID["Fabricate CPU info<br/>return to guest"]
    Reason -->|External interrupt| Inject["Inject interrupt into guest<br/>VMRESUME"]

    IOExit & Halt & EPT & CPID & Inject --> Resume["VMRESUME<br/>Guest continues"]
    Resume --> Run
```

### EPT — Extended Page Tables

Without EPT (software shadow page tables): every guest page table change triggers a VMEXIT. With EPT (hardware-supported):

```
Guest Virtual → Guest Physical → Host Physical
     └── Guest page tables      └── EPT (hardware-walked by MMU)

Without EPT: VMM must intercept every CR3 write, maintain shadow page tables = expensive
With EPT: CPU hardware walks both tables autonomously = near-native memory performance
```

---

## 5. Para-Virtualization Internals — Xen Hypercalls

In para-virtualization (Xen PV mode), the guest OS is modified to replace privileged instructions with **hypercalls** — direct function calls into the hypervisor.

```mermaid
sequenceDiagram
    participant Guest as Modified Guest Kernel
    participant HV as Xen Hypervisor

    Note over Guest: Guest needs to update page table
    Guest->>HV: HYPERCALL: mmu_update(mfn, ptr, val)
    Note over HV: No trap needed — direct function call
    HV->>HV: Validate request
    HV->>HV: Update machine frame number mapping
    HV->>Guest: Return success
    Note over Guest: ~5x faster than trap-and-emulate
```

**Hypercall table:** The Xen hypervisor exposes a set of hypercall functions at a known memory address. The modified guest kernel calls these directly using `SYSCALL` or `INT` — no hardware trap overhead.

---

## 6. QEMU Device Emulation — How VMs Get NICs and Disks

QEMU emulates all peripheral devices that the guest OS sees. This is how an EC2 instance gets a "NIC" and "disk" without access to real hardware.

```mermaid
graph LR
    subgraph Guest_VM["Guest VM"]
        GuestDriver["Guest Kernel<br/>virtio-net driver<br/>Sees: eth0, 1Gbps NIC"]
    end

    subgraph QEMU["QEMU Process (User Space)"]
        VirtIO["virtio backend<br/>Shared memory ring buffers<br/>vring_avail + vring_used"]
        TAP["TAP device<br/>/dev/tap0<br/>Layer 2 virtual NIC"]
    end

    subgraph Host_Net["Host Network"]
        OVS["Open vSwitch<br/>or Linux Bridge"]
        PhysNIC["Physical NIC"]
    end

    GuestDriver <-->|virtio shared memory| VirtIO
    VirtIO <-->|read/write frames| TAP
    TAP <-->|Ethernet frames| OVS
    OVS <-->|802.3 frames| PhysNIC
```

**virtio vs full emulation:**
- Full emulation: QEMU emulates Intel e1000 NIC register-by-register. Every guest I/O = VMEXIT + QEMU handling.
- virtio: Shared memory ring buffers. Guest writes packet descriptor to ring. QEMU picks it up without VMEXIT. ~3x faster.

---

## 7. Live VM Migration — Internals

How AWS moves your EC2 instance between physical hosts with ~50ms downtime:

```mermaid
sequenceDiagram
    participant SrcHost as Source Host
    participant DstHost as Destination Host
    participant Network as Network

    Note over SrcHost: VM running, serving requests

    SrcHost->>DstHost: Phase 1: Pre-copy memory
    loop Copy all dirty pages
        SrcHost->>Network: Send memory page
        Network->>DstHost: Receive page
        Note over SrcHost: VM still running, generating new dirty pages
    end

    Note over SrcHost,DstHost: Phase 2: Iterative copy (dirty pages shrinking)
    SrcHost->>DstHost: Copy pages dirtied during Phase 1

    Note over SrcHost: Phase 3: Stop-and-copy (brief pause)
    SrcHost->>SrcHost: Pause VM (~50ms)
    SrcHost->>DstHost: Copy final dirty pages + CPU state
    SrcHost->>DstHost: Transfer VMCS state (registers, vCPU state)

    Note over DstHost: Phase 4: Resume
    DstHost->>DstHost: VMRESUME on destination
    DstHost->>Network: Gratuitous ARP (update MAC→IP mapping)
    Note over DstHost: VM running on new host, same IP, same state
```

**Why only ~50ms pause?** The final stop-and-copy only transfers the pages dirtied in the last iteration — typically a few MB, which takes milliseconds at LAN speeds.

---

## 8. Failure Scenarios

### Hypervisor Crash — What Happens to VMs?

If KVM/QEMU process crashes on one host, ALL VMs on that host die. This is why:
- AWS spreads your ASG across multiple AZs (different physical hosts)
- RDS Multi-AZ uses synchronous replication to standby in a different host
- EBS stores data independently of the EC2 instance

### VM Escape — The Critical Security Failure

```mermaid
graph TB
    Attacker["Attacker in Guest VM"]
    Attacker -->|"Exploit QEMU vulnerability<br/>(e.g., heap overflow in virtio handler)"| QEMU["QEMU Process<br/>Running as root or with high privileges"]
    QEMU -->|"Escape to host OS"| Host["Host OS<br/>Hypervisor Level"]
    Host -->|"Access other VMs' memory"| OtherVMs["Other Customers' VMs<br/>on same physical host"]
```

**Real examples:** VENOM (2015) — QEMU floppy drive vulnerability allowed VM escape. CVE-2019-14835 (vhost-net buffer overflow). AWS Nitro was partly designed to shrink the attack surface by moving device emulation to dedicated hardware (no QEMU).

---

## 9. Trade-offs Summary

| Technology | Benefit | Cost | Use When |
|-----------|---------|------|----------|
| Full virt (no HW assist) | Works on any CPU | High VMEXIT overhead | Legacy hardware |
| HW-assisted (VT-x/AMD-V) | Near-native CPU speed | Still I/O VMEXIT | Modern VMs (default) |
| Para-virt (Xen PV) | Fast I/O + memory | Modified guest OS | Old Xen deployments |
| virtio | Fast I/O (no VMEXIT for NIC/disk) | Guest needs virtio drivers | All modern Linux VMs |
| EPT/NPT | Fast memory translation | Slightly larger VMEXIT for EPT faults | All modern x86 |
| Containers | Near-native, low overhead | Weak isolation, shared kernel | Trusted workloads |
