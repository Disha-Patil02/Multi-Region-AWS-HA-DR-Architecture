# Setup Guide — AWS HA/DR (Mumbai Primary + N. Virginia Secondary)

Step-by-step console instructions to build and test the full active-passive setup, using the exact resource names from this project. For the high-level project overview, see `README.md`.

**Domain used:** `dishapatil.online` (record: `www.dishapatil.online`)

---

## 0. VPC Note

This project uses the **default VPC** in both regions (`ap-south-1` and `us-east-1`). No custom VPC, subnets, or NAT Gateway were required — the default public subnets across multiple AZs were enough for the ALB and EC2 instances.

---

## 1. Primary Region Setup — Mumbai (ap-south-1)

### Step 1: Launch Template — `HA-DR-launch-template`
1. EC2 (Mumbai) → Launch Templates → Create launch template
2. Name: `HA-DR-launch-template`
3. AMI: `ami-066c4849e6b3a1e3d` (Amazon Linux, matches nginx install commands below)
4. Instance type: `t3.micro`
5. Key pair: `dis-mumbai`
6. Security group: allow **HTTP (80)** from `0.0.0.0/0`, **SSH (22)** from your IP only
7. **User data**:
   ```bash
   #!/bin/bash
   dnf install -y nginx || yum install -y nginx

   TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
   INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
   PRIVATE_IP=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/local-ipv4)
   AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)

   cat > /usr/share/nginx/html/index.html <<EOF2
   <!DOCTYPE html>
   <html>
   <head><title>Server Info</title>
   <style>
   body{font-family:monospace;background:#0d1117;color:#dfe6ee;display:flex;height:100vh;align-items:center;justify-content:center}
   .card{background:#131a24;border:1px solid #232d3b;border-radius:12px;padding:32px 40px;text-align:center}
   h1{color:#4fc3d9;margin:0 0 10px}
   p{color:#7f8ea3;margin:4px 0}
   </style></head>
   <body>
     <div class="card">
       <h1>Served by $INSTANCE_ID</h1>
       <p>Private IP: $PRIVATE_IP</p>
       <p>AZ: $AZ</p>
       <p id="time"></p>
     </div>
     <script>document.getElementById('time').textContent = new Date().toString();</script>
   </body>
   </html>
   EOF2

   systemctl enable nginx
   systemctl start nginx
   ```
   > Uses IMDSv2 (token-based metadata request) to pull the instance ID, private IP, and AZ, and renders them on a styled page — this is exactly the "Served by i-xxxx / AZ: ap-south-1b" page seen in testing.

### Step 2: Target Group — `HA-DR-target-grp`
1. EC2 → Target Groups → Create target group
2. Target type: Instances
3. Protocol: **HTTP**, Port: **80**
4. VPC: default VPC (`vpc-0c906c6e7eeb8f37a`)
5. Health check path: `/`
6. Leave "Register targets" empty — the ASG auto-registers instances

### Step 3: Application Load Balancer — `HA-DR-load-balncer`
1. EC2 → Load Balancers → Create → **Application Load Balancer**
2. Name: `HA-DR-load-balncer`
3. Scheme: Internet-facing
4. VPC: default VPC; AZs: `ap-south-1a` and `ap-south-1b`
5. Security group: allow HTTP (80) inbound
6. Listener: HTTP:80 → forward to `HA-DR-target-grp`
7. Create — DNS name: `HA-DR-load-balncer-152305428.ap-south-1.elb.amazonaws.com`

### Step 4: Auto Scaling Group — `HA-DR-auto-scaling`
1. EC2 → Auto Scaling Groups → Create
2. Name: `HA-DR-auto-scaling`
3. Launch template: `HA-DR-launch-template`
4. VPC + subnets: same 2 AZs as the ALB
5. Attach to existing load balancer → target group `HA-DR-target-grp`
6. Health checks: enable **ELB health checks**
7. Desired / Min / Max capacity: **2 / 2 / 4**
8. Create — confirm both instances show **2/2 Healthy**

