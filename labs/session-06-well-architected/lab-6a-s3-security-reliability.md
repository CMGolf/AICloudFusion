# Lab 6A: Architecting S3 for Security & Reliability

**Session:** 6 — AWS Well-Architected Framework  
**Track:** Solutions Architecture  
**Difficulty:** Beginner  
**Estimated Time:** 25–30 minutes  
**Target Cert:** AWS Solutions Architect – Associate (SAA)

---

## Overview

In this lab, you will play the role of a **Solutions Architect** who has inherited a poorly configured S3 bucket. Your job is to audit it, see the problems with your own eyes, and fix them according to two pillars of the **AWS Well-Architected Framework**:

- **Security pillar** — protect data from unauthorized access
- **Reliability pillar** — protect data from accidental loss

The key idea of this lab is the **feedback loop**: for each problem, you will first SEE the bad state (data exposed to the internet, data lost forever), then apply the fix, then VERIFY the good state. You learn *why* each configuration matters by experiencing the consequence of not having it.

**The scenario:** Your team lead created an S3 bucket to store an HR salary report but forgot to secure it. You have been asked to fix it before it causes a data breach.

---

## Prerequisites

- ✅ Completed **Lab 1A** (AWS CLI installed and configured)
- ✅ AWS CLI authenticated — run `aws sts get-caller-identity` and confirm it returns your account info
- ✅ A web browser

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| Amazon S3 | Cloud storage | Free within 5 GB / 2,000 PUT / 20,000 GET per month (12-month free tier) |

**Estimated cost for this lab: $0.00**

---

## Concepts

Before we start, here is what each concept means:

**AWS Well-Architected Framework (WAF)** is a set of best practices for building good cloud systems. It is organized into six "pillars." This lab focuses on two of them:

- **Security pillar** — "Protect data, systems, and assets." This includes controlling who can access your data and encrypting it.
- **Reliability pillar** — "Ensure a workload performs its function correctly and recovers from failure." This includes protecting against accidental data loss.

**Block Public Access** is an S3 safety feature that prevents your bucket from being exposed to the internet. When OFF, a bucket policy can make your files public. When ON, S3 blocks all public access regardless of any policy.

**Bucket Policy** is a JSON document that controls who can access the files in your bucket. A policy with `"Principal": "*"` means "anyone on the internet."

**Encryption at Rest** means your data is scrambled when stored on disk, so even someone with physical access to the storage cannot read it. AWS encrypts all S3 objects by default.

**Versioning** is an S3 feature that keeps a copy of every version of a file. When ON, deleting a file does not actually remove it — it hides it behind a "delete marker," and you can recover the original. When OFF, a delete is permanent.

---

## ⚠️ Reminder: How to Read Commands in This Lab

Commands are inside gray code boxes. **📋 Copy and paste** them into your terminal.

**Placeholders** look like `<THIS>` — replace them (including the `<` and `>`) with your actual values.

In this lab, you will set your bucket name **once** as a variable, and then every command will use it automatically. This means you only choose your bucket name one time.

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name from Lab 1A | `AdministratorAccess-123456789012` |
| `<YOUR_UNIQUE_BUCKET_NAME>` | A globally unique S3 bucket name you choose | `jane-doe-waf-lab6a` |

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

**Verify it works.** 📋 Copy and paste:

```
aws sts get-caller-identity
```

**✅ You should see** your account ID and role. If you get an error about an expired token, run `aws sso login --profile <YOUR_PROFILE_NAME>` first.

---

### Step 2: Create Your Project Folder

**Step 2a: Create the folder**

**Windows (PowerShell):**

📋 Copy and paste:

```powershell
mkdir ~\Desktop\workshop-lab-6a
cd ~\Desktop\workshop-lab-6a
```

**macOS / Linux:**

📋 Copy and paste:

```bash
mkdir ~/Desktop/workshop-lab-6a
cd ~/Desktop/workshop-lab-6a
```

**Step 2b: Verify you're in the right folder**

