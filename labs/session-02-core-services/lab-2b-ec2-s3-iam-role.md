# Lab 2B: Access S3 from EC2 Using an IAM Role

**Session:** 2 — Core AWS Services  
**Track:** Cloud Basics  
**Difficulty:** Intermediate  
**Estimated Time:** 25–30 minutes

---

## Overview

In this lab, you will launch an EC2 instance that can **read and write files to S3** — without putting any access keys on the server. Instead, you will use an **IAM role** attached to the instance. This is the secure, production-standard way to give cloud resources access to other services.

**What you will build:**
- An S3 bucket for storing files
- An EC2 instance with an IAM role that grants S3 access
- You will connect to the instance and upload/download files to S3 directly from the server

This is a real-world pattern — applications running on EC2 frequently need to read from or write to S3 (logs, uploads, backups, config files). Using IAM roles instead of access keys is an AWS security best practice.

---

## Prerequisites

- ✅ Completed **Lab 1A** (AWS account, CLI configured)
- ✅ Familiarity with Lab 2A concepts (EC2, IAM roles, Session Manager)
- ✅ AWS CLI authenticated

---

## Cost Notice

| Service | What It Is | Free Tier Limit |
|---------|-----------|----------------|
| Amazon EC2 (t2.micro) | Virtual server | 750 hours/month free for 12 months |
| Amazon S3 | Cloud storage | 5 GB storage free for 12 months |
| AWS Systems Manager | Remote connection | Always Free |
| IAM | Access management | Always Free |

**Estimated cost for this lab: $0.00**

---

## Concepts

**IAM Role for EC2** — Instead of storing access keys on a server (which is insecure — if someone hacks the server, they get the keys), you attach a role. The role automatically provides temporary credentials that rotate every few hours. The application on the server uses these credentials without you doing anything.

**Instance Profile** — The "container" that attaches an IAM role to an EC2 instance. You create the role, put it in an instance profile, and attach the profile when launching the instance.

**Why this matters:** In a real job, you would NEVER put access keys on an EC2 instance. You would always use IAM roles. This is tested on the AWS certifications and asked about in interviews.

---

## ⚠️ Placeholders in This Lab

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name | `AdministratorAccess-123456789012` |
| `<YOUR_UNIQUE_BUCKET_NAME>` | A globally unique S3 bucket name | `jane-doe-workshop-session2` |
| `<AMI_ID>` | The AMI ID from Step 3 | `ami-0a59ec92177ec3fad` |
| `<YOUR_INSTANCE_ID>` | The instance ID from Step 5 | `i-0abc123def456789` |

---

## Lab Steps

### Step 1: Set Your AWS Profile

**Windows (PowerShell):**

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**macOS / Linux:**

```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

---

### Step 2: Create an S3 Bucket

📋 Copy and paste, **replacing `<YOUR_UNIQUE_BUCKET_NAME>`**:

```
aws s3 mb s3://<YOUR_UNIQUE_BUCKET_NAME> --region us-east-1
```

**✅ You should see:** `make_bucket: <YOUR_UNIQUE_BUCKET_NAME>`

---

### Step 3: Get the Latest Amazon Linux AMI ID

📋 Copy and paste:

```
aws ssm get-parameters-by-path --path "/aws/service/ami-amazon-linux-latest" --region us-east-1 --query "Parameters[?Name=='/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64'].Value" --output text
```

> **📝 Write down the AMI ID:** ______________________________

---

### Step 4: Create an IAM Role with S3 and Session Manager Access

**Create the trust policy file** (`ec2-trust-policy.json`):

📋 Copy and paste into a new file and save:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

**Create the role and attach policies.** 📋 Copy and paste these commands one at a time:

```
aws iam create-role --role-name workshop-ec2-s3-role --assume-role-policy-document file://ec2-trust-policy.json
```

```
aws iam attach-role-policy --role-name workshop-ec2-s3-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

```
aws iam attach-role-policy --role-name workshop-ec2-s3-role --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
```