### Step 5: Validate Primary
- Open `HA-DR-load-balncer-152305428.ap-south-1.elb.amazonaws.com` in a browser
- Should show the dark "Served by i-xxxxxxxx / AZ: ap-south-1b" card
- Refresh to confirm it alternates between the 2 instances/AZs

---

## 2. Secondary / DR Region Setup — N. Virginia (us-east-1)

Repeat Steps 1–5 above in `us-east-1`, using these exact names:

| Resource | Name | Details |
|---|---|---|
| Launch Template | `Ha-Dr-launch-template` | AMI `ami-0fef201115eefe936`, `t3.micro`, key pair `linux` |
| Target Group | `HA-DR-target-group` | HTTP:80, default VPC (`vpc-0e5fbb4b5a6f9ca56`) |
| ALB | `HA-DR-Load-balancer` | AZs `us-east-1a` / `us-east-1b`, DNS: `HA-DR-Load-balancer-1163195528.us-east-1.elb.amazonaws.com` |
| Auto Scaling Group | `HA-DR-Auto-scaling` | Desired/Min/Max: 2 / 2 / 4 |

Use the **same nginx user data script** from Step 1 above — it automatically prints whichever region/AZ/instance is actually serving the request, so no changes are needed.

> Note: naming was typed slightly differently between regions (`HA-DR-target-grp` vs `HA-DR-target-group`, `HA-DR-load-balncer` vs `HA-DR-Load-balancer`) — this doesn't affect functionality, but keep it in mind if you're scripting this with the CLI/IaC later, since exact names matter there.

---

## 3. Route 53 Failover Configuration

**Hosted zone:** `dishapatil.online`
**Record name:** `www.dishapatil.online`

### Step 1: Health Check — `HA-DR-health-check`
1. Route 53 → Health Checks → Create
2. Type: **Endpoint**, specified by **Domain name**
3. Domain: `www.dishapatil.online`, Protocol: HTTP, Port: **80**, Path: `/`
4. This checks the **Primary (Mumbai)** endpoint

### Step 2: Failover Records (in `dishapatil.online` hosted zone)
1. **Primary record** — `www.dishapatil.online`
   - Type A, Alias: Yes → Alias to Application Load Balancer → `ap-south-1` → `HA-DR-load-balncer`
   - Routing policy: **Failover** → **Primary**
   - Associate health check: `HA-DR-health-check`
2. **Secondary record** — `www.dishapatil.online`
   - Type A, Alias: Yes → Alias to Application Load Balancer → `us-east-1` → `HA-DR-Load-balancer`
   - Routing policy: **Failover** → **Secondary**

### Step 3: Validate
- Browse to `dishapatil.online` → should resolve to Mumbai and show its "Served by" card
- `nslookup dishapatil.online` to confirm which ALB it currently resolves to

---

## 4. Failover Test (as performed in this project)

1. In Mumbai (ap-south-1): delete the `HA-DR-auto-scaling` Auto Scaling Group — this terminates both EC2 instances, leaving the target group with 0 healthy targets
2. `HA-DR-health-check` detects the failure within about a minute
3. Route 53 automatically switches DNS answers to the Secondary record (`HA-DR-Load-balancer` in `us-east-1`)
4. Refreshing `dishapatil.online` now shows the card served by the N. Virginia instance (e.g. `Served by i-0a25c0171df76b3f6`, `AZ: us-east-1a`)

**To fail back:** recreate the Mumbai launch template/target group/ALB/ASG stack, wait for `HA-DR-health-check` to report Healthy again, and Route 53 automatically shifts traffic back to Primary.

---

## 5. Clean-Up (Avoid Ongoing Charges)

In **each region**, delete in this order:
1. Route 53 records (Primary/Secondary on `www.dishapatil.online`) + `HA-DR-health-check`
2. Auto Scaling Group (`HA-DR-auto-scaling` / `HA-DR-Auto-scaling`) — terminates instances automatically
3. Load Balancer (`HA-DR-load-balncer` / `HA-DR-Load-balancer`)
4. Target Group (`HA-DR-target-grp` / `HA-DR-target-group`)
5. Launch Template (optional — safe to keep for reuse)