📋 Copy and paste:

```
pwd
```

**✅ You should see** a path ending in `workshop-lab-6a`.

> **💡 From now on, save ALL files you create in this lab to this folder.**

---

### Step 3: Save Your Bucket Name as a Variable

To make this lab easier, you will save your bucket name **once**. Every command after this will use it automatically — you will not need to type the bucket name again.

**Windows (PowerShell):**

📋 Copy and paste this command, **replacing `<YOUR_UNIQUE_BUCKET_NAME>`** with a unique name (lowercase, numbers, and hyphens only):

```powershell
$BUCKET="<YOUR_UNIQUE_BUCKET_NAME>"
```

**macOS / Linux:**

📋 Copy and paste, **replacing `<YOUR_UNIQUE_BUCKET_NAME>`**:

```bash
BUCKET="<YOUR_UNIQUE_BUCKET_NAME>"
```

**Verify the variable is set.** 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
echo $BUCKET
```

**macOS / Linux:**
```bash
echo $BUCKET
```

**✅ You should see** the bucket name you chose printed back to you.

> **⚠️ Important:** If you close your terminal at any point during this lab, you must set this variable again (repeat this step). The variable only lasts for the current terminal session.

> **💡 Naming rules:** Bucket names must be 3–63 characters, globally unique, lowercase letters/numbers/hyphens only. If you get a `BucketAlreadyExists` error later, someone else has that name — pick a different one and redo this step.

---

## PART 1 — The Security Pillar

### Step 4: Create the Bucket (the "inherited" insecure bucket)

📋 Copy and paste (this uses your `$BUCKET` variable automatically):

**Windows (PowerShell):**
```powershell
aws s3 mb s3://$BUCKET --region us-east-1
```

**macOS / Linux:**
```bash
aws s3 mb s3://$BUCKET --region us-east-1
```

**✅ You should see:** `make_bucket: <your bucket name>`

---

### Step 5: Make the Bucket Public (recreating the mistake)

Your team lead disabled the safety features and added a public policy. Let's recreate that insecure state so you can see the problem.

**Step 5a: Turn off Block Public Access**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3api put-public-access-block --bucket $BUCKET --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"
```

**macOS / Linux:**
```bash
aws s3api put-public-access-block --bucket $BUCKET --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"
```

**✅ No output means success.**

**Step 5b: Create a public bucket policy file**

Open your text editor and create a **new, empty file**. 📋 Copy and paste this into the file:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicRead",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::BUCKET_NAME_HERE/*"
        }
    ]
}
```

**Now replace `BUCKET_NAME_HERE` with your actual bucket name** (the one you chose in Step 3). For example, if your bucket is `jane-doe-waf-lab6a`, the Resource line becomes:
```
"Resource": "arn:aws:s3:::jane-doe-waf-lab6a/*"
```

**Save the file as `public-policy.json`** in your `workshop-lab-6a` folder.

> **⚠️ Common mistakes:** Make sure the file extension is `.json` (not `.json.txt`). Make sure you replaced `BUCKET_NAME_HERE` with your real bucket name.

> **What does this file do?** It says "allow anyone (`*`) on the internet to download (`s3:GetObject`) any file in this bucket." This is the dangerous setting.

**Step 5c: Apply the public policy**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3api put-bucket-policy --bucket $BUCKET --policy file://public-policy.json
```

**macOS / Linux:**
```bash
aws s3api put-bucket-policy --bucket $BUCKET --policy file://public-policy.json
```

**✅ No output means success.** Your bucket is now public.

---

### Step 6: Upload the Sensitive File

**Step 6a: Create the sensitive file**

Open your text editor and create a **new, empty file**. 📋 Copy and paste this into it:

```
CONFIDENTIAL - HR Department
Employee Salary Report Q4 2025

Name            | Role              | Annual Salary
----------------|-------------------|-------------
Jane Smith      | Cloud Engineer    | $125,000
John Doe        | Security Analyst  | $115,000
Alex Johnson    | DevOps Engineer   | $130,000

This document is classified INTERNAL ONLY.
```

