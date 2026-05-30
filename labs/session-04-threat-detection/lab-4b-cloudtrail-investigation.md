# Lab 4B: Investigate Activity with CloudTrail

**Session:** 4 — Threat Detection & AWS Security Services  
**Track:** Cloud Security & IR  
**Difficulty:** Intermediate  
**Estimated Time:** 25–30 minutes  
**Target Cert:** AWS Security Specialty

---

## Overview

In this lab, you will:

1. **Set up CloudTrail** — create a trail that records all API activity to an S3 bucket
2. **Generate suspicious activity** — simulate actions an attacker might take
3. **Investigate with CloudTrail** — query the audit log to find out what happened
4. **Prove immutability** — demonstrate that even deleted users leave a permanent audit trail

By the end of this lab, you will understand how CloudTrail provides an immutable record of everything that happens in your AWS account — and why attackers cannot cover their tracks in a properly configured cloud environment.

---

## Prerequisites

- ✅ Completed **Lab 1A** (AWS CLI installed and configured)
- ✅ AWS CLI authenticated — run `aws sts get-caller-identity` and confirm it returns your account info
- ✅ A text editor to create JSON files (VS Code, Notepad, or any editor)

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| CloudTrail | Audit logging for AWS actions | First active trail is Always Free |
| Amazon S3 | Storage for CloudTrail logs | 0.023 per GB |
| IAM | Identity and Access Management | Always Free |

**Estimated cost for this lab: $0.00** - as you would have already created buckets from previous labs. Make sure your active trail from lab 3c has been deleted.

This command will check for existing trails:
```
aws cloudtrail describe-trails --query "trailList[].{Name:Name,Status:Status}" --region us-east-1
```
If it returns `[]` then there are none active.

---

## Concepts

Before we start, here is what each concept means:

**AWS CloudTrail** is the audit log of everything that happens in your AWS account. Every API call — whether from the console, CLI, or SDK — is recorded: who made the call, what they did, when they did it, and from where. Think of it as a flight recorder (black box) for your cloud environment.

**Trail** is a CloudTrail configuration that delivers log files to an S3 bucket. Without a trail, you can only see the last 90 days of management events in the Event History. With a trail, logs are stored permanently in S3.

**Event History** is CloudTrail's built-in view of the last 90 days of management events. You can search and filter events without creating a trail. This is what we will use for investigation in this lab.

**Lookup Events** is the CloudTrail API that lets you search the event history. You can filter by event name (what happened), username (who did it), resource type, and more. This is the primary tool for forensic investigation.

**Immutable Audit Log** means the log cannot be altered or deleted by the person being audited. Even if an attacker deletes a user, the CloudTrail record of that user's creation and deletion remains. You cannot cover your tracks in AWS.

**Forensic Investigation** is the process of examining evidence after a security incident to determine what happened, who was responsible, and what was affected. CloudTrail is the primary data source for cloud forensics.

---

## ⚠️ Reminder: How to Read Commands in This Lab

Commands are inside gray code boxes. **📋 Copy and paste** them into your terminal.

**Placeholders** look like `<THIS>` — replace them (including the `<` and `>`) with your actual values.

Here are the placeholders you will use in this lab:

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name from Lab 1A | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account ID (from `aws sts get-caller-identity`) | `123456789012` |
| `<TRAIL_BUCKET_NAME>` | A globally unique S3 bucket name for CloudTrail logs | `jane-doe-lab4b-trail-logs` |
| `<EVIDENCE_BUCKET_NAME>` | A globally unique S3 bucket name for the "evidence" bucket | `jane-doe-lab4b-evidence` |
| `<ACCESS_KEY_ID>` | The access key ID of the suspicious user (from Step 7) | `AKIA...` |

---

## Lab Steps

### Step 1: Set Your AWS Profile

**Windows (PowerShell):**

📋 Copy and paste this command, **replacing `<YOUR_PROFILE_NAME>`**:

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**macOS / Linux:**

