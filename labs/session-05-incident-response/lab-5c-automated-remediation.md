# Lab 5C: Build Automated Incident Response with Lambda

**Session:** 5 — Incident Response on AWS  
**Track:** Cloud Security & IR  
**Difficulty:** Advanced  
**Estimated Time:** 45–55 minutes  
**Target Cert:** AWS Security Specialty

---

## Overview

In this lab, you will:

1. **Create a Lambda function** that automatically deactivates newly created access keys
2. **Test the function manually** — prove it works by invoking it with a simulated event
3. **Wire up EventBridge** — create a rule that triggers the Lambda whenever a new access key is created
4. **Understand automated remediation** — how companies enforce "no long-lived credentials" policies

By the end of this lab, you will have built a real automated incident response system. This is **SOAR** (Security Orchestration, Automation, and Response) — the same pattern used by enterprise security teams to respond to threats in seconds instead of hours.

---

## Prerequisites

- ✅ Completed **Lab 1A** (AWS CLI installed and configured)
- ✅ Completed **Lab 5A** (understand credential revocation)
- ✅ AWS CLI authenticated — run `aws sts get-caller-identity` and confirm it returns your account info
- ✅ A text editor to create files (VS Code, Notepad, or any editor)

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| IAM | Identity and Access Management | Always Free |
| AWS Lambda | Serverless compute | Free: 1M requests + 400,000 GB-seconds per month |
| Amazon EventBridge | Event routing | Free for AWS service events |

**Estimated cost for this lab: $0.00**

---

## Concepts

Before we start, here is what each concept means:

**SOAR (Security Orchestration, Automation, and Response)** is the practice of automating security operations. Instead of a human manually deactivating compromised credentials, a system detects the threat and responds automatically — in seconds, not hours.

**Automated Remediation** means fixing security issues without human intervention. Examples:
- Automatically deactivating access keys when they are created (enforce SSO-only)
- Automatically isolating EC2 instances when GuardDuty detects compromise
- Automatically revoking overly permissive security group rules

**EventBridge** is AWS's event bus. It routes events from AWS services (like CloudTrail) to targets (like Lambda functions). When something happens in your account, EventBridge can trigger an automated response.

**The Pattern:** CloudTrail records an API call → EventBridge matches the event against rules → Lambda executes the automated response. This pipeline runs 24/7 without human intervention.

**Why This Matters:** Manual incident response takes 30–60 minutes at best (human gets alert, reads it, logs in, takes action). Automated response takes 1–5 minutes (event → Lambda → remediation). For credential theft, those 30 minutes could mean the difference between a contained incident and a full breach.

**CloudTrail → EventBridge Delay:** CloudTrail events are delivered to EventBridge within 5–15 minutes of the API call. This is not instant, but it is much faster than waiting for a human to notice and respond.

---

## ⚠️ Reminder: How to Read Commands in This Lab

Commands are inside gray code boxes. **📋 Copy and paste** them into your terminal.

**Placeholders** look like `<THIS>` — replace them (including the `<` and `>`) with your actual values.

Here are the placeholders you will use in this lab:

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name from Lab 1A | `AdministratorAccess-123456789012` |
| `<ACCOUNT_ID>` | Your 12-digit AWS account ID | `123456789012` |
| `<LAMBDA_ARN>` | The ARN of your Lambda function (Step 6) | `arn:aws:lambda:us-east-1:123456789012:function:workshop-key-revoker` |
| `<RULE_ARN>` | The ARN of your EventBridge rule (Step 9) | `arn:aws:events:us-east-1:123456789012:rule/workshop-auto-revoke` |
| `<TEST_KEY_ID>` | The Access Key ID of the test user (Step 7) | `AKIAIOSFODNN7EXAMPLE` |

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

> **📝 Note your Account ID** (the 12-digit number): ______________________________

---

### Step 2: Create Your Project Folder

**Step 2a: Create the folder**

**Windows (PowerShell):**

📋 Copy and paste:

```powershell
mkdir ~\Desktop\workshop-lab-5c
cd ~\Desktop\workshop-lab-5c
```

**macOS / Linux:**

📋 Copy and paste:

```bash
mkdir ~/Desktop/workshop-lab-5c
cd ~/Desktop/workshop-lab-5c
```

> **What does this do?** Creates a new folder on your Desktop called `workshop-lab-5c` and moves your terminal into that folder.

**Step 2b: Verify you're in the right folder**

📋 Copy and paste:

```
pwd
```

**✅ You should see** a path ending in `workshop-lab-5c` (e.g., `C:\Users\YourName\Desktop\workshop-lab-5c` on Windows or `/Users/YourName/Desktop/workshop-lab-5c` on Mac).

> **💡 From now on, save ALL files you create in this lab to this folder.** When the lab says "save the file," save it here.

---

### Step 3: Create the Lambda IAM Role

The Lambda function needs permission to deactivate access keys. You will create an IAM role that grants this.

**Step 3a: Create the trust policy**

Open your text editor and create a **new, empty file**.

📋 Copy and paste this entire block into the file:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "lambda.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

Save the file as `trust-policy.json` in your `workshop-lab-5c` folder.

> **What does this do?** This trust policy allows the Lambda service to assume this role. Without it, Lambda cannot use the role's permissions.

**Step 3b: Create the IAM role**

📋 Copy and paste:

```
aws iam create-role --role-name workshop-key-revoker-role --assume-role-policy-document file://trust-policy.json
```

**✅ You should see** JSON output with the role details, including an ARN.

**Step 3c: Attach the IAMFullAccess policy**

📋 Copy and paste:

```
aws iam attach-role-policy --role-name workshop-key-revoker-role --policy-arn arn:aws:iam::aws:policy/IAMFullAccess
```

**✅ No output means success.**

> **⚠️ In production**, you would use a least-privilege policy that only allows `iam:UpdateAccessKey`. We use `IAMFullAccess` here for simplicity in a lab environment.

**Step 3d: Attach the Lambda basic execution policy**

📋 Copy and paste:

```
aws iam attach-role-policy --role-name workshop-key-revoker-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

**✅ No output means success.**

> **What does this do?** `AWSLambdaBasicExecutionRole` allows the Lambda function to write logs to CloudWatch. Without this, you cannot see the function's output or debug errors.

---

### Step 4: Write the Lambda Function

**Step 4a:** Open your text editor and create a **new, empty file**.

**Step 4b:** 📋 Copy and paste this entire Python function into the file:

```python
import json
import boto3

iam_client = boto3.client('iam')

def lambda_handler(event, context):
    # Extract the username and access key ID from the CloudTrail event
    detail = event.get('detail', {})
    request_params = detail.get('requestParameters', {})
    response_elements = detail.get('responseElements', {})
    
    username = request_params.get('userName', 'unknown')
    
    # The access key ID is in the response elements
    access_key_info = response_elements.get('accessKey', {})
    access_key_id = access_key_info.get('accessKeyId', '')
    
    if not access_key_id:
        print(f"No access key ID found in event for user {username}")
        return {'statusCode': 400, 'body': 'No access key ID found'}
    
    # Deactivate the access key
    try:
        iam_client.update_access_key(
            UserName=username,
            AccessKeyId=access_key_id,
            Status='Inactive'
        )
        message = f"SECURITY ALERT: Deactivated access key {access_key_id} for user {username}"
        print(message)
        return {'statusCode': 200, 'body': message}
    except Exception as e:
        error_msg = f"Failed to deactivate key {access_key_id} for user {username}: {str(e)}"
        print(error_msg)
        return {'statusCode': 500, 'body': error_msg}