**Save the file as `employee-salaries.txt`** in your `workshop-lab-6a` folder.

**Step 6b: Upload it to your bucket**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3 cp employee-salaries.txt s3://$BUCKET/employee-salaries.txt
```

**macOS / Linux:**
```bash
aws s3 cp employee-salaries.txt s3://$BUCKET/employee-salaries.txt
```

**✅ You should see:** `upload: ./employee-salaries.txt to s3://<your bucket>/employee-salaries.txt`

---

### Step 7: 🚨 SEE THE PROBLEM — Your Sensitive Data Is Public

This is the feedback loop. Let's prove that anyone on the internet can read your confidential salary data.

**Step 7a: Get the public URL of your file**

📋 Copy and paste this command — it prints the exact URL of your file:

**Windows (PowerShell):**
```powershell
Write-Output "https://$BUCKET.s3.us-east-1.amazonaws.com/employee-salaries.txt"
```

**macOS / Linux:**
```bash
echo "https://$BUCKET.s3.us-east-1.amazonaws.com/employee-salaries.txt"
```

**✅ You should see** a URL printed, something like:
```
https://jane-doe-waf-lab6a.s3.us-east-1.amazonaws.com/employee-salaries.txt
```

**Step 7b: Open the URL in your browser**

1. **Copy the URL** that was printed in Step 7a
2. **Open a new browser tab**
3. **Paste the URL** into the address bar and press Enter

**🚨 You should see the full salary report displayed in your browser.** Anyone in the world with this URL can read your confidential HR data. This is a **major security violation** — it breaks the Security pillar of the Well-Architected Framework.

**Step 7c: Confirm from the command line too**

You can also test access from the terminal. 📋 Copy and paste:

**Windows (PowerShell):**
```powershell
curl.exe -s -o NUL -w "HTTP Status: %{http_code}" "https://$BUCKET.s3.us-east-1.amazonaws.com/employee-salaries.txt"
```

**macOS / Linux:**
```bash
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" "https://$BUCKET.s3.us-east-1.amazonaws.com/employee-salaries.txt"
```

**✅ You should see:** `HTTP Status: 200` — meaning the download succeeded (the data is publicly accessible).

> **💡 Why this matters:** A status code of `200` means "OK, here is the file." For a confidential file, this is exactly what you do NOT want. You want `403` (Access Denied).

---

### Step 8: ✅ FIX THE PROBLEM — Secure the Bucket

Now apply the Security pillar fix: remove the public policy and turn Block Public Access back on.

**Step 8a: Delete the public bucket policy**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3api delete-bucket-policy --bucket $BUCKET
```

**macOS / Linux:**
```bash
aws s3api delete-bucket-policy --bucket $BUCKET
```

**✅ No output means success.**

**Step 8b: Turn Block Public Access back ON**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3api put-public-access-block --bucket $BUCKET --public-access-block-configuration "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

**macOS / Linux:**
```bash
aws s3api put-public-access-block --bucket $BUCKET --public-access-block-configuration "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

**✅ No output means success.** The bucket is now private.

---

### Step 9: ✅ VERIFY THE FIX — Data Is No Longer Public

**Step 9a: Try the URL in your browser again**

1. Go back to the browser tab with your file URL (or paste the URL again)
2. **Refresh the page** (press F5, or Ctrl+R / Cmd+R)

**✅ You should now see an error** — something like "Access Denied" in XML format. The confidential data is no longer visible to the public.

> **💡 You may need to wait 5–10 seconds** and refresh again — it can take a moment for the change to take effect.

