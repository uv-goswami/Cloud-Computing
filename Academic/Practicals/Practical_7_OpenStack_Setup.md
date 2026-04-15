# Practical 7 — Install OpenStack
**Objective:** Set up a local OpenStack environment for private cloud practice.

---

## All Terms You Will Encounter

| Term | Definition |
|------|-----------|
| **OpenStack** | Open-source cloud computing platform for building private/public clouds |
| **DevStack** | Development version of OpenStack — easiest way to install for learning |
| **Controller Node** | OpenStack node running management services (Keystone, Glance, Nova-API) |
| **Compute Node** | Node running actual VMs (Nova-Compute) |
| **All-in-One** | Single machine running all OpenStack services (DevStack default) |
| **Keystone** | OpenStack Identity Service — authentication and authorization |
| **Nova** | OpenStack Compute Service — manages VM lifecycle |
| **Neutron** | OpenStack Networking Service — virtual networks |
| **Glance** | OpenStack Image Service — stores VM images |
| **Cinder** | OpenStack Block Storage Service — persistent volumes |
| **Swift** | OpenStack Object Storage (like AWS S3) |
| **Horizon** | OpenStack Dashboard — web GUI (like AWS Management Console) |
| **Heat** | OpenStack Orchestration (like AWS CloudFormation) |
| **KVM** | Kernel-based Virtual Machine — hypervisor OpenStack uses by default |
| **QEMU** | Machine emulator used with KVM |
| **libvirt** | API for managing virtualization (KVM, QEMU) |
| **Token** | Temporary credential issued by Keystone after authentication |
| **Project/Tenant** | Isolated organizational unit in OpenStack (like AWS account) |
| **local.conf** | DevStack configuration file |

---

## OpenStack Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                   OPENSTACK SERVICES MAP                             │
│                                                                      │
│  USER INTERFACES:                                                    │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │  Horizon (Dashboard — Web GUI)  |  CLI (openstack commands)   │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                       │
│  ┌───────────────────────────▼───────────────────────────────────┐  │
│  │  Keystone (Identity) — Authentication for all services       │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                              │                                       │
│  ┌───────────┬───────────────┼───────────────┬───────────────────┐  │
│  ▼           ▼               ▼               ▼                   │  │
│ Nova        Glance        Neutron          Cinder                 │  │
│ (Compute)   (Images)      (Network)        (Block Storage)        │  │
│  │                          │                                    │  │
│  └──────────────────────────┘                                    │  │
│                 │                                                 │  │
│  ┌──────────────▼──────────────────────────────────────────────┐ │  │
│  │  Hypervisor Layer: KVM + libvirt + QEMU                     │ │  │
│  └──────────────┬──────────────────────────────────────────────┘ │  │
│                 │                                                 │  │
│  ┌──────────────▼──────────────────────────────────────────────┐ │  │
│  │  Physical Hardware (or AWS EC2 — nested virtualization)     │ │  │
│  └─────────────────────────────────────────────────────────────┘ │  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Installation Methods

```
┌──────────────────────────────────────────────────────────────────┐
│            OPENSTACK INSTALLATION OPTIONS                        │
├──────────────────┬───────────────────────────────────────────────┤
│ DevStack         │ Best for learning. Single script. NOT for     │
│                  │ production. Auto-installs all services.       │
├──────────────────┼───────────────────────────────────────────────┤
│ PackStack        │ Ansible-based. Red Hat/CentOS. Faster setup.  │
├──────────────────┼───────────────────────────────────────────────┤
│ Kolla-Ansible    │ Docker containers. Production-grade.          │
├──────────────────┼───────────────────────────────────────────────┤
│ MicroStack       │ Snap package. Ubuntu only. Very quick.        │
└──────────────────┴───────────────────────────────────────────────┘
```

---

## Step-by-Step: DevStack Installation

### Requirements
- Ubuntu 22.04 LTS (VM or bare metal)
- Minimum: 4GB RAM, 2 vCPUs, 20GB disk
- Internet access
- Recommended: Use an AWS EC2 t3.medium (2 vCPU, 4GB) with Ubuntu 22.04

**Note on nested virtualization:** Running OpenStack VMs inside AWS EC2 requires nested virtualization. AWS enables this on some instance types. For DevStack learning, you can run QEMU in "emulation mode" (slower but works).

### Step 1: Prepare the System

```bash
# Update Ubuntu
sudo apt update && sudo apt upgrade -y

# Create a dedicated user for DevStack (don't run as root)
sudo useradd -s /bin/bash -d /opt/stack -m stack
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack

# Switch to stack user
sudo su - stack
```

### Step 2: Clone DevStack

```bash
git clone https://opendev.org/openstack/devstack
cd devstack
```

### Step 3: Create Configuration File

```bash
cat > local.conf << 'EOF'
[[local|localrc]]
ADMIN_PASSWORD=secret123
DATABASE_PASSWORD=secret123
RABBIT_PASSWORD=secret123
SERVICE_PASSWORD=secret123

# Disable services we don't need for learning
disable_service tempest

# Enable Horizon dashboard
enable_service horizon

# Log file location
LOGFILE=/opt/stack/logs/stack.log
LOG_COLOR=True
EOF
```

### Step 4: Run Installation

```bash
./stack.sh
# This takes 20-40 minutes
# Downloads and configures all services
# Watch output for any errors
```

### Step 5: Verify Installation

```bash
# Source credentials
source openrc admin admin

# List services
openstack service list

# Expected output:
# +----+----------+----------+
# | ID | Name     | Type     |
# +----+----------+----------+
# | .. | keystone | identity |
# | .. | nova     | compute  |
# | .. | neutron  | network  |
# | .. | glance   | image    |
# | .. | cinder   | volume   |
# +----+----------+----------+
```

### Step 6: Access Horizon Dashboard

Open browser: `http://YOUR_MACHINE_IP/dashboard`
- Username: `admin`
- Password: `secret123`
- Domain: `Default`

---

## Viva Questions — Practical 7

1. **What is OpenStack? How is it different from AWS?**  
   A: Open-source cloud platform for building your own cloud. AWS is a commercial cloud service (you pay Amazon). OpenStack runs on your own hardware — you own and operate it. OpenStack = private cloud; AWS = public cloud.

2. **Name 5 OpenStack services and their AWS equivalents.**  
   A: Nova (EC2), Glance (AMI registry), Neutron (VPC), Cinder (EBS), Swift (S3), Keystone (IAM), Horizon (AWS Management Console), Heat (CloudFormation).

3. **What is Keystone?**  
   A: OpenStack's identity service. Handles authentication (who are you?) and authorization (what can you do?). Issues tokens for API access. Every OpenStack service calls Keystone to validate requests.

4. **What is DevStack? Is it suitable for production?**  
   A: A set of scripts to quickly deploy OpenStack on a single machine for development/testing. NOT suitable for production — it has no HA, no proper security, and the stack.sh script can break on system changes.

5. **What is nested virtualization?**  
   A: Running a hypervisor (OpenStack + KVM) inside a virtual machine (AWS EC2). The VM is already virtualized, so you're virtualizing within virtualization. Requires CPU support (Intel VT-x with VMCS shadowing, AMD-V with nested page tables).

---