```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**Verify it works and get your Account ID.** 📋 Copy and paste:

```
aws sts get-caller-identity
```

**✅ You should see** your account ID and role.

> **📝 Write down your Account ID** (the 12-digit number in the `"Account"` field): ______________________________

---

### Step 2: Create Your Project Folder

**Step 2a: Create the folder**

**Windows (PowerShell):**

📋 Copy and paste:

```powershell
mkdir ~\Desktop\workshop-lab-4b
cd ~\Desktop\workshop-lab-4b
```

**macOS / Linux:**

📋 Copy and paste:

```bash
mkdir ~/Desktop/workshop-lab-4b
cd ~/Desktop/workshop-lab-4b
```

> **What does this do?** Creates a new folder on your Desktop called `workshop-lab-4b` and moves your terminal into that folder.

**Step 2b: Verify you're in the right folder**

📋 Copy and paste:

```
pwd
```

**✅ You should see** a path ending in `workshop-lab-4b`.

> **💡 From now on, save ALL files you create in this lab to this folder.**

---

### Step 3: Create an S3 Bucket for CloudTrail Logs

CloudTrail needs an S3 bucket to store its log files.

📋 Copy and paste this command, **replacing `<TRAIL_BUCKET_NAME>`**:

```
aws s3 mb s3://<TRAIL_BUCKET_NAME> --region us-east-1
```

> **🔄 Example:**
> ```
> aws s3 mb s3://jane-doe-lab4b-trail-logs --region us-east-1
> ```

**✅ You should see:**

```
make_bucket: <TRAIL_BUCKET_NAME>
```

---

### Step 4: Write and Apply the CloudTrail Bucket Policy

CloudTrail needs permission to write log files to your bucket. You must add a bucket policy that allows the CloudTrail service to put objects in the bucket.

**Step 4a:** Open your text editor and create a **new, empty file**.

**Step 4b:** 📋 Copy and paste this entire block into the file, **replacing `<TRAIL_BUCKET_NAME>`** (3 places) and **`<YOUR_ACCOUNT_ID>`** (1 place):

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AWSCloudTrailAclCheck",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudtrail.amazonaws.com"
            },
            "Action": "s3:GetBucketAcl",
            "Resource": "arn:aws:s3:::<TRAIL_BUCKET_NAME>"
        },
        {
            "Sid": "AWSCloudTrailWrite",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudtrail.amazonaws.com"
            },
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::<TRAIL_BUCKET_NAME>/AWSLogs/<YOUR_ACCOUNT_ID>/*",
            "Condition": {
                "StringEquals": {
                    "s3:x-amz-acl": "bucket-owner-full-control"
                }
            }
        }
    ]
}
```

> **🔄 Example:** If your bucket is `jane-doe-lab4b-trail-logs` and your account ID is `123456789012`:
> - `"Resource": "arn:aws:s3:::jane-doe-lab4b-trail-logs"`
> - `"Resource": "arn:aws:s3:::jane-doe-lab4b-trail-logs/AWSLogs/123456789012/*"`

**Step 4c:** Save the file as `cloudtrail-bucket-policy.json` in your `workshop-lab-4b` folder on your Desktop.

> **⚠️ Common mistakes:** Make sure you replaced `<TRAIL_BUCKET_NAME>` in all 3 places and `<YOUR_ACCOUNT_ID>` in 1 place. The account ID must be exactly 12 digits with no dashes or spaces.

**Step 4d: Apply the bucket policy**

📋 Copy and paste, **replacing `<TRAIL_BUCKET_NAME>`**:

```
aws s3api put-bucket-policy --bucket <TRAIL_BUCKET_NAME> --policy file://cloudtrail-bucket-policy.json
```

**✅ No output means success.**

---

### Step 5: Create and Start the CloudTrail Trail

**Step 5a: Create the trail**

📋 Copy and paste, **replacing `<TRAIL_BUCKET_NAME>`**:

```
aws cloudtrail create-trail --name workshop-investigation-trail --s3-bucket-name <TRAIL_BUCKET_NAME> --is-multi-region-trail
```

**What does this do?**
- `create-trail` — creates a new CloudTrail trail
- `--name workshop-investigation-trail` — the name of your trail
- `--s3-bucket-name` — where to store the log files
- `--is-multi-region-trail` — records activity from ALL AWS regions

**✅ You should see** JSON output with the trail details, including `"Name": "workshop-investigation-trail"`.

**Step 5b: Start logging**

📋 Copy and paste:

```
aws cloudtrail start-logging --name workshop-investigation-trail
```

**✅ No output means success.** CloudTrail is now recording every API call in your account.

---

### Step 6: Generate Suspicious Activity

Now you will simulate actions that a suspicious user might take. Later, you will use CloudTrail to investigate what happened — like a forensic analyst would after a security incident.

> **💡 Scenario:** Imagine you are a security analyst. You have been told that a "suspicious-user" was created in the account, did some things, and then was deleted. Your job is to figure out what happened using CloudTrail.

**Step 6a: Create an "evidence" S3 bucket**

📋 Copy and paste, **replacing `<EVIDENCE_BUCKET_NAME>`**:

```
aws s3 mb s3://<EVIDENCE_BUCKET_NAME> --region us-east-1
```

> **🔄 Example:**
> ```
> aws s3 mb s3://jane-doe-lab4b-evidence --region us-east-1
> ```

**✅ You should see:** `make_bucket: <EVIDENCE_BUCKET_NAME>`

**Step 6b: Upload a file to the evidence bucket**

**Windows (PowerShell):**

📋 Copy and paste, **replacing `<EVIDENCE_BUCKET_NAME>`**:

```powershell
"CONFIDENTIAL: Employee salary data Q4 2025" | Out-File sensitive-data.txt
aws s3 cp sensitive-data.txt s3://<EVIDENCE_BUCKET_NAME>/sensitive-data.txt
```

**macOS / Linux:**

📋 Copy and paste, **replacing `<EVIDENCE_BUCKET_NAME>`**:

```bash
echo "CONFIDENTIAL: Employee salary data Q4 2025" > sensitive-data.txt
aws s3 cp sensitive-data.txt s3://<EVIDENCE_BUCKET_NAME>/sensitive-data.txt
```

**✅ You should see:** `upload: ./sensitive-data.txt to s3://<EVIDENCE_BUCKET_NAME>/sensitive-data.txt`

**Step 6c: Create a suspicious IAM user**

📋 Copy and paste:

```
aws iam create-user --user-name suspicious-user
```

**✅ You should see** JSON output with `"UserName": "suspicious-user"`.

**Step 6d: Create access keys for the suspicious user**

📋 Copy and paste:

```
aws iam create-access-key --user-name suspicious-user
```

**✅ You should see** JSON output with an AccessKeyId and SecretAccessKey.

> **📝 Write down the AccessKeyId:** ______________________________
>
> You will need this for cleanup.

**Step 6e: Delete the access keys (simulating covering tracks)**

📋 Copy and paste, **replacing `<ACCESS_KEY_ID>`** with the AccessKeyId from Step 6d:

```
aws iam delete-access-key --user-name suspicious-user --access-key-id <ACCESS_KEY_ID>
```

**✅ No output means success.**

**Step 6f: Delete the suspicious user (simulating covering tracks)**

📋 Copy and paste:

```
aws iam delete-user --user-name suspicious-user
```

**✅ No output means success.**

> **💡 What just happened?** You simulated an attacker who:
> 1. Created an S3 bucket and uploaded sensitive data (data exfiltration staging)
> 2. Created a backdoor user with access keys
> 3. Deleted the user and keys to cover their tracks
>
> The user is gone. The access keys are gone. But did they really cover their tracks? Let's find out...

---

### Step 7: Wait for CloudTrail to Record Events

