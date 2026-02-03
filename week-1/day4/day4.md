````markdown
# AWS SAA – Week 1, Day 4 Lab
## Private Subnet EC2 (No Public IP) + NO NAT + VPC Endpoints (SSM + S3) + Proof Tests

**Goal:** Run an EC2 instance in a **private subnet** (no public IP) with **NO NAT Gateway**, while still being able to:
- ✅ Connect to the instance using **SSM Session Manager**
- ✅ Access **S3** (your lab bucket) privately

**How:**  
- **S3 Gateway VPC Endpoint** (adds a route in your private route table)  
- **SSM Interface VPC Endpoints** (private ENIs in your subnet):
  - `ssm`
  - `ec2messages`
  - `ssmmessages`

**Success criteria (proof):**
- ✅ Instance has **no public IP**
- ✅ Instance shows **Online** in **SSM Fleet Manager / Managed nodes**
- ✅ Session Manager session works
- ✅ `aws s3 ls` + `aws s3 cp` to your bucket works
- ✅ `curl https://aws.amazon.com` **fails/timeouts** (no internet path)

---

# Before You Start (Cost + Sanity)

## Cost Warning
- **NAT Gateway** costs real money hourly + data.
- **Interface VPC endpoints** also cost hourly (usually cheaper than NAT for many use cases, but still not free).
- For labs: build → test → document → cleanup.

## Prerequisites
You should already have (from Days 1–3):
- VPC: `vpc-saa-wk1`
- Subnets:
  - Public: `subnet-saa-wk1-public-1a`
  - Private: `subnet-saa-wk1-private-1a`
- IGW attached to VPC: `igw-saa-wk1`
- Route tables:
  - Public RT: `rt-saa-wk1-public` with `0.0.0.0/0 → IGW`
  - Private RT: `rt-saa-wk1-private` associated to private subnet
- IAM role:
  - `Role-EC2-S3ReadWeek1` includes:
    - `AmazonSSMManagedInstanceCore`
    - Your S3 read policy for your bucket (example: `Policy-S3Read-Week1Bucket`)
- S3 bucket exists (example: `carl-saa-wk1`) with a test object (example: `saa-test.txt`)

---

# Part 0 — Remove NAT (So We Prove Endpoints Are Doing the Work)

> Day 4 is not “NAT + endpoints”. It’s “endpoints only”.

## Step 0.1 Delete NAT Gateway (if it exists)
1. Go to **VPC → NAT gateways**
2. If you see `nat-saa-wk1-1a`:
   - Select it
   - Actions → **Delete NAT gateway**
3. Wait until it fully deletes (it may show “deleting” for a few minutes)

## Step 0.2 Release Elastic IP (if it exists)
1. Go to **VPC → Elastic IPs**
2. Select the EIP used by NAT (example tag `eip-saa-wk1-nat`)
3. Actions → **Release Elastic IP address**
4. Confirm

## Step 0.3 Remove default route to NAT from private route table (if present)
1. Go to **VPC → Route tables**
2. Select `rt-saa-wk1-private`
3. Routes tab → confirm you **DO NOT** have:
   - `0.0.0.0/0 → nat-...`
4. If it exists:
   - Edit routes → delete that route → Save

✅ After this, a private instance will have **no internet**.

---

# Part 1 — Create S3 Gateway VPC Endpoint (Private Access to S3)

## Step 1.1 Create the endpoint
1. Go to **VPC → Endpoints**
2. Click **Create endpoint**
3. Under **Service category**, choose **AWS services**
4. In search box type: `s3`
5. Select the service:  
   - `com.amazonaws.<your-region>.s3`
6. Endpoint type must be:
   - ✅ **Gateway**

## Step 1.2 Choose VPC
- VPC: `vpc-saa-wk1`

## Step 1.3 Select route tables (critical)
Under **Route tables**, select:
- ✅ `rt-saa-wk1-private`

> This is what makes S3 reachable from the private subnet.

## Step 1.4 Endpoint policy
Keep it simple today:
- Select **Full access** (default)

## Step 1.5 Create endpoint
Click **Create endpoint**

## Step 1.6 Validate route table changed
1. Go to **VPC → Route tables**
2. Select: `rt-saa-wk1-private`
3. Routes tab: you should now see an entry that looks like:
   - `pl-xxxxxxxx (prefix list) → vpce-xxxxxxxx`
This indicates **S3 traffic is routed to the endpoint**, not to the internet.

---

# Part 2 — Create Interface Endpoints for SSM (So Session Manager Works Without Internet)

You must create **three** Interface endpoints:
- `ssm`
- `ec2messages`
- `ssmmessages`

If you skip one, Session Manager will break.

---

## Step 2.1 Create a Security Group for the VPC Endpoints
Interface endpoints are ENIs in your subnet and require a security group.

1. Go to **EC2 → Security Groups**
2. Click **Create security group**
3. Fill:
   - Name: `sg-saa-wk1-vpce`
   - Description: `Allow HTTPS from private instances to VPC endpoints`
   - VPC: `vpc-saa-wk1`

### Inbound rules (important)
Add rule:
- Type: **HTTPS**
- Port: **443**
- Source: **your instance security group** (recommended), e.g. `saa-wk1-public-ssm`

If AWS UI won’t let you pick SG as source, use VPC CIDR:
- Source: `10.10.0.0/16`

### Outbound rules
Leave default:
- All traffic → 0.0.0.0/0

Create SG.

---

## Step 2.2 Create Interface Endpoint: `ssm`
1. **VPC → Endpoints → Create endpoint**
2. Service category: **AWS services**
3. Search: `ssm`
4. Select: `com.amazonaws.<region>.ssm`
5. Endpoint type: ✅ **Interface**
6. VPC: `vpc-saa-wk1`

