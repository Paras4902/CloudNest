# 🚀 CloudNest

> A three-tier web platform on AWS, built with Terraform modules, two environments, free-tier safe, $120 credit budget

![Terraform](https://img.shields.io/badge/Terraform-%3E%3D1.10-7B42BC?logo=terraform)
![AWS](https://img.shields.io/badge/AWS-ap--south--1-FF9900?logo=amazonaws)
![Status](https://img.shields.io/badge/status-in%20progress-yellow)

## 📌 Table of Contents

- [About the Project](#-about-the-project)
- [Features](#-features)
- [Architecture](#-architecture)
- [Tech Stack](#-tech-stack)
- [Project Structure](#-project-structure)
- [Getting Started](#-getting-started)
- [Installation](#-installation)
- [Usage](#-usage)
- [Break-It Drills](#-break-it-drills)
- [Troubleshooting](#-troubleshooting)
- [Cost Guardrails](#-cost-guardrails)
- [Screenshots](#-screenshots)
- [Environment Variables](#-environment-variables)
- [Contributing](#-contributing)
- [License](#-license)
- [Contact](#-contact)

---

## 📖 About the Project

CloudNest is a three-tier web platform (ALB → EC2 Auto Scaling → RDS MySQL) in `ap-south-1`, written entirely in Terraform modules and deployed to two environments (`dev`, `prod`) from the same code.

- **Purpose:** hands-on revision vehicle for the HashiCorp Terraform Associate (004) and AWS SAA-C03, and an interview-grade portfolio project.
- **Why:** concepts only stick after they fail in a real account. Every module is built, applied, broken, and destroyed on its own.
- **Who benefits:** anyone preparing for Terraform/AWS certifications who wants a realistic, cost-capped reference.
- **What makes it different:**
  - The database tier is **not decoration**: the app reads credentials from Secrets Manager and queries RDS, so IAM, SG chaining, and subnet routing are all exercised.
  - Free-tier safe by default: NAT is off, RDS is single-AZ in dev, and `prod` is **plan-only** or applied briefly.
  - Each build phase applies and destroys cleanly in isolation. If phase 5 breaks phase 4, the module boundary is wrong.

> **Status:** blueprint stage. Update this section as each build phase is verified.

## ✨ Features

- **Modular Terraform** – child modules with output chaining, `for_each`, `dynamic` blocks, `locals`, validation, lifecycle rules, pre/postconditions.
- **Two environments, one codebase** – `dev` (10.0.0.0/16) and `prod` (10.1.0.0/16) differ in sizing, not structure; non-overlapping CIDRs allow future peering.
- **Real DB path** – `/db` endpoint fetches the secret via the instance role, connects to RDS, and returns `SELECT VERSION()`.
- **Multi-AZ web tier** – ALB across 2 AZs, ASG with ELB health checks, CPU target-tracking at 50%, instance refresh on launch template changes.
- **Secure defaults** – IMDSv2 required, encrypted RDS and S3, private isolated DB subnets, no public DB access, least-privilege IAM scoped to ARNs.
- **No password in state** – `manage_master_user_password = true` (Secrets Manager) instead of `random_password`.
- **Remote state** – S3 backend with native lockfile (`use_lockfile`), bootstrapped separately.
- **Observability and cost control** – SNS alarms (ALB 5xx, unhealthy hosts, RDS CPU, RDS free storage), CloudWatch dashboard, budgets at $10 and $50 (actual + forecasted).
- **Optional** – CloudFront + Origin Access Control for the assets bucket; HCP Terraform migration.

## 🏗️ Architecture

```text
Internet
   |
[Internet Gateway]
   |
VPC 10.0.0.0/16 (dev)  |  10.1.0.0/16 (prod)
   |
   +- PUBLIC subnets      public-a 10.0.0.0/24 (AZ a), public-b 10.0.1.0/24 (AZ b)
   |    ALB + app instances      RT: 0.0.0.0/0 -> IGW
   |
   +- PRIVATE-APP subnets app-a 10.0.10.0/24, app-b 10.0.11.0/24
   |    empty unless enable_nat = true   RT: 0.0.0.0/0 -> NAT (optional)
   |
   +- PRIVATE-DB subnets  db-a 10.0.20.0/24, db-b 10.0.21.0/24
        isolated, local route only

S3 Gateway Endpoint -> associated with public + app route tables (no hourly charge)
```

**Traffic and security chain**

```text
Internet --80--> alb-sg --80--> app-sg --3306--> db-sg
                                  | egress: 443 -> SSM, Secrets Manager, S3 endpoint
```

| Security group | Ingress | Egress |
|---|---|---|
| `alb-sg` | TCP 80 from 0.0.0.0/0 | TCP 80 to `app-sg` |
| `app-sg` | TCP 80 from `alb-sg` only | Open (SSM, package installs) |
| `db-sg` | TCP 3306 from `app-sg` only | None required |

**App endpoints (served by nginx + a small Python service via `user_data`)**

| Path | Behavior |
|---|---|
| `/` | Returns instance ID and AZ (via IMDSv2). Refresh to watch it alternate across AZs. |
| `/health` | Returns 200 for the target group health check. |
| `/db` | Fetches the secret from Secrets Manager, connects to RDS, returns `SELECT VERSION()`. |

> **Known limitation:** EC2 sits in public subnets by default to avoid NAT cost. Acceptable for a lab, not a production pattern. Set `enable_nat = true` to place instances in private-app subnets.

## 🛠️ Tech Stack

**Infrastructure as Code:**
- Terraform `>= 1.10` (AWS provider)

**Compute and Networking:**
- VPC, subnets, IGW, route tables, S3 gateway endpoint
- ALB, Launch Template, Auto Scaling Group (t3.micro, Amazon Linux 2023)

**Application:**
- nginx + Python service (bootstrapped via `user_data`)

**Database:**
- RDS MySQL 8.0 (db.t3.micro, 20 GB gp3, encrypted)

**Security and Secrets:**
- IAM role + instance profile (SSM Core + scoped inline policy), Secrets Manager, Security Groups, IMDSv2

**Storage and Delivery:**
- S3 (versioned, SSE, lifecycle to IA at 30 days), optional CloudFront + OAC

**Monitoring and Cost:**
- CloudWatch alarms and dashboard, SNS, AWS Budgets

**Tools:**
- Git & GitHub
- AWS CLI, SSM Session Manager
- VS Code

## 📂 Project Structure

```text
cloudnest/
|-- bootstrap/              # state bucket, local state, applied once, never destroyed
|-- modules/
|   |-- vpc/                # VPC, 6 subnets, IGW, route tables, S3 endpoint, optional NAT
|   |-- security/           # 3 SGs + standalone rule resources
|   |-- iam/                # instance role, profile, scoped inline policy
|   |-- storage/            # assets bucket
|   |-- alb/                # ALB, target group, listener
|   |-- compute/            # AMI lookup, launch template, ASG, scaling policy
|   |-- rds/                # subnet group, MySQL instance
|   |-- monitoring/         # SNS, alarms, dashboard, budgets
|   `-- cdn/                # optional CloudFront + OAC
|                           # each module: main.tf, variables.tf, outputs.tf, versions.tf
`-- envs/
    |-- dev/                # main.tf, backend.tf, terraform.tfvars
    `-- prod/               # same modules, different sizing
```

**Conventions**
- Naming: `${var.project}-${var.env}-<resource>`
- `default_tags`: `Project=cloudnest`, `Env`, `ManagedBy=terraform`
- CIDRs generated with `cidrsubnet(var.vpc_cidr, 8, n)` and `for_each` over a map. Never hardcoded.
- AMI IDs resolved via SSM parameter `/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64`. Never hardcoded.
- SG rules use standalone `aws_vpc_security_group_ingress_rule` / `egress_rule` resources, not inline rules (avoids dependency cycles).

## 🚀 Getting Started

### Prerequisites

- Git
- Terraform `>= 1.10` (required for native S3 lockfile). Check with `terraform version`.
- AWS CLI v2 with a **dedicated IAM user or role** profile. Never run Terraform as root.
- An AWS account with Free Tier / credits. Check **Billing > Credits** to confirm which plan you are on.

Check free-tier-eligible instance types for your account instead of guessing:

```bash
aws ec2 describe-instance-types --filters Name=free-tier-eligible,Values=true \
  --query "InstanceTypes[*].InstanceType" --output text
```

### Installation

**1. Clone the repository**

```bash
git clone https://github.com/Paras4902/CloudNest.git
cd CloudNest
```

**2. Configure AWS credentials**

```bash
aws configure --profile cloudnest
export AWS_PROFILE=cloudnest
aws sts get-caller-identity   # confirm it is NOT the root account
```

**3. Create budget alerts first** (before building anything else): $10 and $50, actual + forecasted.

**4. Bootstrap the state bucket (once)**

```bash
cd bootstrap
terraform init
terraform apply
terraform output   # note the bucket name
```

**5. Point the dev backend at the bucket**

```hcl
# envs/dev/backend.tf
terraform {
  required_version = ">= 1.10"
  backend "s3" {
    bucket       = "cloudnest-state-<suffix>"
    key          = "dev/terraform.tfstate"
    region       = "ap-south-1"
    use_lockfile = true
    encrypt      = true
  }
}
```

**6. Deploy dev**

```bash
cd ../envs/dev
terraform init
terraform plan
terraform apply
```

## 💻 Usage

### Build order

Each step applies and destroys cleanly on its own.

| # | Build | Verify |
|---|---|---|
| 1 | `bootstrap/`: state bucket | `terraform output` shows the bucket name |
| 2 | `vpc` + `security` | `terraform state list` shows network resources |
| 3 | `iam` + `storage` | Role and bucket exist; re-run plan is clean |
| 4 | `alb` + `compute` | `curl http://<alb_dns>` returns the instance-ID page |
| 5 | `rds` + `/db` endpoint | `/db` returns the MySQL version string |
| 6 | `monitoring` | Confirm SNS email; trigger an alarm and receive it |
| 7 | `envs/prod` | **Plan only**; the diff should be sizing, not structure |
| 8 | Optional: `cdn`; HCP Terraform migration | `cloud {}` block replaces the S3 backend for dev |

### Verify the deployment

```bash
terraform output alb_dns_name
curl http://<alb_dns>/          # instance ID + AZ (refresh to see AZ alternate)
curl http://<alb_dns>/health    # 200
curl http://<alb_dns>/db        # MySQL version string
```

### Tear down (every session)

```bash
terraform destroy
```

Then check **EC2, ELB, and RDS in the console**. The console catches leaks Terraform does not know about.

For `prod`, `deletion_protection = true` blocks destroy. Flip it to `false` and apply before teardown.

## 🧪 Break-It Drills

| Drill | What to do |
|---|---|
| Drift | Edit an SG rule in the console, run `terraform plan`, read the diff, then compare with `plan -refresh-only`. |
| State surgery | `terraform state rm` the RDS module, then re-adopt it with an `import` block. |
| Safe refactor | Rename a module call and use a `moved` block so plan shows zero destroys. |
| Replace | `terraform apply -replace=<addr>` on one resource (modern taint). |
| Validation | Pass a bad CIDR and confirm the `validation` block rejects it at plan time. |
| Lock | Kill apply mid-run; recover with `terraform force-unlock <ID>` only after confirming no other run is active. |

## 🩺 Troubleshooting

| Symptom | Cause and fix |
|---|---|
| Targets stay unhealthy | `app-sg` lacks ingress from `alb-sg`, or `user_data` failed. Check `/var/log/cloud-init-output.log` via SSM Session Manager. |
| Changed `user_data`, nothing happened | Existing instances keep the old launch template version. Use `latest_version` and configure `instance_refresh`. |
| `/db` returns AccessDenied | Wrong secret ARN in the IAM policy. Use `aws_db_instance.this.master_user_secret[0].secret_arn`. |
| `/db` times out | `db-sg` does not allow 3306 from `app-sg`, or the instance is in a different VPC than expected. |
| SG dependency cycle error | Inline SG rules referencing each other. Move to standalone rule resources. |
| No instance in SSM Session Manager | No route to SSM endpoints (no public IP or NAT), or role lacks `AmazonSSMManagedInstanceCore`. |
| Alarms never email | SNS subscription still `PendingConfirmation`. Confirm it manually. |
| `BucketAlreadyExists` | S3 names are global. Add a `random_id` suffix. |
| `DBSubnetGroupDoesNotCoverEnoughAZs` | RDS subnet group needs subnets in at least 2 AZs. |
| ALB `ValidationError` on subnets | An ALB needs at least 2 AZs. |
| Instance type not eligible for Free Tier | Pick a type from the `describe-instance-types` command above. |
| Destroy fails on S3 | Bucket not empty. Use `force_destroy = true` in lab envs. |
| Destroy fails on RDS | Wants a final snapshot. Use `skip_final_snapshot = true` for the lab only. |
| Destroy hangs on VPC | Leftover ENI (ALB or endpoint) holds the subnet. Wait a few minutes and retry. |
| Error acquiring the state lock | Stale lock. `force-unlock` after confirming no active run. |
| Password in state | `random_password` is stored in plaintext in state even if marked sensitive. Hence `manage_master_user_password` (about $0.40 per secret per month). |

## 💰 Cost Guardrails

- Create `aws_budgets_budget` alerts ($10 and $50) before anything else.
- Filter Cost Explorer by tag `Project=cloudnest`.
- Run `terraform destroy` at the end of every session and verify in the console.
- Cost Explorer lags by up to about a day. A clean bill today does not prove yesterday was clean.
- Estimated cost with everything up: roughly **$0.10 to $0.15 per hour** (estimate, not measured; verify against your own bill).

**Do not create:**
- NAT Gateway (except deliberately testing the toggle for under an hour)
- Multi-AZ RDS, read replicas, or Aurora
- Route 53 hosted zone (flat monthly fee)
- Idle Elastic IPs (public IPv4 is billed hourly even when idle)

## 📸 Screenshots

Add after each verified phase:

- `screenshots/alb-instance-page.png`: `/` showing instance ID and AZ
- `screenshots/db-endpoint.png`: `/db` returning the MySQL version
- `screenshots/asg-scaling.png`: ASG target-tracking activity
- `screenshots/cloudwatch-dashboard.png`: monitoring dashboard
- `screenshots/terraform-plan-prod.png`: prod plan diff (sizing only)

![ALB response](screenshots/alb-instance-page.png)

## 🔐 Environment Variables

CloudNest is configured through Terraform variables per environment (`envs/<env>/terraform.tfvars`), not a `.env` file.

| Variable | dev | prod |
|---|---|---|
| `vpc_cidr` | `10.0.0.0/16` | `10.1.0.0/16` |
| `instance_type` | `t3.micro` | `t3.micro` |
| `asg_min / desired / max` | 1 / 1 / 2 | 2 / 2 / 4 |
| `enable_rds` | `true` | `true` |
| `rds_multi_az` | `false` | `true` (do not apply; not free) |
| `enable_nat` | `false` | `false` |
| `deletion_protection` | `false` | `true` |
| state key | `dev/terraform.tfstate` | `prod/terraform.tfstate` |

Shell environment:

```bash
export AWS_PROFILE=cloudnest
export AWS_REGION=ap-south-1
```

**Security notes**
- Never commit `*.tfstate`, `.terraform/`, or credentials. Add them to `.gitignore`.
- No DB password is ever written in code or state: RDS manages the master password in Secrets Manager.

## 🤝 Contributing

Contributions are welcome.

1. Fork the repository.
2. Create a new branch:

   ```bash
   git checkout -b feature/YourFeature
   ```

3. Commit your changes:

   ```bash
   git commit -m "Add: YourFeature"
   ```

4. Push to your branch:

   ```bash
   git push origin feature/YourFeature
   ```

5. Open a Pull Request.

**Ground rules**
- Each phase must apply and destroy cleanly alone.
- No copy-pasting registry modules on the first pass. Write from provider docs, then compare with `terraform-aws-modules`.
- After each session, add a 3-line "why it failed" note.

## 📄 License

This project is licensed under the MIT License.
See the LICENSE file for details.

## 📬 Contact

**Paras Udapurkar**

- GitHub: https://github.com/Paras4902
- LinkedIn: https://www.linkedin.com/in/parasudapurkar/
- Email: parasudapurkar2512@gmail.com

---

⭐ If you find this project useful, consider giving it a star!