```

**Step 4c:** Save the file as `revoke_key.py` in your `workshop-lab-5c` folder.

> **What does this function do?**
> 1. Receives a CloudTrail event (via EventBridge) that says "a new access key was created"
> 2. Extracts the username and access key ID from the event
> 3. Immediately deactivates the access key
> 4. Logs a security alert message
>
> This enforces a "no long-lived credentials" policy — any new access key is automatically deactivated within minutes of creation.

---

### Step 5: Package and Deploy the Lambda Function

**Step 5a: Zip the function code**

**Windows (PowerShell):**

📋 Copy and paste:

```powershell
Compress-Archive -Path revoke_key.py -DestinationPath revoke_key.zip -Force
```

**macOS / Linux:**

📋 Copy and paste:

```bash
zip revoke_key.zip revoke_key.py
```

**Step 5b: Deploy the Lambda function**

📋 Copy and paste, **replacing `<ACCOUNT_ID>`** with your 12-digit AWS account ID:

```
aws lambda create-function --function-name workshop-key-revoker --runtime python3.12 --role arn:aws:iam::<ACCOUNT_ID>:role/workshop-key-revoker-role --handler revoke_key.lambda_handler --zip-file fileb://revoke_key.zip --region us-east-1 --timeout 30
```

**✅ You should see** JSON output with the function details, including a `FunctionArn`.

> **📝 Write down the FunctionArn:** ______________________________
>
> It looks like: `arn:aws:lambda:us-east-1:<ACCOUNT_ID>:function:workshop-key-revoker`

> **💡 If you get an error** about the role not being found or "The role defined for the function cannot be assumed by Lambda": Wait 10 seconds and try again. IAM role propagation can take a few seconds.

---

### Step 6: Test the Lambda Function (Manual Invoke)

Before wiring up EventBridge, let's prove the Lambda function works by invoking it manually with a simulated event.

**Step 6a: Create a test IAM user**

📋 Copy and paste:

```
aws iam create-user --user-name auto-revoke-test
```

**Step 6b: Create an access key for the test user**

📋 Copy and paste:

```
aws iam create-access-key --user-name auto-revoke-test
```

**✅ You should see** JSON output with `AccessKeyId` and `SecretAccessKey`.

> **📝 Write down the AccessKeyId:** ______________________________

**Step 6c: Verify the key is Active**

📋 Copy and paste:

```
aws iam list-access-keys --user-name auto-revoke-test --query "AccessKeyMetadata[].{Id:AccessKeyId,Status:Status}" --output table
```

**✅ You should see** the key with Status `Active`.

**Step 6d: Create the test event**

Open your text editor and create a **new, empty file**.

📋 Copy and paste this entire block into the file, **replacing `<TEST_KEY_ID>`** with the AccessKeyId from Step 6b:

```json
{
    "detail": {
        "eventName": "CreateAccessKey",
        "requestParameters": {
            "userName": "auto-revoke-test"
        },
        "responseElements": {
            "accessKey": {
                "accessKeyId": "<TEST_KEY_ID>",
                "status": "Active",
                "userName": "auto-revoke-test"
            }
        }
    }
}
```

Save the file as `test-event.json` in your `workshop-lab-5c` folder.

> **What does this do?** This simulates the CloudTrail event that EventBridge would send when a new access key is created. It contains the username and key ID that the Lambda function needs.

**Step 6e: Invoke the Lambda function**

📋 Copy and paste:

```
aws lambda invoke --function-name workshop-key-revoker --payload file://test-event.json --cli-binary-format raw-in-base64-out response.json --region us-east-1
```

**✅ You should see:**

```json
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```

**Step 6f: Check the response**

**Windows (PowerShell):**

📋 Copy and paste:

```powershell
Get-Content response.json
```

**macOS / Linux:**

📋 Copy and paste:

```bash
cat response.json
```

**✅ You should see:**

```json
{"statusCode": 200, "body": "SECURITY ALERT: Deactivated access key <TEST_KEY_ID> for user auto-revoke-test"}
```

**Step 6g: Verify the key is now Inactive**

📋 Copy and paste:

```
aws iam list-access-keys --user-name auto-revoke-test --query "AccessKeyMetadata[].{Id:AccessKeyId,Status:Status}" --output table
```

**✅ You should see** the key with Status `Inactive`.

> **🎉 The Lambda function works!** It successfully received the event, extracted the key information, and deactivated the access key. This proves your automated remediation logic is correct.

---

### Step 7: Wire Up EventBridge (Automation Layer)

Now connect EventBridge so the Lambda function runs automatically whenever a new access key is created.

**Step 7a: Create the event pattern**

Open your text editor and create a **new, empty file**.

📋 Copy and paste this entire block into the file:

```json
{
    "source": ["aws.iam"],
    "detail-type": ["AWS API Call via CloudTrail"],
    "detail": {
        "eventSource": ["iam.amazonaws.com"],
        "eventName": ["CreateAccessKey"]
    }
}
```

Save the file as `eventbridge-pattern.json` in your `workshop-lab-5c` folder.

> **What does this do?** This event pattern tells EventBridge: "Match any CloudTrail event where someone calls the `CreateAccessKey` API on IAM." When this pattern matches, EventBridge will trigger the Lambda function.

**Step 7b: Create the EventBridge rule**

📋 Copy and paste:

```
aws events put-rule --name workshop-auto-revoke --event-pattern file://eventbridge-pattern.json --state ENABLED --region us-east-1
```

**✅ You should see** JSON output with a `RuleArn`:

```json
{
    "RuleArn": "arn:aws:events:us-east-1:<ACCOUNT_ID>:rule/workshop-auto-revoke"
}
```

> **📝 Write down the RuleArn:** ______________________________

**Step 7c: Add Lambda as the target**

📋 Copy and paste, **replacing `<LAMBDA_ARN>`** with the FunctionArn from Step 5b:

```
aws events put-targets --rule workshop-auto-revoke --targets "Id=lambda-target,Arn=<LAMBDA_ARN>" --region us-east-1
```

**✅ You should see:**

```json
{
    "FailedEntryCount": 0,
    "FailedEntries": []
}
```

**Step 7d: Grant EventBridge permission to invoke Lambda**

📋 Copy and paste, **replacing `<RULE_ARN>`** with the RuleArn from Step 7b:

```
aws lambda add-permission --function-name workshop-key-revoker --statement-id eventbridge-invoke --action lambda:InvokeFunction --principal events.amazonaws.com --source-arn <RULE_ARN> --region us-east-1
```

**✅ You should see** JSON output with a `Statement` field.

> **What does this do?** Grants the EventBridge service permission to invoke your Lambda function. Without this, EventBridge would match the event but fail to trigger the function.

---

### Step 8: Understanding the Automation

> **🎉 Congratulations!** You have built a complete automated incident response system:
>
> ```
> Someone creates an access key
>        ↓ (5-15 minutes)
> CloudTrail records the event
>        ↓
> EventBridge matches the event pattern
>        ↓
> Lambda function is triggered
>        ↓
> Access key is automatically deactivated
> ```
>
> **In production, this means:**
> - No human needs to be awake or available
> - Response happens in minutes, not hours
> - Every new access key is automatically deactivated
> - Users are forced to use SSO/roles instead of long-lived keys
>
> **About the 5–15 minute delay:** CloudTrail events are delivered to EventBridge within 5–15 minutes. The manual invoke in Step 6 proved the Lambda works instantly. The EventBridge wiring ensures it runs automatically — you just need to account for the CloudTrail delivery delay.
>
> **⚠️ Important:** While this EventBridge rule is active, any new access key created in your account will be automatically deactivated within minutes. This is why cleanup order matters — disable the rule BEFORE creating any new access keys.

---

### Step 9: Console Checkpoint

Let's verify what you built in the AWS Console:

1. Go to [https://console.aws.amazon.com/](https://console.aws.amazon.com/)
2. Search for **Lambda** in the top search bar and click it
3. Make sure you are in the **us-east-1** region
4. Click **Functions** — you should see `workshop-key-revoker`
5. Click on the function — you can see the code, configuration, and triggers
6. Now search for **EventBridge** in the top search bar and click it
7. Click **Rules** in the left sidebar
8. Select the **default** event bus
9. You should see `workshop-auto-revoke` rule with status **Enabled**
10. Click on the rule — you can see the event pattern and the Lambda target

**✅ Checkpoint:** You can see both the Lambda function and the EventBridge rule in the console. The rule is enabled and targeting your Lambda function. This is a production-ready automated remediation pipeline.

---

## What You Just Did

You built an automated incident response system (SOAR):

1. **Created a Lambda function** that deactivates access keys on demand
2. **Tested it manually** — proved the logic works with a simulated event
3. **Wired up EventBridge** — connected CloudTrail events to automatic Lambda execution
4. **Understood the pipeline** — CloudTrail → EventBridge → Lambda → Remediation

This is the same pattern used by companies like Netflix, Capital One, and Airbnb to enforce security policies at scale. Instead of relying on humans to catch and respond to every security event, they automate the response.

**Real-world applications of this pattern:**
- Automatically deactivate access keys (enforce SSO-only)
- Automatically isolate EC2 instances when GuardDuty detects compromise
- Automatically revoke overly permissive security group rules
- Automatically send alerts to PagerDuty/Slack when high-severity findings appear
- Automatically create forensic snapshots before terminating compromised instances

---

## Cert Prep Callout

**Target Certification:** AWS Security Specialty (SCS-C02)

The Security Specialty exam tests automated remediation heavily. You need to understand:
- The EventBridge + Lambda pattern for automated response
- How CloudTrail events flow to EventBridge
- Event patterns and how to match specific API calls
- The difference between manual and automated incident response
- SOAR concepts and when to apply them

**Sample question type:** "A company wants to automatically disable any newly created IAM access keys to enforce their SSO-only policy. Which combination of services achieves this?"  
**Answer:** CloudTrail + EventBridge + Lambda (CloudTrail records the CreateAccessKey event, EventBridge matches it, Lambda deactivates the key)

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `The role defined for the function cannot be assumed by Lambda` | IAM role has not propagated yet | Wait 10 seconds and try the `create-function` command again |
| Lambda invoke returns `statusCode: 400` | The test event JSON is malformed or missing the key ID | Open `test-event.json` and verify you replaced `<TEST_KEY_ID>` with the actual key ID |
| Lambda invoke returns `statusCode: 500` | The Lambda role does not have IAM permissions | Verify you attached `IAMFullAccess` to the role in Step 3c |
| `put-targets` returns `FailedEntryCount: 1` | The Lambda ARN is wrong | Double-check the ARN from Step 5b. It should start with `arn:aws:lambda:us-east-1:` |
| `add-permission` returns an error about existing statement | You already ran this command | This is fine — the permission already exists |
| EventBridge rule does not trigger Lambda | CloudTrail → EventBridge takes 5–15 minutes | This is expected. The manual invoke in Step 6 proved the Lambda works. |

---

## Cleanup

**⚠️ IMPORTANT: Follow this cleanup order exactly.** You must disable the EventBridge rule BEFORE deleting other resources. If you create access keys while the rule is active, the Lambda will deactivate them.

### Step 1: Remove EventBridge Targets

📋 Copy and paste:

```
aws events remove-targets --rule workshop-auto-revoke --ids lambda-target --region us-east-1
```

**✅ You should see** `"FailedEntryCount": 0`.

### Step 2: Delete the EventBridge Rule

📋 Copy and paste:

```
aws events delete-rule --name workshop-auto-revoke --region us-east-1
```

**✅ No output means success.**

### Step 3: Delete the Lambda Function

📋 Copy and paste:

```
aws lambda delete-function --function-name workshop-key-revoker --region us-east-1
```

**✅ You should see JSON output with a `"StatusCode": 204` field.** which means the delete-function request succeeded.

### Step 4: Detach Policies and Delete the IAM Role

📋 Copy and paste each command:

```
aws iam detach-role-policy --role-name workshop-key-revoker-role --policy-arn arn:aws:iam::aws:policy/IAMFullAccess
```

```
aws iam detach-role-policy --role-name workshop-key-revoker-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

```
aws iam delete-role --role-name workshop-key-revoker-role
```

**✅ No output for each command means success.**

### Step 5: Delete the Test User and Access Key

📋 Copy and paste, **replacing `<TEST_KEY_ID>`** with the key ID from Step 6b:

```
aws iam delete-access-key --user-name auto-revoke-test --access-key-id <TEST_KEY_ID>
```

```
aws iam delete-user --user-name auto-revoke-test
```

**✅ No output for each command means success.**

### Step 6: Delete Local Files

Remove the project folder:

**Windows (PowerShell):**

```powershell
cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-lab-5c
```

**macOS / Linux:**

```bash
cd ~/Desktop
rm -rf ~/Desktop/workshop-lab-5c
```

---

## Help

If you get stuck, post your question in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (copy and paste it)
3. The **full error message** you received (copy and paste it)
4. Your **operating system** (Windows, Mac, or Linux)