**Step 9b: Confirm from the command line**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
curl.exe -s -o NUL -w "HTTP Status: %{http_code}" "https://$BUCKET.s3.us-east-1.amazonaws.com/employee-salaries.txt"
```

**macOS / Linux:**
```bash
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" "https://$BUCKET.s3.us-east-1.amazonaws.com/employee-salaries.txt"
```

**✅ You should see:** `HTTP Status: 403` — meaning Access Denied. The data is now protected.

> **🎉 Security pillar achieved!** You went from `200` (anyone can read) to `403` (access denied). The same file is still there for authorized users, but the public can no longer reach it.

**Step 9c: A note about encryption**

📋 Copy and paste — let's check whether the file is encrypted:

**Windows (PowerShell):**
```powershell
aws s3api head-object --bucket $BUCKET --key employee-salaries.txt --query "ServerSideEncryption" --output text
```

**macOS / Linux:**
```bash
aws s3api head-object --bucket $BUCKET --key employee-salaries.txt --query "ServerSideEncryption" --output text
```

**✅ You should see:** `AES256`

> **💡 Key insight:** AWS automatically encrypts every S3 object at rest (since 2023). Your data was ALWAYS encrypted. But notice — that encryption did NOT stop the public from downloading it in Step 7! This teaches an important Security pillar lesson: **encryption protects data on disk, but access control (who can download) is a separate, equally important layer.** You need both.

---

## PART 2 — The Reliability Pillar

Now you will see why **versioning** matters for the Reliability pillar. First you will experience permanent data loss (no versioning), then you will see how versioning protects you.

### Step 10: 🚨 SEE THE PROBLEM — Delete Without Versioning

Right now your bucket has NO versioning. Let's see what happens when a file is deleted.

**Step 10a: Confirm the file is there**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3 ls s3://$BUCKET/
```

**macOS / Linux:**
```bash
aws s3 ls s3://$BUCKET/
```

**✅ You should see** `employee-salaries.txt` listed.

**Step 10b: Delete the file**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3 rm s3://$BUCKET/employee-salaries.txt
```

**macOS / Linux:**
```bash
aws s3 rm s3://$BUCKET/employee-salaries.txt
```

**✅ You should see:** `delete: s3://<your bucket>/employee-salaries.txt`

**Step 10c: Confirm it is gone**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3 ls s3://$BUCKET/
```

**macOS / Linux:**
```bash
aws s3 ls s3://$BUCKET/
```

**✅ You should see** nothing — no output at all. The bucket is empty.

**Step 10d: Confirm in the Console**

1. Go to [https://console.aws.amazon.com/](https://console.aws.amazon.com/)
2. Search for **S3** and click it
3. Click on your bucket name
4. **The bucket is empty.** The file is gone.

> **🚨 The data is gone forever.** There is no way to recover it. If that salary report was the only copy, it is lost permanently. This breaks the Reliability pillar — your system did not protect against accidental data loss.

---

### Step 11: ✅ FIX THE PROBLEM — Enable Versioning

**Step 11a: Enable versioning on the bucket**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3api put-bucket-versioning --bucket $BUCKET --versioning-configuration Status=Enabled
```

**macOS / Linux:**
```bash
aws s3api put-bucket-versioning --bucket $BUCKET --versioning-configuration Status=Enabled
```

**✅ No output means success.**

**Step 11b: Verify versioning is on**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3api get-bucket-versioning --bucket $BUCKET --query "Status" --output text
```

**macOS / Linux:**
```bash
aws s3api get-bucket-versioning --bucket $BUCKET --query "Status" --output text
```

**✅ You should see:** `Enabled`

**Step 11c: Re-upload the file (since it was lost)**

Re-create the file first (it was deleted from the bucket, but you still have your local copy in the project folder).

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3 cp employee-salaries.txt s3://$BUCKET/employee-salaries.txt
```

**macOS / Linux:**
```bash
aws s3 cp employee-salaries.txt s3://$BUCKET/employee-salaries.txt
```

**✅ You should see** the upload confirmation.

---

### Step 12: ✅ VERIFY THE FIX — Delete and Recover

Now let's prove that with versioning ON, a deleted file can be recovered.

