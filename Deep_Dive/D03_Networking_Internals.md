# D03 — Networking Internals
**Track: Deep Dive | How a packet travels from your laptop to EC2 and back**

---

## 1. VPC Architecture — Not Just a Diagram, The Actual Stack

A VPC is not a separate physical network. It is an **overlay network** implemented in software on top of AWS's physical network.

```mermaid
graph TB
    subgraph Physical_Reality["Physical Reality (What AWS Actually Has)"]
        Switch1["Top-of-Rack Switch"]
        Switch2["Aggregate Switch"]
        Backbone["AWS Backbone Fiber"]
        Host1["Physical Host 1<br/>(many EC2 instances)"]
        Host2["Physical Host 2<br/>(many EC2 instances)"]
    end

    subgraph Virtual_Overlay["Virtual Overlay (What VPC Looks Like to You)"]
        VPC["Your VPC: 10.0.0.0/16"]
        Subnet1["Public Subnet 10.0.1.0/24"]
        Subnet2["Private Subnet 10.0.2.0/24"]
        EC2_A["EC2 at 10.0.1.5<br/>(on Host 1 physically)"]
        EC2_B["EC2 at 10.0.2.8<br/>(on Host 2 physically)"]
    end

    EC2_A -->|"Packet to 10.0.2.8<br/>Looks like same LAN"| EC2_B
    Host1 -->|"Actual path: VXLAN encapsulation<br/>10.0.1.5 → 10.0.2.8 wrapped in<br/>Host1_IP → Host2_IP UDP packet"| Backbone --> Host2
```

**The overlay trick:** When EC2_A at 10.0.1.5 sends a packet to EC2_B at 10.0.2.8, the Nitro card encapsulates the entire Ethernet frame inside a VXLAN UDP packet addressed from Host1's physical IP to Host2's physical IP. The physical network only sees host-to-host traffic. The VPC overlay is invisible to the physical switches.

---

## 2. VXLAN — The Encapsulation Protocol

VXLAN (Virtual Extensible LAN) is how AWS implements VPC overlay networks.

```mermaid
graph LR
    subgraph Original_Frame["Original Layer-2 Frame (Customer Traffic)"]
        InEth["Ethernet Header<br/>Src MAC: VM_A_MAC<br/>Dst MAC: VM_B_MAC"]
        InIP["IP Header<br/>Src: 10.0.1.5<br/>Dst: 10.0.2.8"]
        InPayload["TCP Payload<br/>(HTTP request)"]
    end

    subgraph VXLAN_Wrapper["VXLAN Encapsulated Packet (Physical Network Sees This)"]
        OutEth["Outer Ethernet<br/>Src: Host1_MAC<br/>Dst: Host2_MAC"]
        OutIP["Outer IP Header<br/>Src: Host1_IP (physical)<br/>Dst: Host2_IP (physical)"]
        OutUDP["UDP Header<br/>Dst Port: 4789<br/>(VXLAN port)"]
        VXLAN_Hdr["VXLAN Header<br/>VNI: 12345 (tenant ID)"]
        InEth & InIP & InPayload
    end
```

**VNI (VXLAN Network Identifier):** 24-bit field — supports up to 16 million distinct virtual networks. Each VPC gets a VNI. Traffic in VPC A with VNI=100 cannot be delivered to VPC B with VNI=200 — the physical switches would just forward the outer packet; the Nitro card on the destination host checks VNI and only delivers to the correct VM.

---

## 3. Packet Flow — Internet → Your EC2

This is the complete path of an HTTP request from a browser in Delhi to your EC2 in Mumbai (ap-south-1).

```mermaid
sequenceDiagram
    participant Browser as Browser (Delhi)
    participant DNS as Route 53 DNS
    participant CF as CloudFront Edge (Delhi PoP)
    participant IGW as Internet Gateway (AWS)
    participant NitroNW as Nitro Network Card (EC2 Host)
    participant SG as Security Group (enforced by Nitro)
    participant EC2 as Your EC2 Instance

    Browser->>DNS: Resolve api.myapp.com
    DNS->>Browser: IP = 13.235.x.x (ALB or EC2 public IP)

    Browser->>CF: TCP SYN to 13.235.x.x
    Note over CF: TLS termination if CloudFront configured
    CF->>IGW: Forward to VPC

    IGW->>NitroNW: Packet arrives at AWS backbone edge
    Note over NitroNW: VXLAN encapsulate: outer dst = EC2's physical host IP
    NitroNW->>SG: Check inbound Security Group rules
    Note over SG: Port 80/443 allowed from 0.0.0.0/0? YES
    SG->>EC2: Deliver inner packet to VM
    EC2->>EC2: Application processes request

    EC2->>SG: Response packet (src: 10.0.1.5, dst: browser_IP)
    SG->>NitroNW: Stateful: return traffic auto-allowed
    NitroNW->>IGW: VXLAN decapsulate, send to IGW
    IGW->>Browser: Response
```