> **What does this do?** Creates a role with two permissions:
> - `AmazonSSMManagedInstanceCore` — allows Session Manager to connect
> - `AmazonS3FullAccess` — allows the instance to read/write any S3 bucket

```
aws iam create-instance-profile --instance-profile-name workshop-ec2-s3-profile
```

```
aws iam add-role-to-instance-profile --instance-profile-name workshop-ec2-s3-profile --role-name workshop-ec2-s3-role
```

**Wait 10 seconds** for IAM to propagate.

---

### Step 5: Launch the EC2 Instance

📋 Copy and paste, **replacing `<AMI_ID>`**:

**Windows (PowerShell):**

```powershell
aws ec2 run-instances --image-id <AMI_ID> --instance-type t2.micro --iam-instance-profile Name=workshop-ec2-s3-profile --region us-east-1 --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=workshop-s3-access}]" --query "Instances[0].InstanceId" --output text
```

**macOS / Linux:**

```bash
aws ec2 run-instances \
    --image-id <AMI_ID> \
    --instance-type t2.micro \
    --iam-instance-profile Name=workshop-ec2-s3-profile \
    --region us-east-1 \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=workshop-s3-access}]' \
    --query "Instances[0].InstanceId" \
    --output text
```

**✅ You should see** an instance ID like: `i-0abc123def456789`

> **📝 Write down your Instance ID:** ______________________________

---

### Step 6: Wait for the Instance and Connect

📋 Copy and paste, **replacing `<YOUR_INSTANCE_ID>`**:

```
aws ec2 wait instance-running --instance-ids <YOUR_INSTANCE_ID> --region us-east-1
```

**Wait 30–60 seconds** for Session Manager to register, then connect:

**Connect via the AWS Console:**
1. Go to **EC2** → **Instances**
2. Select your instance → click **Connect** → **Session Manager** tab → **Connect**

---

### Step 7: Upload a File to S3 from the EC2 Instance

You are now on the EC2 instance. Let's create a file and upload it to your S3 bucket.

📋 Copy and paste these commands **inside the Session Manager terminal** (on the EC2 instance), **replacing `<YOUR_UNIQUE_BUCKET_NAME>`**:

**Create a test file:**

```bash
echo "Hello from EC2! This file was uploaded from a virtual server in the cloud." > /tmp/hello-from-ec2.txt
```

**Upload it to S3:**

```bash
aws s3 cp /tmp/hello-from-ec2.txt s3://<YOUR_UNIQUE_BUCKET_NAME>/hello-from-ec2.txt
```

**✅ You should see:** `upload: /tmp/hello-from-ec2.txt to s3://<YOUR_UNIQUE_BUCKET_NAME>/hello-from-ec2.txt`

> **💡 Notice:** You did NOT configure any access keys on this instance. The `aws s3 cp` command works because the IAM role attached to the instance automatically provides temporary credentials. This is the secure way to give EC2 instances access to AWS services.

**List the files in your bucket:**

```bash
aws s3 ls s3://<YOUR_UNIQUE_BUCKET_NAME>/
```

**✅ You should see** your `hello-from-ec2.txt` file listed.

---

### Step 8: Download a File from S3 to the EC2 Instance

Let's also test downloading. First, create another file in S3 from your **local terminal** (not the EC2 session).

**Disconnect from Session Manager** (type `exit` or close the tab).

**On your local terminal,** create and upload a file:

📋 Copy and paste, **replacing `<YOUR_UNIQUE_BUCKET_NAME>`**:

```
echo "This file was uploaded from my local computer" > local-file.txt
```

> **Windows PowerShell:** `"This file was uploaded from my local computer" | Out-File -Encoding utf8 local-file.txt`

```
aws s3 cp local-file.txt s3://<YOUR_UNIQUE_BUCKET_NAME>/local-file.txt
```

**Now reconnect to the EC2 instance** (via Console → EC2 → Connect → Session Manager).

**Download the file from S3 to the instance:**

```bash
aws s3 cp s3://<YOUR_UNIQUE_BUCKET_NAME>/local-file.txt /tmp/downloaded-file.txt
```

