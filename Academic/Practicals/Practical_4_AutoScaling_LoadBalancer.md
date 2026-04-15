# Practical 4 — Configure Auto Scaling and Load Balancing
**Objective:** Set up an Auto Scaling Group with a Launch Template and an Application Load Balancer to distribute and auto-manage traffic.

---

## All Terms You Will Encounter

| Term | Definition |
|------|-----------|
| **Auto Scaling Group (ASG)** | AWS service that automatically adds/removes EC2 instances based on demand |
| **Launch Template** | Blueprint for EC2 instances: AMI, instance type, key pair, security group |
| **Scaling Policy** | Rules that define when and how to scale (add/remove instances) |
| **Target Tracking Policy** | Keep a metric (e.g., CPU) at a target value by scaling automatically |
| **Step Scaling Policy** | Scale by a fixed number of instances when metric crosses a threshold |
| **Cooldown Period** | Time ASG waits after scaling before evaluating again (prevents thrashing) |
| **Application Load Balancer (ALB)** | Layer-7 (HTTP/HTTPS) load balancer that routes traffic to healthy instances |
| **Target Group** | Collection of EC2 instances that receive traffic from ALB |
| **Health Check** | ALB periodically pings instances; unhealthy = removed from rotation |
| **Listener** | ALB component that checks for connection requests on a specific port/protocol |
| **Listener Rule** | Condition + action: if path = /api/* then forward to backend target group |
| **Round Robin** | Default ALB routing: distribute requests evenly across all healthy instances |
| **Sticky Sessions** | Route all requests from same client to same instance (session affinity) |
| **Min/Max/Desired Capacity** | ASG bounds: never below min, never above max, target = desired |
| **Scale Out** | Add instances (horizontal scale — more servers) |
| **Scale Up** | Use bigger instance type (vertical scale — more powerful server) |
| **Scale In** | Remove instances when load decreases |
| **User Data** | Bootstrap script running on EC2 launch — used to install/start web server |
| **Instance Refresh** | Rolling update of ASG instances when launch template changes |
| **stress** | Linux tool to artificially generate CPU load for testing |

---

## Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                  AUTO SCALING + ALB ARCHITECTURE                     │
│                                                                      │
│  Internet Traffic                                                    │
│       │                                                              │
│       ▼                                                              │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  Application Load Balancer (ALB)                               │  │
│  │  my-alb.ap-south-1.elb.amazonaws.com                          │  │
│  │                                                                │  │
│  │  Listener: Port 80 (HTTP)                                      │  │
│  │  Rule: Forward ALL → Target Group "web-servers"               │  │
│  │                                                                │  │
│  │  Health Check: GET /  → 200 OK every 30s                      │  │
│  └──────────────────────────┬─────────────────────────────────────┘  │
│                             │  Round-Robin Distribution               │
│            ┌────────────────┼───────────────────┐                    │
│            ▼                ▼                   ▼                    │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│   │  EC2 Inst 1  │  │  EC2 Inst 2  │  │  EC2 Inst 3  │              │
│   │  t2.micro    │  │  t2.micro    │  │  t2.micro    │              │
│   │  Running     │  │  Running     │  │  (launched   │              │
│   │  Apache      │  │  Apache      │  │  by ASG when │              │
│   │              │  │              │  │  CPU > 70%)  │              │
│   └──────────────┘  └──────────────┘  └──────────────┘              │
│            │                                                         │
│   ┌────────▼──────────────────────────────────────────────────────┐  │
│   │  Auto Scaling Group (ASG)                                     │  │
│   │  Min: 1   Desired: 2   Max: 5                                 │  │
│   │                                                               │  │
│   │  Policy: Scale OUT if CPU > 70% for 2 min → add 1 instance   │  │
│   │  Policy: Scale IN  if CPU < 30% for 10 min → remove 1        │  │
│   └───────────────────────────────────────────────────────────────┘  │
│                             │                                        │
│   ┌─────────────────────────▼─────────────────────────────────────┐  │
│   │  CloudWatch Alarm                                             │  │
│   │  Monitors: CPUUtilization of ASG                              │  │
│   │  Alarm State → Triggers Scaling Policy                       │  │
│   └───────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Step-by-Step

### Step 1: Create Launch Template

A Launch Template is the blueprint every new instance will use.

1. EC2 → Launch Templates → **Create Launch Template**
2. Name: `web-server-template`
3. AMI: Amazon Linux 2 (Free Tier)
4. Instance type: `t2.micro`
5. Key pair: Select your existing key
6. Security Group: Create new `alb-ec2-sg`
   - Inbound: HTTP (80) from ALB security group (we'll add this after)
   - Inbound: SSH (22) from My IP
7. **User Data** (scroll down, expand "Advanced details"):

```bash
#!/bin/bash
yum update -y
yum install -y httpd stress
systemctl start httpd
systemctl enable httpd
# Create webpage showing which instance served the request
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)
echo "<h1>Hello from Instance: $INSTANCE_ID</h1><p>AZ: $AZ</p>" > /var/www/html/index.html
```

**What this does:** On every new instance, installs Apache, creates a page that shows which instance served you. Refresh the ALB URL and you'll see different instance IDs — proof load balancing works.

8. Create Launch Template

---

### Step 2: Create Application Load Balancer

1. EC2 → Load Balancers → **Create Load Balancer** → Application Load Balancer
2. Name: `my-web-alb`
3. Scheme: **Internet-facing** (accepts traffic from internet)
4. IP address type: IPv4
5. VPC: Select your VPC
6. Availability Zones: Select 2 subnets in different AZs (ALB needs minimum 2 AZs)
7. Security Group: Create new `alb-sg`
   - Inbound: HTTP (80) from 0.0.0.0/0
   - Inbound: HTTPS (443) from 0.0.0.0/0
8. Listeners: Port 80, HTTP
9. **Target Group** → Create New:
   - Name: `web-servers-tg`
   - Target type: Instances
   - Protocol: HTTP, Port: 80
   - Health check path: `/`
   - Healthy threshold: 2 checks
   - Interval: 30 seconds
10. Create ALB

**Now update EC2 Security Group:** `alb-ec2-sg` inbound → HTTP (80) → Source: `alb-sg` (the ALB's security group)
This means: only the ALB can send HTTP to EC2, not the public internet directly.

---

### Step 3: Create Auto Scaling Group

1. EC2 → Auto Scaling Groups → **Create Auto Scaling Group**
2. Name: `web-asg`
3. Launch Template: `web-server-template` → Latest version
4. Next → VPC: your VPC, Subnets: select both (multi-AZ for high availability)
5. **Attach to Load Balancer** → Attach to existing target group: `web-servers-tg`
6. Health check type: **ELB** (use ALB health checks, not just EC2 health)
7. Group Size:
   - Minimum: **1**
   - Desired: **2**
   - Maximum: **5**
8. **Scaling Policies** → Target Tracking:
   - Metric: Average CPU Utilization
   - Target value: **70**
   - Scale-in cooldown: 300 seconds
   - Scale-out cooldown: 60 seconds
9. Create ASG

---

### Step 4: Test Load Balancing

1. Wait ~3 minutes for instances to launch and register as healthy in target group
2. Go to ALB → Copy DNS name: `my-web-alb-xxxx.ap-south-1.elb.amazonaws.com`
3. Open in browser: You'll see "Hello from Instance: i-0xxxx"
4. Refresh multiple times — instance ID should occasionally change (round-robin)
5. Open browser incognito + normal — both should work

---

### Step 5: Test Auto Scaling (Simulate High CPU)

```bash
# SSH into one of the instances
ssh -i key.pem ec2-user@INSTANCE_PUBLIC_IP

# Run stress for 10 minutes on all CPUs
stress --cpu $(nproc) --timeout 600

# In AWS Console: Watch CloudWatch → EC2 → CPUUtilization
# When it crosses 70%, ASG launches new instances
# Instances appear in EC2 dashboard and Target Group
```

---

## ALB Routing Rules Explained

```
┌──────────────────────────────────────────────────────────────────┐
│              ALB LISTENER RULES (Layer 7 Routing)                │
│                                                                  │
│  Rule 1 (Priority 1):                                           │
│  IF   path = /api/*                                             │
│  THEN Forward to → backend-api-tg (different target group)     │
│                                                                  │
│  Rule 2 (Priority 2):                                           │
│  IF   host = admin.myapp.com                                    │
│  THEN Forward to → admin-servers-tg                             │
│                                                                  │
│  Rule 3 (Default):                                              │
│  THEN Forward to → web-servers-tg                               │
│                                                                  │
│  This is why it's called APPLICATION load balancer — it         │
│  understands HTTP application-level content, not just ports.   │
└──────────────────────────────────────────────────────────────────┘
```

---

## Expected Output
- ALB DNS name resolves to web page from one of your instances
- Refreshing shows different instance IDs (round-robin proof)
- ASG maintains 2 instances even if one fails (auto-replace)
- CPU stress causes new instance launch within 2-3 minutes

---

## Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| Target Group shows instances "unhealthy" | Security group blocks ALB | Allow HTTP 80 from ALB security group |
| ASG doesn't launch instances | Launch template error | Check Launch Template for correct AMI/key/SG |
| ALB returns 502 Bad Gateway | No healthy targets | Check health check path exists (`/`) on instances |
| CPU stress doesn't trigger scaling | Cooldown period active | Wait 5 minutes and retry |
| Page shows same instance every time | Sticky sessions enabled | Check Target Group → Stickiness is disabled |

---

## Viva Questions — Practical 4

1. **What is the difference between vertical and horizontal scaling?**  
   A: Vertical = bigger instance (more CPU/RAM on same machine, has hardware limit). Horizontal = more instances (add servers, essentially unlimited). ASG does horizontal scaling.

2. **What is the purpose of a Launch Template?**  
   A: Defines the configuration for every new instance ASG creates: AMI, instance type, key pair, security group, user data. Without it, ASG doesn't know how to create instances.

3. **What is a Target Group? What is a Health Check?**  
   A: Target Group = set of instances receiving traffic from ALB. Health Check = ALB periodically sends HTTP request to each instance (e.g., GET /). If it gets 200 OK, instance is healthy. If 2 checks fail, instance is "unhealthy" and removed from rotation.

4. **Why do you need minimum 2 Availability Zones for ALB?**  
   A: ALB is deployed across AZs for high availability. If one AZ has a problem, ALB continues routing to healthy AZ. Single AZ ALB would be a single point of failure.

5. **What is the cooldown period in Auto Scaling?**  
   A: Time ASG waits after a scaling action before evaluating scaling policies again. Prevents "thrashing" — rapidly adding and removing instances because metric oscillates around threshold.

6. **What happens if an instance fails a health check?**  
   A: ALB stops routing traffic to it immediately. If ASG's health check type is ELB, ASG also terminates the unhealthy instance and launches a replacement to maintain desired capacity.

7. **What is the difference between ALB and Classic Load Balancer?**  
   A: ALB = Layer 7, understands HTTP/HTTPS, path-based routing, host-based routing, WebSocket. Classic LB = Layer 4 (TCP) and basic Layer 7, less intelligent. AWS recommends ALB for all new applications.

8. **What is "desired capacity" in ASG?**  
   A: The number of instances ASG tries to maintain under current load. ASG scales desired capacity up/down within min/max bounds. If an instance terminates unexpectedly, ASG launches replacement to match desired.

9. **What is User Data? When does it run?**  
   A: A script provided at instance launch that runs ONCE when the instance first starts (not on every reboot). Used for: installing software, configuring the app, starting services.

10. **How does ASG know when to scale?**  
    A: CloudWatch monitors metrics (CPU, custom metrics). CloudWatch Alarm fires when metric crosses threshold. Alarm triggers ASG scaling policy: add X instances (scale out) or remove X instances (scale in).

---
