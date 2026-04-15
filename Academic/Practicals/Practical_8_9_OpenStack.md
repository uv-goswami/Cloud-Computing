# Practical 8 — Launch Your First OpenStack Instance
**Objective:** Create a VM using OpenStack (project setup, image upload, flavor, instance launch).

---

## All Terms You Will Encounter

| Term | Definition |
|------|-----------|
| **Project (Tenant)** | Isolated organizational unit in OpenStack. VMs, networks, quotas are per-project |
| **User** | Identity in Keystone. Belongs to projects with assigned roles |
| **Role** | Permission set in OpenStack: admin (full access), member (normal user), reader (read-only) |
| **Image** | VM disk snapshot stored in Glance (equivalent to AWS AMI) |
| **Flavor** | VM hardware profile: vCPUs, RAM, disk (equivalent to AWS instance type) |
| **Instance** | A running VM in OpenStack (equivalent to AWS EC2 instance) |
| **Key Pair** | SSH authentication: public key injected into VM, private key you hold |
| **Security Group** | Virtual firewall for OpenStack instances (same concept as AWS) |
| **Floating IP** | Public IP assigned to instance (equivalent to AWS Elastic IP) |
| **Console** | Web-based terminal to VM via Horizon (VNC console) |
| **Metadata** | Key-value pairs attached to instances, similar to AWS EC2 tags |
| **QCOW2** | QEMU Copy-On-Write v2 — standard disk image format for OpenStack |
| **Glance** | OpenStack image service |
| **Nova** | OpenStack compute service (manages VMs) |
| **Horizon** | OpenStack web dashboard |

---

## OpenStack VM Launch Flow

```
┌──────────────────────────────────────────────────────────────────────┐
│                OPENSTACK VM CREATION FLOW                            │
│                                                                      │
│  1. User authenticates → Keystone issues Token                      │
│                                                                      │
│  2. User sends: "Launch instance with image X, flavor m1.small"     │
│     → Nova-API receives request (validates token with Keystone)     │
│                                                                      │
│  3. Nova-API → Nova-Scheduler: "Which compute node has capacity?"   │
│     Nova-Scheduler: Filters nodes by RAM, CPU, disk requirements    │
│     Returns: "Use Compute Node 2"                                    │
│                                                                      │
│  4. Nova-API → Nova-Compute (on Node 2):                            │
│     "Launch VM with these specs"                                     │
│                                                                      │
│  5. Nova-Compute → Glance: "Give me image X"                        │
│     Glance streams image to Compute Node                            │
│                                                                      │
│  6. Nova-Compute → Neutron: "Create network port for this VM"       │
│     Neutron creates virtual NIC, assigns IP                         │
│                                                                      │
│  7. Nova-Compute → libvirt → KVM: "Start VM with this config"       │
│     KVM creates VM, boots OS from image                             │
│                                                                      │
│  8. VM is ACTIVE. User can SSH or use console.                      │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Step-by-Step

### Step 1: Create a Project

```bash
# Source admin credentials
source /opt/stack/devstack/openrc admin admin

# Create a new project
openstack project create --description "My test project" myproject

# Create a user
openstack user create --password mypass123 myuser

# Assign role to user in project
openstack role add --project myproject --user myuser member

# Verify
openstack project list
openstack user list
```

**OR via Horizon:**
Identity → Projects → Create Project → name: `myproject`
Identity → Users → Create User → username: `myuser`, assign to `myproject`

---

### Step 2: Upload an Image to Glance

```bash
# Download Ubuntu Cloud Image (small, cloud-optimized)
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

# Upload to Glance
openstack image create "Ubuntu-22.04" \
  --file jammy-server-cloudimg-amd64.img \
  --disk-format qcow2 \
  --container-format bare \
  --public

# Verify
openstack image list
```

**What QCOW2 means:** QEMU Copy-On-Write v2. The image file only stores differences from a base — saves disk space. When OpenStack launches a VM, it creates a copy-on-write layer on top of the base image; the base stays unchanged and can be shared by multiple VMs.

---

### Step 3: Define a Flavor

```bash
# Create a flavor (VM hardware profile)
openstack flavor create --vcpus 1 --ram 512 --disk 5 m1.tiny

# View existing flavors
openstack flavor list

# Expected:
# m1.tiny:  1 vCPU, 512MB RAM,  5GB disk
# m1.small: 1 vCPU, 2048MB RAM, 20GB disk
# m1.medium: 2 vCPU, 4096MB RAM, 40GB disk
```

---

### Step 4: Create Key Pair

```bash
# Generate key pair
openstack keypair create mykey > mykey.pem
chmod 400 mykey.pem

# Verify
openstack keypair list
```

---

### Step 5: Create Security Group

```bash
# Create security group
openstack security group create web-sg --description "Allow SSH and HTTP"

