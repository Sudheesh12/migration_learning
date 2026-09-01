## Project 1: Lift-and-Shift a VM Fleet with Terraform + Packer

### Concept
You'll simulate a real migration engagement: a company has "on-prem" servers (your local Hyper-V/VMware VMs) that need to move to AWS. You'll build golden images, provision the target cloud environment as code, migrate data, and do a scripted cutover — the same workflow used in real Migration Factory engagements.

### Architecture

```
[ON-PREM SIMULATION]              [AWS TARGET]
Hyper-V/VMware Host                VPC (10.0.0.0/16)
├── App Server (large VM)    →     ├── Public Subnet (ALB)
├── DB Server (large VM)     →     ├── Private Subnet (EC2 app tier)
└── File Server (large VM)   →     └── Private Subnet (RDS/EC2 DB)
                                    + Security Groups, NAT Gateway
```

### Phase 1 — Build the "on-prem" environment
- Set up 2–3 VMs in Hyper-V or VMware Workstation (since you specified large VMs — go with something like 4 vCPU/8GB RAM per VM if your host can handle it; keep an eye on host resource limits, 3 large VMs concurrently can be heavy).
- Install a real workload on them: e.g., a LAMP/LEMP stack app server, a MySQL/PostgreSQL DB server, and a file server with sample data (a few GB).
- Document the "current state" like a real assessment: OS version, installed packages, open ports, dependencies. This mirrors the discovery phase real migration engineers do first.

### Phase 2 — Build golden images with Packer
- Write a Packer template that takes a base AMI (Amazon Linux 2023 or Ubuntu) and provisions it to match your on-prem app server config (install same packages, configure same services) using shell provisioners or Ansible.
- Output: a versioned AMI you can reference in Terraform. This simulates "re-platforming" the image for cloud rather than raw copy.

### Phase 3 — Provision target infrastructure with Terraform
Build modules for:
- VPC with public + private subnets across 2 AZs
- Security groups mirroring on-prem firewall rules
- EC2 instances (app tier) from your Packer AMI, in an Auto Scaling Group
- RDS instance for the DB tier (or EC2 with MySQL if you want more control practice)
- NAT Gateway, Internet Gateway, route tables
- S3 bucket for the file server data

Keep this in a clean module structure (`modules/vpc`, `modules/compute`, `modules/database`) — this is what makes it portfolio-worthy versus a single flat `main.tf`.

### Phase 4 — Data migration
- DB: use `mysqldump`/`pg_dump` for an initial load, then a script to replay recent transactions for near-zero downtime (or just document a maintenance-window approach for v1 — simpler and fine for a portfolio piece).
- Files: `aws s3 sync` or `rclone` from your local file server to S3, with checksum validation after transfer.

### Phase 5 — Cutover automation
Write a runbook script (Python or Bash) that:
1. Runs pre-cutover health checks against the new AWS environment
2. Performs final data sync (delta only)
3. Updates DNS (use Route 53 — even a free `.xyz` domain works, or just simulate with a hosts-file style script)
4. Runs post-cutover validation (hits health endpoints, checks DB connectivity)
5. Has a documented rollback step if validation fails

### Budget guardrails (important with $200 credit)
- RDS + EC2 + NAT Gateway running 24/7 will burn credit fast — NAT Gateway alone is ~$32/month plus data processing. **Stop/destroy resources when not actively working** (`terraform destroy` between sessions).
- Stick to free-tier-eligible instance types (t2.micro/t3.micro) for the AWS side even though your on-prem VMs are large — the point is to prove the pipeline works, not to run production-scale AWS compute.
- Use AWS Budgets to set a $20–30 alert so you don't get surprised.

### Deliverables for your portfolio
- GitHub repo with Terraform modules + Packer templates
- Architecture diagram (before/after)
- A written runbook (README) documenting the migration process
- A short writeup: "what I'd do differently for a production migration" (shows judgment, not just execution)

Want me to help you scaffold the actual Terraform module structure and Packer template next, or do you want to get the on-prem VMs running first?