### Subnets
Select:
- ✅ `subnet-saa-wk1-private-1a`

### Security groups
Select:
- ✅ `sg-saa-wk1-vpce`

### Private DNS (critical)
- ✅ Enable **Private DNS** (sometimes labeled “Enable DNS name”)

Create endpoint.

---

## Step 2.3 Create Interface Endpoint: `ec2messages`
Repeat the same steps:
1. Create endpoint
2. Search: `ec2messages`
3. Select: `com.amazonaws.<region>.ec2messages`
4. Type: Interface
5. VPC: `vpc-saa-wk1`
6. Subnet: `subnet-saa-wk1-private-1a`
7. SG: `sg-saa-wk1-vpce`
8. Private DNS: ✅ enabled
9. Create

---

## Step 2.4 Create Interface Endpoint: `ssmmessages`
Repeat again:
1. Create endpoint
2. Search: `ssmmessages`
3. Select: `com.amazonaws.<region>.ssmmessages`
4. Same subnet + SG + Private DNS enabled
5. Create

---

# Part 3 — Launch the Day 4 EC2 Instance (Private Subnet, No Public IP)

You can reuse your Day 3 private instance if you kept it, but **cleanest** is launching a new one.

## Step 3.1 Launch instance
Go to **EC2 → Instances → Launch instance**

Set:
- Name: `ec2-saa-wk1-day4-private-vpce`
- AMI: Amazon Linux 2023
- Type: `t3.micro` (or `t2.micro`)
- Key pair: **Proceed without a key pair** ✅

### Network settings (must be exact)
Click **Edit**:
- VPC: `vpc-saa-wk1`
- Subnet: ✅ `subnet-saa-wk1-private-1a`
- Auto-assign public IP: **Disabled**
- Security group: `saa-wk1-public-ssm` (or your standard “instance SG”)

### Advanced details
- IAM instance profile: ✅ `Role-EC2-S3ReadWeek1`

Launch instance.

## Step 3.2 Validate it is private
Open instance details:
- Public IPv4: **blank / none**
- Private IP: `10.10.2.x`
- Subnet: private subnet

---

# Part 4 — Validate SSM Works Without NAT

## Step 4.1 Confirm it appears in Managed nodes
1. Go to **Systems Manager → Fleet Manager → Managed nodes**
2. Wait 1–5 minutes
3. Confirm instance shows **Online**

## Step 4.2 Start Session Manager session
1. **Systems Manager → Session Manager**
2. Click **Start session**
3. Select `ec2-saa-wk1-day4-private-vpce`

✅ If this works without NAT, your SSM endpoints are correct.

---

# Part 5 — Proof Tests (Run Inside the SSM Session)

> These tests prove that AWS private connectivity works AND the internet is blocked.

## Test 1 — Confirm role identity (STS)
```bash
aws sts get-caller-identity
````

✅ Expected: ARN contains:

* `assumed-role/Role-EC2-S3ReadWeek1/...`

## Test 2 — Confirm S3 access works (via Gateway Endpoint)

Replace `YOUR_BUCKET_NAME`:

```bash
aws s3 ls s3://YOUR_BUCKET_NAME
aws s3 cp s3://YOUR_BUCKET_NAME/saa-test.txt .
cat saa-test.txt
```

✅ Expected:

* List succeeds
* Copy succeeds
* File prints

## Test 3 — Confirm internet is NOT reachable (should fail/timeout)

```bash
curl -I https://aws.amazon.com
```

✅ Expected:

* Timeout / failure (no NAT, no IGW route in private RT)

This proves:

* S3 is reachable privately
* SSM is reachable privately
* The internet is not reachable (no egress)

---

# Troubleshooting (Fast Fix Map)

## If instance does NOT show in Managed nodes / Session Manager

Check in this exact order:

1. Interface endpoints exist:

* `ssm`
* `ec2messages`
* `ssmmessages`

2. Private DNS is enabled on each endpoint:

* VPC → Endpoints → select endpoint → verify “Private DNS enabled”

3. Endpoint SG inbound:

* `sg-saa-wk1-vpce` inbound allows **443** from instance SG (or VPC CIDR)

4. Instance IAM role includes:

* `AmazonSSMManagedInstanceCore`

5. VPC DNS settings:

* DNS resolution ON
* DNS hostnames ON

6. Reboot instance (SSM agent can stick):

* EC2 → Instance state → Reboot

## If S3 access fails but SSM works

Check:

* S3 endpoint is **Gateway** type
* You selected **private route table** when creating S3 endpoint
* The private RT now contains a `pl-... → vpce-...` route
* IAM policy for bucket is correct

---

# Cleanup (Stop Ongoing Endpoint Costs)

Interface endpoints cost hourly. If you’re done for the day:

## Step C.1 Delete Interface endpoints

VPC → Endpoints:

* Delete:

  * `com.amazonaws.<region>.ssm`
  * `com.amazonaws.<region>.ec2messages`
  * `com.amazonaws.<region>.ssmmessages`

## Step C.2 Delete Gateway endpoint (optional)

* Delete the S3 Gateway endpoint (optional; gateway endpoints have no hourly cost but keep your environment clean)

## Step C.3 Terminate the Day 4 instance

EC2 → Instances → Terminate `ec2-saa-wk1-day4-private-vpce`

---

# Day 4 Completion Evidence (Save these outputs)

From the Day 4 private instance SSM session save:

1. `aws sts get-caller-identity`
2. `aws s3 ls s3://YOUR_BUCKET_NAME`
3. `curl -I https://aws.amazon.com` (shows timeout/failure)

✅ If all three behave as expected, Day 4 is complete.

