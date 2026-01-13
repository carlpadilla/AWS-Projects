
# AWS SAA – Week 1, Day 2 Lab
## EC2 → S3 Read-Only (One Bucket Only) using IAM Role + STS + SSM (No SSH Keys)

**Goal:** Launch an EC2 instance that can **only** read from **one specific S3 bucket** using an **IAM role** (STS temporary credentials).  
**Access method:** **AWS Systems Manager (SSM) Session Manager** (no SSH, no key pair).  
**Success criteria:**  
- ✅ EC2 can `ls` and `cp` objects from **your** bucket  
- ✅ EC2 gets **AccessDenied** trying to access other buckets  
- ✅ `aws sts get-caller-identity` shows `assumed-role/Role-EC2-S3ReadWeek1/...`

---

## Architecture (What you’re building)

```

EC2 Instance
|
| (Instance Profile / Role)
v
IAM Role: Role-EC2-S3ReadWeek1
|
| Policies:
| - AmazonSSMManagedInstanceCore (SSM connectivity)
| - Policy-S3Read-Week1Bucket (least privilege S3 access)
v
STS issues temporary credentials automatically

```

---

## Prerequisites (must be true before you start)

### A) Region
Pick ONE region and stay there the whole lab (example: `us-east-2`).

### B) VPC Baseline (must exist)
You need a lab VPC with a working public subnet.

Minimum VPC requirements:
- VPC: `vpc-saa-wk1` (CIDR example: `10.10.0.0/16`)
- Public subnet: `subnet-saa-wk1-public-1a` (example CIDR: `10.10.1.0/24`)
- Internet Gateway attached to the VPC
- Public route table associated to the public subnet with:
  - `0.0.0.0/0 → Internet Gateway`
- Public subnet **Auto-assign public IPv4** enabled
- VPC settings:
  - Enable DNS resolution = **ON**
  - Enable DNS hostnames = **ON**

> If any of these are wrong, the instance may have no public IP and/or won’t register in SSM.

---

# Part 1 — Create the S3 Bucket + Test File (Console)

## 1. Create bucket
**S3 → Buckets → Create bucket**
- Bucket name: `saa-wk1` (or any globally unique name)
- Region: same as your lab region
- “Block all public access”: **ON**
- Create bucket

