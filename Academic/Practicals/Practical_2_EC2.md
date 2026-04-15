# Practical 2 — Launch Your First Amazon EC2 Instance
**Objective:** Deploy a virtual machine on AWS, configure security, and connect via SSH.

---

## All Terms You Will Encounter

| Term | Definition |
|------|-----------|
| **EC2** | Elastic Compute Cloud — AWS's virtual machine service |
| **Instance** | A running virtual machine in EC2 |
| **AMI** | Amazon Machine Image — OS template for launching instances |
| **Instance Type** | Hardware profile (vCPUs, RAM, storage) e.g., t2.micro |
| **vCPU** | Virtual CPU — a virtual processor core |
| **Key Pair** | SSH authentication: public key on AWS, private key (.pem) you download |
| **Security Group** | Virtual firewall controlling inbound/outbound traffic for instance |
| **Public IP** | Internet-accessible IP assigned to instance (changes on restart) |
| **Elastic IP** | Static public IP that persists even when instance stops |
| **SSH** | Secure Shell — encrypted protocol for remote command-line access |
| **Inbound Rules** | Rules for traffic ENTERING the instance |
| **Outbound Rules** | Rules for traffic LEAVING the instance |
| **Port 22** | Default SSH port |
| **Port 80** | HTTP web traffic |
| **Port 443** | HTTPS encrypted web traffic |
| **0.0.0.0/0** | Anywhere on the internet (all IP addresses) |
| **EBS Volume** | The virtual hard disk attached to your EC2 instance |
| **User Data** | Startup script that runs when instance first launches |
| **Instance State** | Running, Stopped, Terminated |
| **Stop vs Terminate** | Stop = save and pause (EBS persists). Terminate = delete everything |

---

## EC2 Architecture — What You're Actually Building

```
┌──────────────────────────────────────────────────────────────────┐
│                    YOUR EC2 SETUP                                │
│                                                                  │
│  YOU (your laptop)                                              │
│       │                                                         │
│       │ SSH (Port 22) with .pem key                            │
│       │                                                         │
│       ▼                                                         │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  Security Group: my-first-sg                          │    │
│  │  Inbound: Port 22 from YOUR_IP/32                     │    │
│  │  Outbound: All traffic (default)                      │    │
│  └──────────────────────┬─────────────────────────────────┘    │
│                         │                                       │
│  ┌──────────────────────▼─────────────────────────────────┐    │
│  │  EC2 Instance: t2.micro                                │    │
│  │  AMI: Amazon Linux 2 (free tier eligible)              │    │
│  │  Public IP: 13.231.XXX.XXX (assigned by AWS)          │    │
│  │                                                        │    │
│  │  ┌──────────────────────────────────────────────────┐  │    │
│  │  │  EBS Volume: 8GB gp2 (root volume)              │  │    │
│  │  │  Contains: OS files, your installed software    │  │    │
│  │  └──────────────────────────────────────────────────┘  │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                  │
│  VPC: Default VPC (automatically created by AWS)               │
│  Subnet: Public subnet in your region                          │
└──────────────────────────────────────────────────────────────────┘
```

---

## Step-by-Step: Launch EC2 Instance

### Step 1: Navigate to EC2
1. AWS Console → Services → Compute → EC2
2. Click **"Launch Instance"** (orange button)

### Step 2: Name Your Instance
- Name: `my-first-ec2` (anything descriptive)

### Step 3: Choose AMI (OS Template)
1. Under "Application and OS Images": Select **Amazon Linux 2 AMI**
2. Look for: **"Free tier eligible"** label
3. Architecture: 64-bit (x86)

**Why Amazon Linux 2?** AWS-optimized, best integration with AWS services, free tier eligible, good documentation.

### Step 4: Choose Instance Type
1. Select: **t2.micro** (1 vCPU, 1 GB RAM)
2. Confirm "Free tier eligible" badge

```
┌──────────────────────────────────────────────────────────────────┐
│               INSTANCE TYPE NAMING CONVENTION                    │
│                                                                  │
│  t  2  .  micro                                                  │
│  │  │     │                                                     │
│  │  │     └── Size: nano, micro, small, medium, large, xlarge  │
│  │  └──────── Generation: higher = newer, better               │
│  └──────────── Family: t=burstable, m=general, c=compute       │
│                         r=memory, p=GPU, i=storage-optimized    │
└──────────────────────────────────────────────────────────────────┘
```

### Step 5: Create Key Pair
1. "Key pair (login)" section → Click **"Create new key pair"**
2. Key pair name: `my-first-key`
3. Key pair type: **RSA**
4. Private key file format: **.pem** (for Linux/Mac SSH)
5. Click **"Create key pair"** → Browser downloads `my-first-key.pem`

**⚠️ CRITICAL:** Save this .pem file. AWS does NOT store the private key. If lost, you cannot SSH into the instance.

### Step 6: Configure Security Group
1. "Network settings" → Click **"Edit"**
2. Security group name: `my-first-sg`
3. You'll see default: SSH (Port 22) from 0.0.0.0/0

**⚠️ WARNING:** Change SSH source from 0.0.0.0/0 to "My IP" for security!
- Click dropdown → "My IP" → AWS auto-fills your current IP

### Step 7: Configure Storage
- Default: 8 GB gp2 (leave as is for practice)
- For context: gp2 = General Purpose SSD (older), gp3 = newer/cheaper

### Step 8: Launch
1. Review "Summary" panel on right
2. Click **"Launch Instance"**
3. Wait 1-2 minutes → Status: "Running"

---

## Connecting to Your Instance

