# Setup Guide — AWS HA/DR (Mumbai Primary + N. Virginia Secondary)

Step-by-step console instructions to build and test the full active-passive setup. For the high-level project overview, see `README.md`.

---

## 0. VPC Decision (Read Before You Start)

You can use the **default VPC** in both regions — that's what this guide assumes, and it's exactly what you already built successfully.

Move to a **custom VPC** later (not required now) if you want:
- Private subnets for EC2 (more secure than public-only)
- Predictable, non-overlapping CIDRs across regions (e.g. `10.0.0.0/16` Mumbai, `10.1.0.0/16` Virginia) in case you ever peer them
- A NAT Gateway per region for outbound-only internet from private instances

If you do switch to a custom VPC, every step below is identical — you just pick your own VPC/subnets instead of the default ones.

---

## 1. Primary Region Setup — Mumbai (ap-south-1)

### Step 1: Launch Template
1. EC2 → Launch Templates → Create launch template
2. Name: `webserver-lt-mumbai`
3. AMI: choose OS (e.g. Amazon Linux 2023)
4. Instance type: `t2.micro` / `t3.micro` (free-tier friendly)
5. Key pair: select or create
6. Security group: allow **HTTP (80)** from `0.0.0.0/0`, **SSH (22)** from your IP only
7. **User data** (bootstraps the web server on boot):
   ```bash
  #!/bin/bash
dnf install -y nginx || yum install -y nginx

TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
PRIVATE_IP=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/local-ipv4)
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)

cat > /usr/share/nginx/html/index.html <<EOF
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
EOF

systemctl enable nginx
systemctl start nginx

   ```
   > Tip: including the AZ/region in the page output makes failover testing visually obvious.

### Step 2: Target Group
1. EC2 → Target Groups → Create target group
2. Target type: Instances
3. Protocol: **HTTP**, Port: **80**
4. VPC: default (or your custom VPC)
5. Health check path: `/`
6. Leave "Register targets" empty — the ASG will auto-register instances

### Step 3: Application Load Balancer (ALB)
1. EC2 → Load Balancers → Create → **Application Load Balancer**
2. Name: `alb-mumbai`
3. Scheme: Internet-facing
4. VPC: same as target group; select **at least 2 AZs**
5. Security group: allow HTTP (80) inbound
6. Listener: HTTP:80 → forward to the target group from Step 2
7. Create — note the ALB's DNS name (e.g. `alb-mumbai-xxxxx.ap-south-1.elb.amazonaws.com`)

### Step 4: Auto Scaling Group (ASG)
1. EC2 → Auto Scaling Groups → Create
2. Name: `asg-mumbai`
3. Launch template: `webserver-lt-mumbai`
4. VPC + subnets: select 2+ AZs
5. Attach to existing load balancer → select the target group from Step 2
6. Health checks: enable **ELB health checks** (not just EC2 status checks) — this makes ASG replace instances that fail at the app layer, not just the hardware layer
7. Desired/Min/Max capacity: e.g. 2 / 2 / 4
8. Scaling policy: target tracking on CPU (e.g. 50%) or as needed
9. Create — confirm instances launch and register as healthy in the target group

### Step 5: Validate Primary
- Hit the ALB DNS name in a browser → should serve the page from Step 1's user data
- Refresh multiple times → confirm traffic alternates between instances/AZs

---

## 2. Secondary / DR Region Setup — N. Virginia (us-east-1)

Repeat **Steps 1–5 above exactly**, but in `us-east-1`, using these names:
- `webserver-lt-virginia`
- `tg-virginia`
- `alb-virginia`
- `asg-virginia`

**Important:** this stack should be fully live and running at all times (active-passive/hot-standby), not built on demand — that's what makes failover fast. See the README's "Design Notes" section for a cheaper alternative ("pilot light") if you want to reduce standing cost later.

---

## 3. Route 53 Failover Configuration

### Step 1: Hosted Zone
- Route 53 → Hosted Zones → use an existing hosted zone or create one for your domain

### Step 2: Health Check (on Primary)
1. Route 53 → Health Checks → Create
2. Type: **Endpoint**
3. Specify: Domain name or IP → use the **Mumbai ALB DNS name**
4. Protocol: HTTP, Port 80, Path `/`
5. Request interval: 30s (or 10s for faster failover, at higher cost)
6. Failure threshold: 3 (default) — lower to 2 for faster detection if you want

### Step 3: Failover Records
1. Route 53 → Hosted zone → Create record
2. **Primary record:**
   - Record name: your domain/subdomain
   - Alias: Yes → Alias to Application/Classic Load Balancer → region `ap-south-1` → select `alb-mumbai`
   - Routing policy: **Failover**
   - Failover record type: **Primary**
   - Associate with the health check from Step 2
3. **Secondary record:**
   - Same record name
   - Alias to `alb-virginia` in `us-east-1`
   - Routing policy: **Failover**
   - Failover record type: **Secondary**
   - Health check optional (can also monitor Virginia)

### Step 4: Validate
- Query your domain → should resolve to the Mumbai ALB and serve Mumbai's page
- Use `dig yourdomain.com` or `nslookup` to confirm which ALB it's resolving to

---

## 4. Failover Test

1. Delete/disable the Mumbai ASG (or stop its instances) so the target group has no healthy targets
2. Route 53 health check detects failure (within ~1–2 minutes, based on interval × threshold)
3. DNS answers switch to the Secondary record (Virginia ALB)
4. Refreshing the domain now serves the N. Virginia page

**To fail back:** recreate/re-enable the Mumbai stack, wait for its health check to pass again, and Route 53 automatically shifts traffic back to Primary.

---

## 5. Clean-Up (Avoid Ongoing Charges)

Delete in this order, in **each region**:
1. Route 53 records (Primary/Secondary) + Health Check
2. Auto Scaling Group (terminates instances automatically)
3. Load Balancer
4. Target Group
5. Launch Template (optional — safe to keep for reuse)
6. Any custom VPC resources (NAT Gateway, EIPs — these cost money even when idle)