**Step 12a: Delete the file (again)**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3 rm s3://$BUCKET/employee-salaries.txt
```

**macOS / Linux:**
```bash
aws s3 rm s3://$BUCKET/employee-salaries.txt
```

**✅ You should see** the delete confirmation.

**Step 12b: Confirm the normal listing shows it gone**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3 ls s3://$BUCKET/
```

**macOS / Linux:**
```bash
aws s3 ls s3://$BUCKET/
```

**✅ You should see** nothing — the file appears gone, just like before.

**Step 12c: But look at the VERSIONS — the file is still there!**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3api list-object-versions --bucket $BUCKET --query "Versions[].{Key:Key,VersionId:VersionId}" --output table
```

**macOS / Linux:**
```bash
aws s3api list-object-versions --bucket $BUCKET --query "Versions[].{Key:Key,VersionId:VersionId}" --output table
```

**✅ You should see** a table with `employee-salaries.txt` and a `VersionId` (a long string of letters and numbers).

> **📝 Copy the VersionId** from the table — you will need it in the next step. It looks something like `BWxsoyftUEzIL_d8dzJ28l.h89dXsV4d`.

**Step 12d: Confirm in the Console**

1. Go to your bucket in the S3 Console
2. The bucket looks empty
3. **Toggle the "Show versions" switch** at the top of the object list (turn it ON)
4. **Now you can see the file and its versions**, including a "delete marker"

> **💡 What is a delete marker?** When versioning is on, deleting a file just adds a "delete marker" on top — it hides the file but keeps the actual data underneath. That is why you can recover it.

**Step 12e: Recover the file**

📋 Copy and paste this command, **replacing `<VERSION_ID>`** with the VersionId you copied in Step 12c:

**Windows (PowerShell):**
```powershell
aws s3api copy-object --bucket $BUCKET --copy-source "$BUCKET/employee-salaries.txt?versionId=<VERSION_ID>" --key employee-salaries.txt
```

**macOS / Linux:**
```bash
aws s3api copy-object --bucket $BUCKET --copy-source "$BUCKET/employee-salaries.txt?versionId=<VERSION_ID>" --key employee-salaries.txt
```

**✅ You should see** JSON output with a `CopyObjectResult` and a `LastModified` timestamp.

**Step 12f: Confirm the file is back**

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3 ls s3://$BUCKET/
```

**macOS / Linux:**
```bash
aws s3 ls s3://$BUCKET/
```

**✅ You should see** `employee-salaries.txt` listed again. **You recovered the "lost" file!**

> **🎉 Reliability pillar achieved!** Without versioning, the delete in Step 10 was permanent. With versioning, the delete in Step 12 was recoverable. This is how real systems protect against accidental deletion, ransomware, and human error.

---

## What You Just Did

You acted as a Solutions Architect and applied two Well-Architected Framework pillars to a real S3 bucket:

| Pillar | Problem You Saw | Fix You Applied | How You Verified |
|--------|----------------|-----------------|------------------|
| **Security** | Confidential data readable by anyone on the internet (HTTP 200) | Removed public policy + enabled Block Public Access | URL returned 403, data protected |
| **Reliability** | Deleted file gone forever | Enabled versioning | Deleted file recovered from version history |

You also learned a critical lesson: **encryption alone is not security.** Your data was encrypted the whole time, but it was still publicly downloadable. Access control and data recovery are separate, essential layers.

---

## Cert Prep Callout

**Target Certification:** AWS Solutions Architect – Associate (SAA)

The SAA exam tests these concepts heavily:
- S3 Block Public Access and bucket policies (Security)
- S3 versioning and how delete markers work (Reliability/Durability)
- The difference between encryption at rest and access control
- The Well-Architected Framework pillars and how to apply them