> **⏱️ CloudTrail events take 5–15 minutes to appear in the Event History.**
>
> While you wait, read the **Concepts** section above if you haven't already, or review the **Cert Prep Callout** at the bottom of this lab.
>
> After 5–10 minutes, continue to Step 8.

---

### Step 8: Investigate with CloudTrail

Now put on your forensic analyst hat. The suspicious user has been deleted, but CloudTrail remembers everything.

**Step 8a: List recent events**

📋 Copy and paste:

```
aws cloudtrail lookup-events --max-results 10 --query "Events[].{Time:EventTime,Name:EventName,User:Username,Source:EventSource}" --output table
```

**✅ You should see** a table showing recent API calls in your account, including events like `CreateUser`, `CreateAccessKey`, `DeleteAccessKey`, `DeleteUser`, `CreateBucket`, `PutObject`.

> **💡 If the table is empty or shows very few events:** CloudTrail events take 5–15 minutes to appear. Wait a few more minutes and try again. This is normal behavior.

---

**Step 8b: Filter by specific event — find user creation**

📋 Copy and paste:

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateUser --query "Events[].{Time:EventTime,User:Username,Source:EventSource}" --output table
```

**✅ You should see** the `CreateUser` event — proving that someone created a user, even though that user no longer exists.

---

**Step 8c: Filter by username — find everything the suspicious user touched**

📋 Copy and paste:

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=Username,AttributeValue=suspicious-user --query "Events[].{Time:EventTime,Name:EventName,Source:EventSource}" --output table
```

> **💡 Note:** This searches for events where `suspicious-user` was the **actor** (the one making API calls). Since we created the user but never used their credentials to make calls, this may return empty results. The important thing is that the **creation and deletion** of the user is logged under YOUR username.

---

**Step 8d: Find the deletion events — prove tracks cannot be covered**

