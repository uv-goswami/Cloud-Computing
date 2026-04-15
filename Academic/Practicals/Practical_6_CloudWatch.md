# Practical 6 — Monitor Resources Using AWS CloudWatch
**Objective:** Set up CloudWatch metrics, alarms, and SNS notifications for an EC2 instance.

---

## All Terms You Will Encounter

| Term | Definition |
|------|-----------|
| **CloudWatch** | AWS monitoring, logging, and alerting service |
| **Metric** | Numerical time-series data point (e.g., CPUUtilization = 45%) |
| **Namespace** | Category grouping metrics: AWS/EC2, AWS/RDS, AWS/Lambda |
| **Dimension** | Filter to narrow metrics (e.g., InstanceId = i-0abc123) |
| **Alarm** | Watches a metric; triggers action when threshold crossed |
| **Alarm States** | OK (normal), ALARM (threshold crossed), INSUFFICIENT_DATA |
| **Period** | How often metric is evaluated (60 seconds, 5 minutes) |
| **Evaluation Periods** | How many consecutive periods must breach threshold before alarm fires |
| **SNS** | Simple Notification Service — publish/subscribe messaging |
| **SNS Topic** | Named channel; publishers send to it, subscribers receive from it |
| **Subscription** | A subscriber endpoint: email, SMS, Lambda, SQS, HTTP |
| **Dashboard** | Custom CloudWatch page with graphs for your key metrics |
| **CloudWatch Logs** | Store and search application/system log files |
| **Log Group** | Container for log streams from the same source |
| **Log Stream** | Sequence of log events from a single source instance |
| **CloudWatch Agent** | Software installed on EC2 to send OS-level metrics (RAM, disk) |
| **Custom Metric** | Metric you publish from your application (not auto-collected by AWS) |
| **Statistics** | Aggregation of data points: Average, Sum, Maximum, Minimum, p99 |

---

## Architecture: CloudWatch Monitoring Pipeline

```
┌──────────────────────────────────────────────────────────────────────┐
│                  CLOUDWATCH MONITORING ARCHITECTURE                  │
│                                                                      │
│  DATA SOURCES                                                        │
│  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐        │
│  │ EC2    │  │  RDS   │  │Lambda  │  │  ALB   │  │  ASG   │        │
│  │metrics │  │metrics │  │metrics │  │metrics │  │metrics │        │
│  └───┬────┘  └───┬────┘  └───┬────┘  └───┬────┘  └───┬────┘        │
│      └───────────┴───────────┴───────────┴───────────┘             │
│                              │                                       │
│                              ▼ (Auto-collected every 5 min)         │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  CloudWatch Metrics Storage                                  │   │
│  │  Namespace: AWS/EC2                                          │   │
│  │  ┌──────────────────────┐  ┌──────────────────────────────┐  │   │
│  │  │ CPUUtilization       │  │ NetworkIn / NetworkOut       │  │   │
│  │  │ DiskReadOps          │  │ StatusCheckFailed            │  │   │
│  │  │ DiskWriteOps         │  │ ...etc                       │  │   │
│  │  └──────────────────────┘  └──────────────────────────────┘  │   │
│  └─────────────────────────────┬────────────────────────────────┘   │
│                                │                                     │
│                                ▼                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  CloudWatch Alarm: "high-cpu-alarm"                          │   │
│  │  Metric: CPUUtilization > 80%                                │   │
│  │  For: 2 consecutive 5-min periods                            │   │
│  │  State: OK → ALARM                                           │   │
│  └─────────────────────────────┬────────────────────────────────┘   │
│                                │ Alarm fires                         │
│               ┌────────────────┼────────────────┐                   │
│               ▼                ▼                ▼                   │
│    ┌───────────────┐  ┌─────────────────┐  ┌──────────────────┐    │
│    │  SNS Topic    │  │  Auto Scaling   │  │  EC2 Action      │    │
│    │  → Email you  │  │  → Scale out    │  │  → Stop/Reboot   │    │
│    │  → SMS        │  │                 │  │  instance        │    │
│    └───────────────┘  └─────────────────┘  └──────────────────┘    │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Step-by-Step

### Step 1: Launch EC2 Instance (if not running)
Use your existing EC2 from Practical 2, or launch a new t2.micro with Amazon Linux 2.

---

### Step 2: Create SNS Topic for Email Notifications

1. AWS Console → SNS → Topics → **Create Topic**
2. Type: **Standard** (not FIFO — we need multiple subscribers, not ordering)
3. Name: `ec2-alerts`
4. Create Topic

5. Topic → **Create Subscription**
6. Protocol: **Email**
7. Endpoint: `your-email@example.com`
8. Create Subscription

9. **Check your email** → Click "Confirm subscription" link
   - Must confirm or SNS cannot send emails to you

---

### Step 3: Create CloudWatch Alarm for CPU

1. CloudWatch → Alarms → All Alarms → **Create Alarm**
2. **Select Metric** → EC2 → Per-Instance Metrics → CPUUtilization
3. Find your instance ID → Select → Next
4. Define Threshold:
   - Threshold type: Static
   - Whenever CPUUtilization is: **Greater than**
   - Threshold value: **80**
   - Period: **5 minutes**
   - Evaluation periods: **2** (alarm fires after 2 consecutive breaches)
5. **Configure Actions:**
   - Alarm state trigger: **In alarm**
   - Send notification to: `ec2-alerts` (your SNS topic)
6. Name: `EC2-High-CPU-Alarm`
7. Create Alarm

---

### Step 4: Create CloudWatch Dashboard

1. CloudWatch → Dashboards → **Create Dashboard**
2. Name: `EC2-Monitoring`
3. Add widget:
   - Line chart → Metrics → EC2 → Per-Instance → CPUUtilization (your instance)
4. Add another widget:
   - Line chart → NetworkIn + NetworkOut
5. Save dashboard

---

### Step 5: Simulate High CPU and Trigger Alarm

```bash
# SSH into your EC2 instance
ssh -i key.pem ec2-user@YOUR_PUBLIC_IP