# Add SSH rule
openstack security group rule create web-sg \
  --protocol tcp \
  --dst-port 22 \
  --remote-ip 0.0.0.0/0

# Add HTTP rule
openstack security group rule create web-sg \
  --protocol tcp \
  --dst-port 80 \
  --remote-ip 0.0.0.0/0

# Add ICMP (ping)
openstack security group rule create web-sg \
  --protocol icmp \
  --remote-ip 0.0.0.0/0
```

---

### Step 6: Launch Instance

```bash
# Get the network ID
NETWORK_ID=$(openstack network list | grep private | awk '{print $2}')

# Launch instance
openstack server create \
  --flavor m1.tiny \
  --image "Ubuntu-22.04" \
  --key-name mykey \
  --security-group web-sg \
  --network $NETWORK_ID \
  my-first-vm

# Watch status
openstack server show my-first-vm
# Wait for status: ACTIVE
```

---

### Step 7: Access via Console (Horizon)

1. Horizon → Project → Compute → Instances
2. Click your instance name
3. Console tab → Click inside the console window
4. Login with Ubuntu credentials (cloud images use key auth, not password)

---

## Viva Questions — Practical 8

1. **What is Nova? What is Nova-Scheduler?**  
   A: Nova = OpenStack compute service, manages VM lifecycle. Nova-Scheduler = component that decides which compute node gets the new VM, based on filters (RAM, CPU, disk availability, affinity rules).

2. **What is a flavor in OpenStack? What is the AWS equivalent?**  
   A: Pre-defined VM hardware specification (vCPUs, RAM, disk). AWS equivalent = instance type (t2.micro, m5.large).

3. **What is QCOW2?**  
   A: QEMU Copy-On-Write v2 — disk image format. Copy-on-write means the base image is read-only; writes go to a separate "delta" file. Multiple VMs can share one base image; each gets its own delta layer.

4. **What is the difference between stopping and deleting an instance in OpenStack?**  
   A: Stop (Shelve/Suspend) = instance halted, disk preserved, resources partially freed. Delete = instance and its local disk destroyed. Persistent volumes (Cinder) survive deletion if not attached-and-deleted simultaneously.

5. **What role does Glance play in VM launch?**  
   A: Stores VM images (boot disks). When Nova-Compute launches a VM, it requests the image from Glance, downloads it to the compute node, and creates a bootable disk. Glance = image registry and delivery service.

---

---

# Practical 9 — Set Up OpenStack Networking (Neutron)
**Objective:** Configure OpenStack Neutron networks, routers, and floating IPs for external access.

---

## All Terms You Will Encounter

| Term | Definition |
|------|-----------|
| **Neutron** | OpenStack Networking Service — creates/manages virtual networks |
| **Network** | Virtual Layer-2 broadcast domain in OpenStack |
| **Subnet** | IP address range within a network (like AWS subnet) |
| **Router** | Virtual router connecting networks (like AWS route table + NAT) |
| **Floating IP** | Public IP that can be associated with a VM (like AWS Elastic IP) |
| **Fixed IP** | Private IP permanently assigned to VM's virtual NIC |
| **External Network** | Network with external connectivity (maps to physical network) |
| **Provider Network** | Network mapped to physical network infrastructure |
| **Tenant Network** | Private network created by project for its VMs |
| **vSwitch (OVS)** | Open vSwitch — software virtual switch Neutron uses |
| **ML2 Plugin** | Neutron's Modular Layer 2 plugin — abstracts different switching technologies |
| **VXLAN** | Virtual Extensible LAN — tunneling protocol for overlay networks |
| **Security Group** | Instance-level firewall (same as AWS Security Group) |
| **Port** | Virtual NIC connection point in Neutron network |
| **DHCP** | Protocol for automatic IP assignment (Neutron runs a DHCP agent) |

---

## Neutron Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│               OPENSTACK NEUTRON ARCHITECTURE                         │
│                                                                      │
│  PHYSICAL WORLD:                                                     │
│  Internet ────── Physical Router ────── Physical Switch              │
│                                               │                      │
│                                          Physical Host               │
│                                               │                      │
│  VIRTUAL WORLD (Neutron manages this):        │                      │
│                                               │                      │
│  ┌────────────────────────────────────────────▼──────────────────┐  │
│  │  Open vSwitch (br-ex) ← External bridge (connects to physical)│  │
│  │              │                                                 │  │
│  │         OVS Tunnels (VXLAN)                                   │  │
│  │              │                                                 │  │
│  │  ┌───────────▼────────────────────────────────────────────┐   │  │
│  │  │  Virtual Router (qrouter)                              │   │  │
│  │  │  • NAT: Floating IP ↔ Fixed IP                        │   │  │
│  │  │  • Routes between tenant network and external network │   │  │
│  │  └────────────────────────────────────────────────────────┘   │  │
│  │              │                                                 │  │
│  │  ┌───────────▼────────────────────────────────────────────┐   │  │
│  │  │  Tenant Network (private)  10.0.0.0/24                 │   │  │
│  │  │  DHCP Agent (dnsmasq) assigns IPs to VMs               │   │  │
│  │  │                                                        │   │  │
│  │  │   VM1 (10.0.0.5) ──── VM2 (10.0.0.6)                 │   │  │
│  │  └────────────────────────────────────────────────────────┘   │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Step-by-Step

### Step 1: Create Private (Tenant) Network

```bash
source /opt/stack/devstack/openrc myuser myproject