---

## 4. Security Groups — How They're Implemented

Security Groups are NOT iptables rules in the guest OS. They are enforced at the **Nitro card level** — outside the VM.

```mermaid
graph TB
    subgraph EC2_Host["Physical EC2 Host"]
        subgraph VM["Guest VM"]
            App["Application<br/>Kernel<br/>Network Stack"]
        end

        subgraph NitroCard["Nitro Card (Hardware)"]
            SG_Engine["Security Group Engine<br/>Hardware connection tracker<br/>Enforces rules BEFORE VM sees packet"]
            VPC_Logic["VPC Routing + VXLAN<br/>Encap/Decap"]
        end
    end

    Internet["Internet / Other VPCs"] --> VPC_Logic
    VPC_Logic --> SG_Engine
    SG_Engine -->|"Allowed traffic only"| VM
    VM --> SG_Engine
    SG_Engine -->|"Stateful: return traffic auto-allowed"| VPC_Logic
```

**Why this matters for security:** The VM cannot bypass Security Groups by modifying its own iptables. Security Groups are enforced at the hypervisor/hardware level. Even if the guest OS is fully compromised, it cannot receive traffic not allowed by the Security Group.

**Stateful implementation:** The Nitro card maintains a connection tracking table (similar to Linux conntrack). When an inbound connection is established, the (src_ip, src_port, dst_ip, dst_port, protocol) tuple is stored. Return traffic matching this tuple is automatically allowed without a separate outbound rule evaluation.

---

## 5. VPC Routing — How Subnets Actually Route

```mermaid
graph TB
    subgraph VPC_Router["VPC Router (Implicit, Always Exists at x.x.x.1)"]
        Router_Logic["Route Table Lookup<br/>Most specific prefix wins"]
    end

    subgraph Route_Table_Public["Public Subnet Route Table"]
        R1P["10.0.0.0/16 → local"]
        R2P["0.0.0.0/0 → igw-xxx"]
    end

    subgraph Route_Table_Private["Private Subnet Route Table"]
        R1Pr["10.0.0.0/16 → local"]
        R2Pr["0.0.0.0/0 → nat-xxx"]
    end

    EC2_Public["EC2 in Public Subnet<br/>sends to 8.8.8.8"]
    EC2_Private["EC2 in Private Subnet<br/>sends to 8.8.8.8"]

    EC2_Public -->|"Default route"| Route_Table_Public
    Route_Table_Public -->|"0.0.0.0/0 matches"| IGW["Internet Gateway"]
    IGW --> Internet

    EC2_Private -->|"Default route"| Route_Table_Private
    Route_Table_Private -->|"0.0.0.0/0 matches"| NATGW["NAT Gateway<br/>(in public subnet)<br/>Translates src IP to Elastic IP"]
    NATGW -->|"via IGW"| Internet
```

**Longest prefix match:** If a route table has both `10.0.0.0/16 → local` and `10.0.1.0/24 → transit-gw`, a packet to `10.0.1.5` matches BOTH. The `/24` is more specific → transit-gw wins.

---

## 6. NAT Gateway Internals

NAT Gateway is not a simple IP masquerade. It is an AWS-managed network appliance:

