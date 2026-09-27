# AWS High Availability & Disaster Recovery (Active-Passive) Project

Multi-region HA/DR architecture using **Auto Scaling, Application Load Balancer, and Route 53 Failover Routing**, with **Mumbai (ap-south-1)** as the primary region and **N. Virginia (us-east-1)** as the secondary/DR region.

> For click-by-click console instructions, see [`SETUP_GUIDE.md`](./SETUP_GUIDE.md).

---

## Architecture

```
                             ┌─────────────────────────┐
                             │      Route 53            │
                             │  Hosted Zone (your domain)│
                             │  Failover Routing Policy  │
                             │  + Health Check on Primary│
                             └───────────┬───────────────┘
                                         │
                 ┌───────────────────────┴───────────────────────┐
                 │ PRIMARY (Healthy)                              │ SECONDARY (Standby)
                 ▼                                                 ▼
     ┌─────────────────────────┐                       ┌─────────────────────────┐
     │  Region: ap-south-1     │                       │  Region: us-east-1      │
     │  (Mumbai)               │                       │  (N. Virginia)          │
     │                         │                       │                         │
     │  ALB (public subnets)   │                       │  ALB (public subnets)   │
     │        │                │                       │        │                │
     │  Target Group (HTTP)    │                       │  Target Group (HTTP)    │
     │        │                │                       │        │                │
     │  Auto Scaling Group     │                       │  Auto Scaling Group     │
     │  (Launch Template)      │                       │  (Launch Template)      │
     │  EC2 x N (multi-AZ)     │                       │  EC2 x N (multi-AZ)     │
     └─────────────────────────┘                       └─────────────────────────┘
```

**Failover logic:** Route 53 continuously health-checks the primary ALB endpoint. If it fails, DNS answers switch to the secondary record automatically (typically within ~30–90 seconds depending on health check interval/threshold), routing users to the N. Virginia stack instead.

---

## What's Built

| Component | Mumbai (Primary) | N. Virginia (Secondary) |
|---|---|---|
| Region | ap-south-1 | us-east-1 |
| Launch Template | webserver-lt-mumbai | webserver-lt-virginia |
| Target Group | tg-mumbai | tg-virginia |
| ALB | alb-mumbai | alb-virginia |
| ASG | asg-mumbai | asg-virginia |
| Route 53 Role | Primary + Health Check | Secondary |

Each region independently runs: **Launch Template → Target Group → Application Load Balancer → Auto Scaling Group**, all in the **default VPC**, across multiple Availability Zones.

---

## VPC: Default vs Custom

Currently using the **default VPC** in both regions — this is fine for a validated PoC like this one.

| | Default VPC | Custom VPC |
|---|---|---|
| Good for | Learning, demos, quick PoCs (current state) | Production, real DR, security compliance |
| Subnet control | Auto-created, all public | You define public/private tiers |
| Security posture | Everything internet-facing by default | Can isolate app/DB layers with private subnets + NAT |
| Peering/Transit Gateway later | Harder to manage cleanly | Easier, predictable CIDR planning |
| Cost | No extra NAT Gateway needed | NAT Gateway costs if using private subnets |

**Recommendation:** keep the default VPC while this stays a PoC. Move to a custom VPC (public subnets for ALB, private subnets for EC2, non-overlapping CIDRs per region, e.g. `10.0.0.0/16` Mumbai / `10.1.0.0/16` Virginia) if you turn this into a production-style or portfolio project. The ASG/ALB/Target Group logic doesn't change — only subnet placement does.

---

## Failover Test Result ✅

1. Deleted the Mumbai ASG/instances → target group had no healthy targets
2. Route 53 health check failed within ~1 minute
3. DNS automatically switched to the Secondary (N. Virginia) record
4. Domain started serving traffic from N. Virginia with no manual intervention

Failback happens automatically once the Mumbai stack is healthy again.

---

## Design Notes & Possible Improvements

- **Current pattern = Hot Standby / Active-Passive** — both regions run 24/7. Fast failover, but doubles compute cost.
- **Cheaper alternative — Pilot Light:** keep Virginia's Launch Template/Target Group/ALB defined but ASG desired capacity = 0; use a CloudWatch Alarm + Lambda to scale it up only when Mumbai fails.
- Add **HTTPS** (ACM certificate + port 443 listener) instead of plain HTTP for a production-realistic setup.
- Add a database tier with **RDS Multi-Region/Read Replica** — this project currently covers only the compute/web tier.
- Add **CloudWatch Alarms + SNS** so you get notified when failover actually happens.

---