**Sample question type:** "A company stored confidential data in an S3 bucket. Despite the data being encrypted, it was leaked publicly. What was the most likely cause?"  
**Answer:** A bucket policy or ACL allowed public access — encryption protects data at rest but does not control who can access it.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `$BUCKET` commands fail with "bucket name required" | The variable was lost (you closed the terminal) | Repeat Step 3 to set the `$BUCKET` variable again |
| `BucketAlreadyExists` | Someone else has that bucket name | Pick a new name and redo Step 3, then Step 4 |
| Browser still shows the file after blocking | The change takes a few seconds; browser may have cached it | Wait 10 seconds, then hard-refresh (Ctrl+Shift+R / Cmd+Shift+R) |
| `curl` shows 200 after you blocked access | Same caching/propagation delay | Wait 10 seconds and run the curl command again |
| `MalformedPolicy` when applying the policy | The bucket name in `public-policy.json` is wrong | Open the file and confirm you replaced `BUCKET_NAME_HERE` with your real bucket name |
| `list-object-versions` shows nothing | Versioning was not enabled before the delete | Versioning only protects deletes that happen AFTER it is enabled. Make sure you did Step 11 before Step 12. |

---

## Cleanup

**⚠️ Important:** A versioned bucket cannot be deleted until ALL versions and delete markers are removed. Follow these steps carefully.

### Step 1: List every version and delete marker in the bucket

A versioned bucket keeps hidden versions and delete markers. You must remove all of them before the bucket can be deleted.

📋 Copy and paste — this shows everything still in the bucket:

**Windows (PowerShell):**
```powershell
aws s3api list-object-versions --bucket $BUCKET --query "[Versions[].{Key:Key,VersionId:VersionId},DeleteMarkers[].{Key:Key,VersionId:VersionId}]" --output table
```

**macOS / Linux:**
```bash
aws s3api list-object-versions --bucket $BUCKET --query "[Versions[].{Key:Key,VersionId:VersionId},DeleteMarkers[].{Key:Key,VersionId:VersionId}]" --output table
```

**✅ You should see** a table listing one or more `VersionId` values for `employee-salaries.txt`. These are the hidden versions and delete markers.

> **📝 Copy each VersionId** shown in the table. You will delete them one at a time in the next step.

### Step 2: Delete each version

For **each** `VersionId` from the table above, run this command, **replacing `<VERSION_ID>`** with one of the IDs. Repeat once per version/marker shown.

📋 Copy and paste (run once per VersionId):

**Windows (PowerShell):**
```powershell
aws s3api delete-object --bucket $BUCKET --key employee-salaries.txt --version-id <VERSION_ID>
```

**macOS / Linux:**
```bash
aws s3api delete-object --bucket $BUCKET --key employee-salaries.txt --version-id <VERSION_ID>
```

**✅ Each command** returns a small JSON confirmation.

> **💡 Tip (macOS / Linux only):** You can delete everything in one command instead:
> ```bash
> aws s3api delete-objects --bucket $BUCKET --delete "$(aws s3api list-object-versions --bucket $BUCKET --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}' --output json)"
> aws s3api delete-objects --bucket $BUCKET --delete "$(aws s3api list-object-versions --bucket $BUCKET --query '{Objects: DeleteMarkers[].{Key:Key,VersionId:VersionId}}' --output json)"
> ```

### Step 3: Verify the bucket is empty

📋 Copy and paste:

```
aws s3api list-object-versions --bucket $BUCKET --query "[Versions,DeleteMarkers]" --output json
```

**✅ You should see** `[null,null]` or empty lists. If you still see versions, repeat Step 2 for the remaining IDs.

### Step 4: Delete the bucket

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
aws s3 rb s3://$BUCKET
```

**macOS / Linux:**
```bash
aws s3 rb s3://$BUCKET
```

**✅ You should see:** `remove_bucket: <your bucket>`

### Step 5: Delete local files

**Windows (PowerShell):**
```powershell
cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-6a
```

**macOS / Linux:**
```bash
cd ~/Desktop
rm -rf ~/Desktop/workshop-lab-6a
```

---

## Help

If you get stuck, post your question in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (copy and paste it)
3. The **full error message** you received (copy and paste it)
4. Your **operating system** (Windows, Mac, or Linux)