📋 Copy and paste:

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteUser --query "Events[].{Time:EventTime,User:Username,Source:EventSource}" --output table
```

**✅ You should see** the `DeleteUser` event — CloudTrail recorded that someone deleted `suspicious-user`, who did it, and when.

📋 Copy and paste:

```
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=DeleteAccessKey --query "Events[].{Time:EventTime,User:Username,Source:EventSource}" --output table
```

**✅ You should see** the `DeleteAccessKey` event — even though the key is gone, the record of its creation and deletion remains.

---

### Step 9: The Key Insight — You Cannot Cover Your Tracks

> **🔑 Critical lesson:** The suspicious user was deleted. The access keys were deleted. But CloudTrail still has a complete record of:
> - **Who** created the user (your admin identity)
> - **When** the user was created (exact timestamp)
> - **What** access keys were generated
> - **When** the keys and user were deleted
> - **From where** the actions were taken (source IP address)
>
> **In a properly configured AWS environment, you cannot cover your tracks.** This is why CloudTrail is the foundation of cloud forensics and compliance.

---

### Step 10: Console Checkpoint

Let's see the event history in the AWS Console:

1. Go to [https://console.aws.amazon.com/](https://console.aws.amazon.com/)
2. Search for **CloudTrail** in the top search bar and click it
3. Click **Event history** in the left sidebar
4. You should see a list of recent events in your account
5. Use the filter dropdown to search for:
   - **Event name:** `CreateUser` — find when the suspicious user was created
   - **Event name:** `DeleteUser` — find when it was deleted
6. Click on any event to see the full JSON details, including the source IP address and request parameters

**✅ Checkpoint:** You can see the complete audit trail in the console. Every action is recorded with who, what, when, and from where. This is exactly how security teams investigate incidents.

---

## What You Just Did

You performed a forensic investigation using CloudTrail:

1. **Set up audit logging** — created a CloudTrail trail that records all API activity
2. **Simulated suspicious activity** — created and deleted a user (like an attacker covering tracks)
3. **Investigated with CloudTrail** — used lookup-events to find exactly what happened
4. **Proved immutability** — demonstrated that deleted resources still leave a permanent audit trail

In the real world, this is how security teams investigate breaches:
- An alert fires (from GuardDuty or another tool)
- The analyst queries CloudTrail to understand the full scope of the incident
- They determine: what was accessed, what was changed, and who was responsible
- The evidence is preserved for legal proceedings or compliance reporting

---

## Cert Prep Callout

**Target Certification:** AWS Security Specialty (SCS-C02)

The Security Specialty exam tests CloudTrail extensively. You need to understand:
- How to create and configure trails (multi-region, organization trails)
- How to protect CloudTrail logs (S3 bucket policies, Object Lock, separate security account)
- How to use `lookup-events` for investigation
- The difference between management events and data events
- How CloudTrail integrates with other services (EventBridge, Athena, Security Hub)

**Sample question type:** "After a security incident, an analyst needs to determine which IAM user deleted an S3 bucket last Tuesday. Which AWS service provides this information?"  
**Answer:** AWS CloudTrail — use `lookup-events` with the `EventName=DeleteBucket` filter

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `An error occurred (InsufficientS3BucketPolicyException)` when creating the trail | The bucket policy does not grant CloudTrail permission to write | Open `cloudtrail-bucket-policy.json` and verify: (1) bucket name is correct in all 3 places, (2) account ID is correct, (3) the policy was applied with `put-bucket-policy` |
| `An error occurred (TrailAlreadyExistsException)` | The trail already exists from a previous attempt | Delete it first: `aws cloudtrail delete-trail --name workshop-investigation-trail` then try again |
| `lookup-events` returns empty results | Events take 5–15 minutes to appear | Wait 10–15 minutes and try again. This is normal CloudTrail behavior. |
| `An error occurred (EntityAlreadyExists)` when creating the user | The user already exists from a previous attempt | Delete it first: `aws iam delete-user --user-name suspicious-user` then try again |
| `An error occurred (MalformedPolicyDocument)` | JSON syntax error in the bucket policy file | Check for missing commas, brackets, or quotes. Verify all placeholders were replaced. |
| `An error occurred (NoSuchEntity)` when deleting the user | The user does not exist (already deleted or never created) | This is fine — continue to the next step |

---

## Cleanup

**⚠️ Important:** Always clean up resources after completing a lab. Follow these steps in order.

### Step 1: Stop CloudTrail Logging

📋 Copy and paste:

```
aws cloudtrail stop-logging --name workshop-investigation-trail
```

**✅ No output means success.**

### Step 2: Delete the CloudTrail Trail

📋 Copy and paste:

```
aws cloudtrail delete-trail --name workshop-investigation-trail
```

**✅ No output means success.**

### Step 3: Empty and Delete the Trail Logs Bucket

📋 Copy and paste, **replacing `<TRAIL_BUCKET_NAME>`**:

```
aws s3 rm s3://<TRAIL_BUCKET_NAME> --recursive
```

```
aws s3 rb s3://<TRAIL_BUCKET_NAME>
```

**✅ You should see** `remove_bucket: <TRAIL_BUCKET_NAME>`.

### Step 4: Empty and Delete the Evidence Bucket

📋 Copy and paste, **replacing `<EVIDENCE_BUCKET_NAME>`**:

```
aws s3 rm s3://<EVIDENCE_BUCKET_NAME> --recursive
```

```
aws s3 rb s3://<EVIDENCE_BUCKET_NAME>
```

**✅ You should see** `remove_bucket: <EVIDENCE_BUCKET_NAME>`.

### Step 5: Delete Local Files

Remove the project folder:

**Windows (PowerShell):**

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-4b
```

**macOS / Linux:**

```bash
cd ~/Desktop
rm -rf ~/Desktop/workshop-lab-4b
```

---

## Help

If you get stuck, post your question in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (copy and paste it)
3. The **full error message** you received (copy and paste it)
4. Your **operating system** (Windows, Mac, or Linux)