### Step 1: Fix Key Permissions (Linux/Mac)
```bash
chmod 400 ~/Downloads/my-first-key.pem
# "chmod 400" = owner can read only, no one else can access
# SSH refuses to use keys with too-open permissions
```

### Step 2: Get Public IP
- EC2 Console → Your instance → Copy "Public IPv4 address"

### Step 3: Connect
```bash
ssh -i ~/Downloads/my-first-key.pem ec2-user@YOUR_PUBLIC_IP

# Example:
ssh -i ~/Downloads/my-first-key.pem ec2-user@13.231.55.123

# -i : identity file (your private key)
# ec2-user : default username for Amazon Linux 2
# For Ubuntu AMI, username is "ubuntu" not "ec2-user"
```

### Step 4: Verify You're In
```bash
# Once connected, you'll see:
[ec2-user@ip-172-31-XX-XX ~]$

# Run some commands to verify:
whoami           # shows: ec2-user
hostname         # shows: ip-172-31-XX-XX
uname -r         # shows kernel version
df -h            # shows disk usage
free -h          # shows RAM usage
curl http://169.254.169.254/latest/meta-data/  # EC2 metadata service
```

---

## Install a Web Server (Verify It Works)

```bash
# Update packages
sudo yum update -y

# Install Apache web server
sudo yum install httpd -y

# Start Apache
sudo systemctl start httpd
sudo systemctl enable httpd  # auto-start on reboot

# Create a test page
echo "<h1>Hello from EC2!</h1>" | sudo tee /var/www/html/index.html
```

Now: Update Security Group to allow HTTP:
- EC2 Console → Security Groups → `my-first-sg` → Inbound rules → Add rule
- Type: HTTP, Port: 80, Source: 0.0.0.0/0

Visit: `http://YOUR_PUBLIC_IP` in browser → You should see "Hello from EC2!"

---

## Diagram: What Happens When You SSH

```
┌──────────────────────────────────────────────────────────────────┐
│                    SSH CONNECTION FLOW                           │
│                                                                  │
│  Your Laptop                                                    │
│  ssh -i key.pem ec2-user@13.231.55.123                         │
│       │                                                         │
│       │ 1. DNS: 13.231.55.123 (already IP, skip DNS)           │
│       │ 2. TCP: Connect to port 22                             │
│       │ 3. SSH Handshake: Exchange public keys                  │
│       │ 4. Your private key proves your identity                │
│       │ 5. Encrypted session established                        │
│       ▼                                                         │
│  AWS Network → Security Group checks: Is Port 22 allowed       │
│  from this IP? → YES → Packet reaches EC2                      │
│       ▼                                                         │
│  EC2 Instance                                                   │
│  SSH daemon (sshd) on Port 22                                   │
│  Checks: Does client's public key match stored authorized_keys? │
│  YES → Shell session granted                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## Expected Output

After completing this practical:
- EC2 instance running (green "Running" status)
- SSH connection successful
- Web server installed and visible in browser

---

## Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `Permission denied (publickey)` | Wrong username or key | Amazon Linux = ec2-user, Ubuntu = ubuntu, check key path |
| `Connection timed out` | Security Group blocking SSH | Check SG inbound rules allow Port 22 from your IP |
| `WARNING: UNPROTECTED PRIVATE KEY FILE` | Key permissions too open | Run `chmod 400 key.pem` |
| `Host key verification failed` | Re-launched instance, same IP | Run `ssh-keygen -R YOUR_IP` to clear old host key |
| Cannot reach web server | Port 80 not in Security Group | Add HTTP inbound rule to Security Group |

---

## Viva Questions — Practical 2

1. **What is an AMI? Can you create your own?**  
   A: Amazon Machine Image = OS + software template. Yes — launch EC2, configure it, then Actions → Image → Create Image. Future instances launch with your exact setup.

2. **What is t2.micro? Break down the name.**  
   A: t=burstable general purpose family, 2=second generation, micro=smallest size. 1 vCPU, 1GB RAM. Free tier eligible.

3. **Why do we chmod 400 the .pem file?**  
   A: SSH refuses to use private key files that are readable by others (security risk). chmod 400 = owner read-only, no access for group/others.

4. **What happens if you terminate the instance? Is data lost?**  
   A: Yes. Termination deletes the instance and by default deletes the root EBS volume. Stop ≠ Terminate. Stopped instances retain EBS data.

5. **What is the metadata service at 169.254.169.254?**  
   A: EC2 Instance Metadata Service (IMDS). Every EC2 instance can query this IP to get information about itself: instance ID, instance type, region, IAM role credentials, etc. Used by applications running on EC2.

6. **Why is the public IP different from the private IP?**  
   A: Private IP is the IP within the VPC (e.g., 172.31.x.x). This is for communication between AWS resources. Public IP is the internet-facing address. When you access EC2 from the internet, traffic goes to public IP; AWS NAT translates to private IP.

7. **What is user data in EC2?**  
   A: A script you provide at launch time that runs once when the instance first starts. Used for: installing software, configuring the instance, running setup commands automatically.

8. **What is the difference between stopping and terminating an EC2 instance?**  
   A: Stop = instance paused, EBS data preserved, no compute charge (EBS storage still billed). Terminate = instance deleted, EBS root volume deleted by default, no further charges.

9. **What is an Elastic IP? Why would you need one?**  
   A: A static public IPv4 address. Regular EC2 public IPs change every time you stop/start the instance. Elastic IP persists. Needed for: pointing DNS records to EC2, production servers that must have consistent IP.

10. **The Security Group allows SSH from 0.0.0.0/0 — what's the risk?**  
    A: Anyone on the internet can attempt to SSH into your instance. They still need the private key (which is strong protection), but brute force against weak keys is possible. Best practice: restrict to your IP or your organization's IP range.

---
