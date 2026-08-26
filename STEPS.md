# 🛠️ Deployment Steps

Full, reproducible steps to deploy this project from scratch in your own AWS account.

> All resources below were created in `us-east-1`. Adjust the region if you deploy elsewhere.

---

## Step 1 — Create Key Pair
- EC2 → Key Pairs → Create key pair
- Name: `ats-keypair` | Format: `.pem`
- Download and store the `.pem` file securely

## Step 2 — Create VPC
- VPC → Create VPC → **VPC only**
- Name tag: `ats-cv-vpc`
- IPv4 CIDR block: `10.0.0.0/16`
- Confirm **DNS hostnames** and **DNS resolution** are enabled

## Step 3 — Create Subnets

| Name | Availability Zone | CIDR |
|---|---|---|
| `ats-public-subnet-1` | us-east-1a | `10.0.1.0/24` |
| `ats-public-subnet-2` | us-east-1b | `10.0.2.0/24` |

- For both subnets: **Actions → Edit subnet settings → Enable auto-assign public IPv4 address**

## Step 4 — Create Internet Gateway
- VPC → Internet Gateways → Create internet gateway → `ats-igw`
- Actions → Attach to VPC → `ats-cv-vpc`

## Step 5 — Create Route Table
- VPC → Route Tables → Create route table → `ats-public-rt` (VPC: `ats-cv-vpc`)
- Routes → Edit routes → Add route: `0.0.0.0/0` → Target: `ats-igw`
- Subnet associations → Edit → select both public subnets → Save

## Step 6 — Create Security Groups

**ALB security group — `ats-alb-sg`**
| Direction | Type | Port | Source |
|---|---|---|---|
| Inbound | HTTP | 80 | `0.0.0.0/0` |

**EC2 security group — `ats-ec2-sg`**
| Direction | Type | Port | Source |
|---|---|---|---|
| Inbound | HTTP | 80 | `ats-alb-sg` (security group reference, not an IP) |
| Inbound | SSH | 22 | Admin IP / `0.0.0.0/0` for lab use only |

> This chaining is what forces all HTTP traffic to go through the load balancer — the EC2 instances are not directly reachable on port 80 from the internet.

## Step 7 — Launch EC2 Instances

Launch **two** `t2.micro` Amazon Linux 2023 instances:

| Setting | Instance 1 | Instance 2 |
|---|---|---|
| Name | `ats-ec2-1` | `ats-ec2-2` |
| Subnet | `ats-public-subnet-1` | `ats-public-subnet-2` |
| Security group | `ats-ec2-sg` | `ats-ec2-sg` |
| Key pair | `ats-keypair` | `ats-keypair` |

User data (Advanced details) for both instances:
```bash
#!/bin/bash
yum update -y
yum install -y python3 python3-pip
pip3 install flask requests
mkdir -p /home/ec2-user/ats-app/templates
```

## Step 8 — Target Group & Load Balancer

**Target Group**
- Type: Instances | Name: `ats-target-group` | Protocol: HTTP | Port: 80 | VPC: `ats-cv-vpc`
- Health check path: `/health`
- Register both EC2 instances

**Application Load Balancer**
- Name: `ats-alb` | Scheme: Internet-facing | VPC: `ats-cv-vpc`
- Subnets: both public subnets | Security group: `ats-alb-sg`
- Listener: HTTP:80 → forward to `ats-target-group`
- Note the **DNS name** — this is the application URL

## Step 9 — Create S3 Bucket
- Name: `ats-cv-storage-{YOUR_ACCOUNT_ID}` (must be globally unique)
- Same region as everything else
- Keep default settings (Block Public Access enabled)

## Step 10 — Create DynamoDB Table
- Table name: `ats-cv-records`
- Partition key: `cv_id` (String)
- Capacity mode: On-demand

## Step 11 — IAM Role for Lambda
- IAM → Roles → Create role → AWS service → Lambda
- Attach a custom policy (see [`IAM/lambda-policy.json`](../IAM/lambda-policy.json)) scoped to:
  - CloudWatch Logs (create log group/stream, put log events)
  - `s3:PutObject` / `s3:GetObject` on the specific bucket only
  - `dynamodb:PutItem` / `GetItem` / `UpdateItem` on the specific table only
- Role name: `ats-lambda-role`

## Step 12 — Lambda Function 1: CV Generator
- Name: `ats-cv-generator` | Runtime: Python 3.12 | Role: `ats-lambda-role`
- Code: [`code/lambda_generator/lambda_function.py`](../code/lambda_generator/lambda_function.py)
- Environment variables:
  - `BUCKET_NAME` = `ats-cv-storage-{YOUR_ACCOUNT_ID}`
  - `TABLE_NAME` = `ats-cv-records`
- General configuration: Timeout 30s, Memory 256 MB

## Step 13 — Lambda Function 2: JD Analyzer
- Name: `ats-jd-analyzer` | Runtime: Python 3.12 | Role: `ats-lambda-role`
- Code: [`code/lambda_analyzer/lambda_function.py`](../code/lambda_analyzer/lambda_function.py)
- Environment variable: `TABLE_NAME` = `ats-cv-records`

## Step 14 — API Gateway
- Create REST API → `ats-cv-api` (Regional)
- Resource `/generate` → POST → Lambda proxy → `ats-cv-generator` → Enable CORS
- Resource `/analyze` → POST → Lambda proxy → `ats-jd-analyzer` → Enable CORS
- Deploy API → new stage `prod` → note the **Invoke URL**

## Step 15 — Deploy Flask App on EC2

Connect to **each** instance via **EC2 Instance Connect** and run:

```bash
mkdir -p /home/ec2-user/ats-app/templates
cd /home/ec2-user/ats-app
sudo pip3 install flask requests
```

Deploy [`code/backend/app.py`](../code/backend/app.py) to `/home/ec2-user/ats-app/app.py`
and [`code/backend/templates/index.html`](../code/backend/templates/index.html) to
`/home/ec2-user/ats-app/templates/index.html`.

> ⚠️ **Gotcha:** the `UserData` script creates `/home/ec2-user/ats-app` as `root`, so writing files there as `ec2-user` will fail with `Permission denied`. Use `sudo` when creating the files (see `CONCEPTS.md` for the full story).

Create the systemd service so the app survives reboots and restarts on failure:

```ini
# /etc/systemd/system/ats-app.service
[Unit]
Description=ATS Flask App
After=network.target

[Service]
User=root
WorkingDirectory=/home/ec2-user/ats-app
Environment="API_GATEWAY_URL=https://YOUR_API_ID.execute-api.us-east-1.amazonaws.com/prod"
ExecStart=/usr/bin/python3 /home/ec2-user/ats-app/app.py
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable ats-app
sudo systemctl start ats-app
sudo systemctl status ats-app
```

Repeat on both instances.

## Step 16 — Verify & Test
1. EC2 → Target Groups → `ats-target-group` → Targets tab → wait for both to show **healthy**
2. Open the ALB DNS name in a browser
3. Fill in the form and click **Generate CV** → confirm a presigned S3 download link is returned
4. Paste a job description and click **Analyze** → confirm a match score, missing keywords, and a suggestion are returned

---

## 🧹 Cleanup Order

To avoid ongoing charges, delete resources in this order:
1. Load Balancer → Target Group
2. EC2 instances
3. NAT Gateway (if used) → Internet Gateway detach/delete
4. Lambda functions → IAM role/policy
5. API Gateway
6. S3 bucket (empty it first) → DynamoDB table
7. Subnets → Route Tables → VPC