# Install stress tool
sudo yum install stress -y

# Generate 100% CPU load for 15 minutes
sudo stress --cpu $(nproc) --timeout 900

# In another terminal / CloudWatch console:
# Watch the CPUUtilization graph spike to 100%
# After 2 evaluation periods (10 minutes), alarm state → ALARM
# Check your email for SNS notification
```

---

### Step 6: View CloudWatch Logs (Bonus — EC2 Agent)

By default, CloudWatch collects EC2 metrics but NOT RAM usage or disk usage (those require the CloudWatch Agent).

```bash
# Install CloudWatch Agent
sudo yum install amazon-cloudwatch-agent -y

# Create config file
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard

# Start the agent
sudo systemctl start amazon-cloudwatch-agent
sudo systemctl enable amazon-cloudwatch-agent
```

After setup, CloudWatch shows: Memory Used %, Disk Used % — metrics not available without the agent.

---

## Expected Output
- SNS subscription confirmed via email
- CloudWatch alarm created (initial state: OK or INSUFFICIENT_DATA)
- After stress test: alarm state changes to ALARM
- Email received: "ALARM: EC2-High-CPU-Alarm"
- Dashboard shows CPU spike

---

## Common Errors and Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| Email not received | Subscription not confirmed | Check spam; click confirmation link in email |
| Alarm stays in INSUFFICIENT_DATA | Instance not sending metrics | Instance must be running; wait 5 minutes |
| CPU doesn't reach 80% | t2.micro is "burstable" | Use `--cpu 1 --timeout 600`; check instance CPU credits |
| Can't find instance in CloudWatch | Wrong region | Ensure CloudWatch and EC2 are in same region |

---

## Viva Questions — Practical 6

1. **What is the difference between a CloudWatch Metric and an Alarm?**  
   A: Metric = raw data point collected over time (CPUUtilization = 45%). Alarm = watches a metric and transitions between states (OK/ALARM/INSUFFICIENT_DATA) when metric crosses a defined threshold. Alarms can trigger actions; metrics are just data.

2. **What are the 3 states of a CloudWatch Alarm?**  
   A: OK (metric is within threshold), ALARM (metric has breached threshold for configured evaluation periods), INSUFFICIENT_DATA (not enough data to determine state — common on newly created alarms or stopped instances).

3. **What is SNS? How is it different from SES?**  
   A: SNS (Simple Notification Service) = pub/sub for system-to-system notifications: CloudWatch → SNS → email/SMS/Lambda. SES (Simple Email Service) = transactional email service for sending emails from applications (newsletters, receipts). Different use cases.

4. **Why use 2 evaluation periods instead of 1 for the CPU alarm?**  
   A: A single spike might be transient (one-time burst). Requiring 2 consecutive periods means the high CPU is sustained. Prevents false alarms and unnecessary notifications.

5. **Why doesn't CloudWatch collect RAM usage by default?**  
   A: RAM is OS-level metric. CloudWatch collects hypervisor-level metrics (CPU, network, disk I/O from outside the VM). RAM usage happens inside the OS — AWS can't see it without an agent running inside. CloudWatch Agent sends OS-level metrics from inside the instance.

6. **What is a CloudWatch Dashboard?**  
   A: Custom web page in CloudWatch with widgets (line charts, numbers, alarms) showing metrics you care about. Useful for NOC (Network Operations Center) displays, incident response, quick status checks.

7. **What is the difference between SNS Standard and FIFO topic?**  
   A: Standard = best-effort ordering, at-least-once delivery, unlimited throughput. FIFO = guaranteed order, exactly-once delivery, 300 transactions/sec limit. CloudWatch alarms use Standard (order doesn't matter for alerts).

---