## 2. Upload a test object
Inside the bucket:
- Upload file: `saa-test.txt`
- Example contents:
```

this is a test for S3

````

---

# Part 2 — Create the Least Privilege IAM Policy (Console)

## 3. Create policy
**IAM → Policies → Create policy → JSON**

Paste and replace `YOUR_BUCKET_NAME`:

```json
{
"Version": "2012-10-17",
"Statement": [
  {
    "Sid": "ListOnlyThisBucket",
    "Effect": "Allow",
    "Action": "s3:ListBucket",
    "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME"
  },
  {
    "Sid": "GetOnlyObjectsInThisBucket",
    "Effect": "Allow",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME/*"
  }
]
}
````

**Name:** `Policy-S3Read-Week1Bucket`
**Create policy**

### Why there are two statements (exam-relevant)

* `s3:ListBucket` applies to the **bucket** ARN: `arn:aws:s3:::bucket`
* `s3:GetObject` applies to the **object** ARN: `arn:aws:s3:::bucket/*`

---

# Part 3 — Create the EC2 Role (Console)

## 4. Create role

**IAM → Roles → Create role**

* Use case: **EC2**
* Attach permissions:

  * ✅ `AmazonSSMManagedInstanceCore` (AWS managed)
  * ✅ `Policy-S3Read-Week1Bucket` (your custom policy)
* Role name: `Role-EC2-S3ReadWeek1`
* Create role

> If you forget `AmazonSSMManagedInstanceCore`, the instance will not show up in Fleet Manager / Session Manager.

---

# Part 4 — Create the Security Group (Console)

## 5. Create security group

**EC2 → Security Groups → Create security group**

* Name: `saa-wk1-public-ssm`
* Description: `SSM-only public instance SG for saa week1`
* VPC: `vpc-saa-wk1`

Rules:

* Inbound: **none** (SSM does not require inbound)
* Outbound: allow all (default is fine)

  * `All traffic → 0.0.0.0/0`

Create security group.

---

# Part 5 — Launch EC2 Instance (Console)

## 6. Launch instance

**EC2 → Instances → Launch instance**

* Name: `ec2-saa-wk1-day2`
* AMI: Amazon Linux 2023
* Type: `t3.micro` (or `t2.micro`)
* Key pair: **Proceed without a key pair** (SSM only)

### Network settings (critical)

Click **Edit**:

* VPC: `vpc-saa-wk1`
* Subnet: `subnet-saa-wk1-public-1a`
* Auto-assign public IP: **Enable**
* Security group: **Select existing** → `saa-wk1-public-ssm`

### Advanced details (critical)

* IAM instance profile: `Role-EC2-S3ReadWeek1`

Launch instance.

## 7. Verify instance details

Open instance details and confirm:

* **Subnet** = public subnet
* **Public IPv4** is present
* **IAM Role** = `Role-EC2-S3ReadWeek1`

---

# Part 6 — Connect with SSM (Console)

## 8. Confirm it appears in Fleet Manager

**Systems Manager → Fleet Manager → Managed nodes**

* Wait 1–5 minutes
* Instance should appear **Online**

> If it doesn’t appear, rebooting the instance can force SSM agent registration (you experienced this).

## 9. Start a session

**Systems Manager → Session Manager → Start session**

* Select `ec2-saa-wk1-day2`
* Start session

You should now see a shell prompt (example: `sh-5.2$`).

---

# Part 7 — Validation Commands (Run inside the SSM session)

## 10. Confirm role identity via STS (proof you’re using the role)

```bash
aws sts get-caller-identity
```

✅ Expected: ARN contains:

```
assumed-role/Role-EC2-S3ReadWeek1/...
```

## 11. List your bucket (should work)

```bash
aws s3 ls s3://YOUR_BUCKET_NAME
```

✅ Expected: shows `saa-test.txt`

## 12. Download and read the file (should work)

```bash
aws s3 cp s3://YOUR_BUCKET_NAME/saa-test.txt .
cat saa-test.txt
```

✅ Expected: prints your test content.

## 13. Attempt to access a bucket you do NOT have permission to (must fail)

```bash
aws s3 ls s3://aws-cli
```

✅ Expected:

```
AccessDenied
```

## 14. Optional: confirm temporary creds are present (SSM/role session)

```bash
env | grep AWS
```

✅ Expected: shows temp env vars like:

* `AWS_ACCESS_KEY_ID`
* `AWS_SECRET_ACCESS_KEY`
* `AWS_SESSION_TOKEN`

---

# Common Failure Points + Fixes (Quick Troubleshooting)

## A) No public IP on instance

Cause: launched into private subnet OR public subnet auto-assign disabled.

Fix:

* Ensure subnet is public
* Enable auto-assign public IPv4 on public subnet
* Relaunch instance (public IP assignment is easiest at launch)

## B) Instance not showing in Fleet Manager

Most common causes:

* Role missing `AmazonSSMManagedInstanceCore`
* Instance can’t reach AWS (bad routes / no IGW / DNS off)
* SSM agent stuck (reboot often fixes)

Fix checklist:

* Confirm instance role is attached
* Confirm role has `AmazonSSMManagedInstanceCore`
* Confirm VPC DNS hostnames/resolution enabled
* Confirm public route: `0.0.0.0/0 → IGW`
* Reboot instance

## C) S3 access fails to your bucket

Cause: policy ARN wrong (bucket vs objects).

Fix:

* `ListBucket` → `arn:aws:s3:::BUCKET`
* `GetObject` → `arn:aws:s3:::BUCKET/*`
* Confirm bucket name matches exactly

---

# Cleanup (Avoid Charges)

## Minimum cleanup

* Terminate EC2 instance:

  * **EC2 → Instances → Instance state → Terminate**

## Optional cleanup (if you want a clean slate)

* Delete S3 bucket (must empty it first)
* Delete IAM role and policy (not required; can reuse for future labs)

---

# Day 2 Completion Evidence (What to save in your notes)

Paste these outputs into your study notes:

1. `aws sts get-caller-identity` (shows assumed role)
2. `aws s3 ls s3://YOUR_BUCKET_NAME` (succeeds)
3. `aws s3 ls s3://aws-cli` (AccessDenied)