# Create private network
openstack network create private-net

# Create subnet on this network
openstack subnet create private-subnet \
  --network private-net \
  --subnet-range 10.0.10.0/24 \
  --dns-nameserver 8.8.8.8

# Verify
openstack network list
openstack subnet list
```

---

### Step 2: Create Public (External) Network

```bash
# Must be admin to create external network
source /opt/stack/devstack/openrc admin admin

# DevStack creates this automatically as "public"
# Verify it exists:
openstack network list --external

# If needed, create manually:
openstack network create \
  --external \
  --provider-physical-network public \
  --provider-network-type flat \
  external-net

openstack subnet create external-subnet \
  --network external-net \
  --subnet-range 172.24.4.0/24 \
  --allocation-pool start=172.24.4.10,end=172.24.4.200 \
  --no-dhcp
```

---

### Step 3: Create Router and Connect Networks

```bash
source /opt/stack/devstack/openrc myuser myproject

# Create router
openstack router create my-router

# Set external gateway (connect router to external network)
openstack router set my-router --external-gateway public

# Add private subnet interface to router
openstack router add subnet my-router private-subnet

# Verify router ports
openstack port list --router my-router
```

**What this does:** The router has two "legs":
1. One leg connected to external network (has a public IP)
2. One leg connected to private network (acts as gateway for VMs)
Traffic from private VMs → router → NAT → external network → internet.

---

### Step 4: Launch Instance in Private Network

```bash
# Launch VM in our private network
openstack server create \
  --flavor m1.tiny \
  --image "Ubuntu-22.04" \
  --key-name mykey \
  --security-group web-sg \
  --network private-net \
  my-networked-vm

# Get instance details
openstack server show my-networked-vm
# Note the fixed IP: 10.0.10.X
```

---

### Step 5: Assign Floating IP

```bash
# Allocate a floating IP from the external pool
openstack floating ip create public

# List floating IPs
openstack floating ip list
# Note the floating IP: 172.24.4.X

# Associate floating IP with instance
openstack server add floating ip my-networked-vm 172.24.4.X

# Verify
openstack server show my-networked-vm
# Now shows: 10.0.10.X, 172.24.4.X
```

---

### Step 6: SSH via Floating IP

```bash
ssh -i mykey.pem ubuntu@172.24.4.X

# From inside VM, test internet connectivity
ping 8.8.8.8          # Should reach Google DNS
curl http://ipinfo.io  # Shows your external IP
```

---

## Viva Questions — Practical 9

1. **What is the difference between a fixed IP and a floating IP in OpenStack?**  
   A: Fixed IP = private IP permanently assigned to VM's NIC from the subnet (like AWS private IP, assigned by DHCP agent). Floating IP = public IP from external pool that can be associated/disassociated with any VM (like AWS Elastic IP). NAT maps floating IP to fixed IP.

2. **What is Open vSwitch (OVS)?**  
   A: Software-defined virtual switch that Neutron uses by default. Runs on the Linux host, creates virtual switch ports for VMs. Supports VXLAN tunneling to connect VMs across different physical hosts. Like a smart software-controlled switch inside the server.

3. **Why does the router need two interfaces?**  
   A: One interface connects to the external (public) network — this is how traffic exits to internet. One interface connects to tenant (private) network — this is the default gateway for private VMs. The router NATs between them.

4. **What is VXLAN used for in OpenStack Neutron?**  
   A: Virtual Extensible LAN — tunneling protocol for creating overlay networks. Encapsulates Layer 2 frames inside UDP packets, allowing VMs on different physical hosts to be in the same virtual network. Extends tenant networks across physical hosts transparently.

5. **What happens to the floating IP if you delete the instance?**  
   A: Floating IP is disassociated and returned to the pool (still allocated to the project). You must explicitly `openstack floating ip delete` to release it. Allocated floating IPs are billed even when unassigned.

6. **What is the DHCP Agent in Neutron?**  
   A: Neutron runs a DHCP server (dnsmasq) for each tenant subnet. When a VM boots, it sends DHCP Discover; the Neutron DHCP agent responds with the configured fixed IP, gateway, and DNS. Similar to how a router in your home assigns IPs to devices.

---