```mermaid
sequenceDiagram
    participant PrivateEC2 as EC2 (10.0.2.5)
    participant NAT as NAT Gateway (10.0.1.100, EIP: 54.x.x.x)
    participant Internet as Internet Server (1.2.3.4)

    PrivateEC2->>NAT: Packet: src=10.0.2.5:54321, dst=1.2.3.4:443
    Note over NAT: NAPT (Network Address Port Translation)<br/>src=10.0.2.5:54321 → 54.x.x.x:12345<br/>Store mapping in NAT table
    NAT->>Internet: Packet: src=54.x.x.x:12345, dst=1.2.3.4:443

    Internet->>NAT: Response: src=1.2.3.4:443, dst=54.x.x.x:12345
    Note over NAT: Lookup NAT table: 54.x.x.x:12345 → 10.0.2.5:54321
    NAT->>PrivateEC2: Packet: src=1.2.3.4:443, dst=10.0.2.5:54321
```

**Why private subnet instances can initiate but not receive:** The NAT table entry is created only when the private instance initiates the connection. There's no entry for an internet server initiating to the NAT public IP — it has nowhere to forward it (no port mapping exists). This is the "outbound only" property.

---

## 7. OpenStack Neutron Internals — ML2 Plugin and OVS

```mermaid
graph TB
    subgraph Neutron_Architecture["OpenStack Neutron Architecture"]
        NeutronServer["Neutron Server<br/>API + Plugin Manager"]

        subgraph ML2["ML2 Plugin (Modular Layer 2)"]
            TypeDriver["Type Driver<br/>VXLAN / VLAN / Flat / GRE"]
            MechDriver["Mechanism Driver<br/>OVS / LinuxBridge / SR-IOV"]
        end

        subgraph DHCP["DHCP Agent"]
            Dnsmasq["dnsmasq<br/>One per subnet<br/>Runs in network namespace"]
        end

        subgraph L3["L3 Agent"]
            qrouter["qrouter namespace<br/>iptables NAT for floating IPs<br/>Routing between networks"]
        end

        subgraph OVS_Bridge["Open vSwitch on Compute Node"]
            br_int["br-int<br/>Integration bridge<br/>All VMs connect here"]
            br_tun["br-tun<br/>Tunnel bridge<br/>VXLAN endpoints"]
            br_ex["br-ex<br/>External bridge<br/>Physical NIC connected"]
        end
    end

    NeutronServer --> ML2
    ML2 --> OVS_Bridge
    DHCP & L3 --> OVS_Bridge
    br_int <--> br_tun <--> br_ex
```

**Packet path in OpenStack Neutron:**
1. VM sends packet → TAP device → br-int (local switching)
2. br-int → br-tun (VXLAN encapsulation with VNI)
3. VXLAN packet traverses physical network
4. Destination host: br-tun decapsulates → br-int → TAP → destination VM

---

## 8. DNS Resolution Inside a VPC

```mermaid
sequenceDiagram
    participant EC2 as EC2 Instance
    participant Resolver as VPC DNS Resolver (169.254.169.253)
    participant PHZ as Private Hosted Zone (Route 53)
    participant Public as Public DNS

    EC2->>Resolver: Query: api.internal.myapp.com
    Resolver->>PHZ: Check private hosted zone for myapp.com
    PHZ->>Resolver: A record: 10.0.3.45
    Resolver->>EC2: 10.0.3.45

    EC2->>Resolver: Query: api.github.com
    Resolver->>PHZ: Not in private zone
    Resolver->>Public: Forward to public DNS
    Public->>Resolver: A record: 140.82.121.4
    Resolver->>EC2: 140.82.121.4
```

**VPC DNS resolver:** Every VPC has a built-in DNS resolver at the 2nd IP of the VPC CIDR (e.g., 10.0.0.2 for 10.0.0.0/16), also reachable at 169.254.169.253. It handles Route 53 Private Hosted Zones and forwards public DNS queries to AWS public resolvers.

---

## 9. Network Performance Limits and Trade-offs

| Feature | Mechanism | Limit | Trade-off |
|---------|-----------|-------|-----------|
| Enhanced Networking (ENA) | SR-IOV + virtio-net | Up to 100Gbps | Requires ENA driver in guest |
| Placement Groups (cluster) | Same physical rack | Lowest latency | Single AZ, hardware failure risk |
| Placement Groups (spread) | Different racks | Highest HA | Max 7 instances per AZ |
| Jumbo frames | MTU 9001 within VPC | ~15% throughput gain | Only works within VPC, not internet |
| NAT Gateway bandwidth | Managed scaling | 45Gbps burst | $0.045/GB data processed |
| Internet Gateway | No bandwidth limit | AWS throttles abusers | Egress: $0.09/GB to internet |