**Read it:**

```bash
cat /tmp/downloaded-file.txt
```

**✅ You should see:** `This file was uploaded from my local computer`

> **🎉 You just demonstrated a real-world pattern:** Your local machine uploaded a file to S3, and your EC2 instance downloaded it. This is how applications share data — through S3 as a central storage layer.

---

### Step 9: Verify in the AWS Console

**✅ Checkpoint:**
1. Go to **S3** in the AWS Console
2. Click on your bucket
3. You should see both files: `hello-from-ec2.txt` and `local-file.txt`
4. Click on `hello-from-ec2.txt` → click **Download** or **Open** to verify the content

---

## What You Just Did

You built a secure connection between an EC2 instance and S3 using an IAM role — the production-standard approach. You:

1. Created an IAM role with S3 permissions
2. Launched an EC2 instance with that role attached
3. Uploaded a file from the instance to S3 (no access keys needed)
4. Downloaded a file from S3 to the instance
5. Demonstrated the pattern of using S3 as shared storage between your local machine and cloud servers

This is how real applications work — web servers store user uploads in S3, batch processing jobs read input from S3 and write results back, and deployment pipelines pull artifacts from S3.

---

## Cert Prep Callout

**Target Certification:** AWS Solutions Architect – Associate (SAA)

IAM roles for EC2 are heavily tested on the SAA exam. The exam expects you to know that roles (not access keys) are the correct way to grant EC2 instances access to other AWS services.

**Sample question type:** "An application running on EC2 needs to access an S3 bucket. What is the most secure way to grant this access?"

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `Unable to locate credentials` on the EC2 instance | The IAM role is not attached or hasn't propagated | Wait 1–2 minutes. Verify the instance profile is attached: check EC2 → Instances → select instance → Security tab → IAM Role |
| `Access Denied` when accessing S3 | The role doesn't have S3 permissions | Verify you attached `AmazonS3FullAccess` to the role |
| Session Manager won't connect | SSM agent hasn't registered | Wait 1–2 minutes after instance is running. Verify `AmazonSSMManagedInstanceCore` is attached to the role. |

---

## Cleanup

### Step 1: Disconnect from Session Manager

Type `exit` or close the browser tab.

### Step 2: Terminate the Instance

📋 Copy and paste, **replacing `<YOUR_INSTANCE_ID>`**:

```
aws ec2 terminate-instances --instance-ids <YOUR_INSTANCE_ID> --region us-east-1
```

### Step 3: Empty and Delete the S3 Bucket

📋 Copy and paste, **replacing `<YOUR_UNIQUE_BUCKET_NAME>`**:

```
aws s3 rm s3://<YOUR_UNIQUE_BUCKET_NAME> --recursive
```

```
aws s3 rb s3://<YOUR_UNIQUE_BUCKET_NAME>
```

### Step 4: Delete the IAM Role and Instance Profile

📋 Copy and paste these commands in order:

```
aws iam detach-role-policy --role-name workshop-ec2-s3-role --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
```

```
aws iam detach-role-policy --role-name workshop-ec2-s3-role --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
```

```
aws iam remove-role-from-instance-profile --instance-profile-name workshop-ec2-s3-profile --role-name workshop-ec2-s3-role
```

```
aws iam delete-instance-profile --instance-profile-name workshop-ec2-s3-profile
```

```
aws iam delete-role --role-name workshop-ec2-s3-role
```

### Step 5: Delete Local Files

**macOS / Linux:** `rm ec2-trust-policy.json local-file.txt`

**Windows:** `Remove-Item ec2-trust-policy.json, local-file.txt`

### Step 6: Verify Cleanup

**✅ Checkpoint:**
1. **EC2** → Instances → your instance shows **Terminated**
2. **S3** → your bucket is gone
3. **IAM** → Roles → `workshop-ec2-s3-role` is gone

---

## Help

If you get stuck, post your question in the **Lab Help** channel on Microsoft Teams. Include:
1. The **command** you ran
2. The **full error message**
3. Which **step number** you are on
