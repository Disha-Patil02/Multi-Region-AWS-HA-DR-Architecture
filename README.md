# AWS High Availability & Disaster Recovery (Active-Passive) Project

Multi-region HA/DR architecture using **Auto Scaling, Application Load Balancer, and Route 53 Failover Routing**, with **Mumbai (ap-south-1)** as the primary region and **N. Virginia (us-east-1)** as the secondary/DR region.

**Live domain used in this project:** `dishapatil.online`

> For click-by-click console instructions with exact resource names, see [`SETUP_GUIDE.md`](./SETUP_GUIDE.md).

---

## Architecture

```
                             ┌─────────────────────────┐
                             │      Route 53             │
                             │  Zone: dishapatil.online   │
                             │  Record: www.dishapatil... │
                             │  Failover Routing Policy   │
                             │  Health Check: HA-DR-health-check
                             └───────────┬───────────────┘
                                         │
                 ┌───────────────────────┴───────────────────────┐
                 │ PRIMARY (Healthy)                              │ SECONDARY (Standby)
                 ▼                                                 ▼
     ┌─────────────────────────┐                       ┌─────────────────────────┐
     │  Region: ap-south-1     │                       │  Region: us-east-1      │
     │  (Mumbai)               │                       │  (N. Virginia)          │
     │                         │                       │                         │
     │  ALB: HA-DR-load-balncer│                       │  ALB: HA-DR-Load-balancer│
     │        │                │                       │        │                │
     │  TG: HA-DR-target-grp   │                       │  TG: HA-DR-target-group │
     │        │                │                       │        │                │
     │  ASG: HA-DR-auto-scaling│                       │  ASG: HA-DR-Auto-scaling│
     │  LT: HA-DR-launch-template                      │  LT: Ha-Dr-launch-template
     │  EC2 x2 (multi-AZ)      │                       │  EC2 x2 (multi-AZ)      │
     └─────────────────────────┘                       └─────────────────────────┘
```

**Failover logic:** Route 53's `HA-DR-health-check` continuously checks `www.dishapatil.online` on port 80. If Mumbai fails, DNS answers switch to the Secondary record (N. Virginia's ALB) automatically — validated to happen within about a minute.

---

## What's Built

| Component | Mumbai (Primary) | N. Virginia (Secondary) |
|---|---|---|
| Region | ap-south-1 | us-east-1 |
| Launch Template | `HA-DR-launch-template` (AMI `ami-066c4849e6b3a1e3d`, key pair `dis-mumbai`) | `Ha-Dr-launch-template` (AMI `ami-0fef201115eefe936`, key pair `linux`) |
| Target Group | `HA-DR-target-grp` | `HA-DR-target-group` |
| ALB | `HA-DR-load-balncer` | `HA-DR-Load-balancer` |
| Auto Scaling Group | `HA-DR-auto-scaling` (2/2/4) | `HA-DR-Auto-scaling` (2/2/4) |
| Instance type | t3.micro | t3.micro |
| Route 53 Role | Primary + `HA-DR-health-check` | Secondary |

Each region independently runs: **Launch Template → Target Group → Application Load Balancer → Auto Scaling Group**, in the **default VPC**, across 2 Availability Zones. Both use the same nginx-based user data script that displays the serving instance ID, private IP, AZ, and timestamp — useful for visually confirming failover.

---

## VPC: Default vs Custom

This project uses the **default VPC** in both regions — sufficient since the goal was to prove out ASG + ALB + Route 53 failover mechanics, not network isolation.

| | Default VPC (used here) | Custom VPC |
|---|---|---|
| Good for | Learning, demos, quick PoCs | Production, real DR, security compliance |
| Subnet control | Auto-created, all public | You define public/private tiers |
| Security posture | Everything internet-facing by default | Can isolate app/DB layers with private subnets + NAT |
| Cost | No extra NAT Gateway needed | NAT Gateway costs if using private subnets |

**If extending this project:** move to a custom VPC per region (e.g. `10.0.0.0/16` Mumbai / `10.1.0.0/16` Virginia, non-overlapping CIDRs) with public subnets for the ALB and private subnets for EC2. The ASG/ALB/Target Group logic stays identical — only subnet placement changes.

---

## Failover Test Result ✅

1. Deleted the `HA-DR-auto-scaling` group in Mumbai → target group had 0 healthy targets
2. `HA-DR-health-check` reported unhealthy within ~1 minute
3. Route 53 automatically switched `www.dishapatil.online` to the Secondary record
4. Browser refresh showed the page served by an N. Virginia instance (`us-east-1a`) instead of Mumbai

Failback happens automatically once the Mumbai stack is rebuilt and healthy again.

---

## Design Notes & Possible Improvements

- **Current pattern = Hot Standby / Active-Passive** — both regions run 24/7. Fast failover, but doubles compute cost.
- **Cheaper alternative — Pilot Light:** keep Virginia's `Ha-Dr-launch-template` / `HA-DR-target-group` / `HA-DR-Load-balancer` defined but ASG desired capacity = 0; scale up via CloudWatch Alarm + Lambda only when Mumbai fails.
- Add **HTTPS** (ACM certificate + port 443 listener) instead of plain HTTP.
- Add a database tier (e.g. RDS Multi-Region/Read Replica) — this project currently covers only the compute/web tier.
- Add **CloudWatch Alarms + SNS** to get notified automatically when failover happens.
- Standardize naming across regions (currently `HA-DR-target-grp` vs `HA-DR-target-group`, etc.) if this gets scripted with Terraform/CloudFormation later.

---

## Clean-Up

To avoid ongoing charges, tear down in each region: Route 53 records + `HA-DR-health-check` → ASG → ALB → Target Group → Launch Template (optional to keep).

Full teardown order is in [`SETUP_GUIDE.md`](./SETUP_GUIDE.md#5-clean-up-avoid-ongoing-charges).
