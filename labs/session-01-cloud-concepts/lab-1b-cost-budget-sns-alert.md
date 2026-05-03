# Lab 1B: Create a Cost Budget Alert with Email Notification

**Session:** 1 — Cloud Concepts & AWS Global Infrastructure  
**Track:** Cloud Basics  
**Difficulty:** Intermediate  
**Estimated Time:** 25–30 minutes

---

## Overview

In this lab, you will create a **monthly cost budget** in AWS and connect it to an **email notification** so you get alerted if your account starts spending money. This is one of the very first things a cloud engineer sets up on any new AWS account — cost visibility is critical because cloud services can incur charges if left running.

By the end of this lab, you will have:
- An SNS topic (a notification channel) that can send emails
- A budget that monitors your AWS spending
- An automatic email alert that fires if spending approaches your limit

---

## Prerequisites

- ✅ Completed **Lab 1A** (AWS CLI installed and configured)
- ✅ AWS CLI authenticated — run `aws sts get-caller-identity` and confirm it returns your account info
- ✅ Your **AWS account ID** (the 12-digit number from Lab 1A, Step 6)
- ✅ An **email address** you can check during this lab (you will need to click a confirmation link)

---

## Cost Notice

| Service | What It Is | Free Tier Limit |
|---------|-----------|----------------|
| AWS Budgets | Tracks your spending and sends alerts | 2 budgets free per account |
| Amazon SNS | Sends notifications (email, SMS, etc.) | 1,000 email notifications/month free |

**Estimated cost for this lab: $0.00**

---

## Concepts

Before we start, here is what each service does and why we are using it:

**AWS Budgets** is a service that lets you set a spending limit on your AWS account. You tell it "alert me if I spend more than X dollars this month," and it watches your spending for you. Think of it like setting a spending alert on your bank account — you set a threshold, and the bank texts you when you get close.

**Amazon SNS (Simple Notification Service)** is a messaging service. Here is how it works:
1. You create a **"topic"** — think of this as a mailing list
2. You **subscribe** an email address to the topic — this is like signing up for the mailing list
3. Other AWS services (like Budgets) can **send messages** to the topic
4. Everyone subscribed to the topic receives the message

In this lab, we will create an SNS topic, subscribe your email to it, and then tell AWS Budgets to send alerts to that topic whenever your spending gets close to your budget limit.

---

## ⚠️ Reminder: How to Read Commands in This Lab

Commands are inside gray code boxes. **📋 Copy and paste** them into your terminal — do not type them by hand.

**Placeholders** look like `<THIS>` — you must replace them (including the `<` and `>`) with your actual values.

Here are the placeholders you will use in this lab:

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name from Lab 1A | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account number | `123456789012` |
| `<YOUR_EMAIL_ADDRESS>` | An email address you can check right now | `jane.doe@gmail.com` |
| `<YOUR_TOPIC_ARN>` | The TopicArn value returned in Step 2 (you will get this during the lab) | `arn:aws:sns:us-east-1:123456789012:workshop-budget-alert` |

---

## Lab Steps

### Step 1: Set Your AWS Profile

Before running any AWS commands, set your default profile so you do not have to type it on every command.

**Windows (PowerShell):**

📋 Copy and paste this command, **replacing `<YOUR_PROFILE_NAME>`** with your actual profile name from Lab 1A:

```powershell
$env:AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**macOS / Linux:**

```bash
export AWS_PROFILE="<YOUR_PROFILE_NAME>"
```

**Verify it works.** 📋 Copy and paste this and press Enter:

```
aws sts get-caller-identity
```

**✅ You should see** your account ID and role. If you get an error about an expired token, run `aws sso login --profile <YOUR_PROFILE_NAME>` first, then set the profile again.

---

### Step 2: Create an SNS Topic (Your Notification Channel)

First, we will create an SNS topic. This is the "mailing list" that will receive budget alerts and forward them to your email.

📋 Copy and paste this command exactly as-is (no placeholders to replace here):

```
aws sns create-topic --name workshop-budget-alert --region us-east-1
```

**What does this do?**
- `aws sns create-topic` — tells AWS to create a new SNS topic
- `--name workshop-budget-alert` — gives the topic a name so we can find it later
- `--region us-east-1` — creates it in the US East (N. Virginia) region

**✅ You should see output like this:**

```json
{
    "TopicArn": "arn:aws:sns:us-east-1:123456789012:workshop-budget-alert"
}
```

> **📝 Important: Copy the `TopicArn` value and save it somewhere** (a notepad, sticky note, etc.). You will need it in the next steps. It looks like:
> ```
> arn:aws:sns:us-east-1:123456789012:workshop-budget-alert
> ```
> This is the unique address of your topic — other AWS services use this address to send messages to it.

**✅ Checkpoint — Verify in the AWS Console:**
1. Open your browser and go to [https://console.aws.amazon.com/](https://console.aws.amazon.com/)
2. In the **search bar at the top**, type **SNS** and click **Simple Notification Service**
3. Make sure the **region** in the top-right corner says **US East (N. Virginia)** — if it says something else, click the region dropdown and select **US East (N. Virginia)**
4. In the left sidebar, click **Topics**
5. You should see **workshop-budget-alert** in the list

---

### Step 3: Subscribe Your Email to the Topic

Now we will add your email address to the topic so you receive notifications.

📋 Copy and paste this command, **replacing the two placeholders**:

```
aws sns subscribe --topic-arn <YOUR_TOPIC_ARN> --protocol email --notification-endpoint <YOUR_EMAIL_ADDRESS> --region us-east-1
```

> **🔄 Example:** If your TopicArn is `arn:aws:sns:us-east-1:123456789012:workshop-budget-alert` and your email is `jane.doe@gmail.com`, the command would be:
> ```
> aws sns subscribe --topic-arn arn:aws:sns:us-east-1:123456789012:workshop-budget-alert --protocol email --notification-endpoint jane.doe@gmail.com --region us-east-1
> ```

**What does this do?**
- `aws sns subscribe` — adds a subscriber to the topic
- `--topic-arn <YOUR_TOPIC_ARN>` — which topic to subscribe to
- `--protocol email` — send notifications via email
- `--notification-endpoint <YOUR_EMAIL_ADDRESS>` — the email address to send to

**✅ You should see:**

```json
{
    "SubscriptionArn": "pending confirmation"
}
```

> The status says "pending confirmation" because AWS needs to verify that you actually own this email address. You will confirm it in the next step.

---

### Step 4: Confirm Your Email Subscription

AWS just sent a confirmation email to the address you provided. You need to click the link in that email to activate the subscription.

1. **Open your email inbox** (check the address you used in Step 3)
2. **Look for an email** from **AWS Notifications** with the subject "AWS Notification - Subscription Confirmation"
3. **Check your spam/junk folder** if you do not see it in your inbox
4. **Click the "Confirm subscription" link** in the email
5. A web page will open confirming your subscription is active

> **⏳ If the email has not arrived after 2–3 minutes**, check your spam folder. If it is still not there, go back to Step 3 and run the subscribe command again.

**✅ Checkpoint — Verify in the AWS Console:**
1. Go back to the AWS Console → **SNS** → **Subscriptions** (in the left sidebar)
2. Find the subscription for your email address
3. The **Status** column should say **Confirmed** (not "PendingConfirmation")

---

### Step 5: Create the Budget Definition File

AWS Budgets needs a configuration file that describes your budget. You will create a small JSON file (a structured text file that AWS can read).

**Create a new file** called `budget.json` in your current directory. You can do this with any text editor (VS Code, Notepad, etc.).

📋 Copy and paste this entire block into the file and save it:

```json
{
    "BudgetName": "workshop-monthly-budget",
    "BudgetLimit": {
        "Amount": "1",
        "Unit": "USD"
    },
    "BudgetType": "COST",
    "TimeUnit": "MONTHLY"
}
```

**What does each line mean?**

| Field | Value | What It Means |
|-------|-------|---------------|
| `BudgetName` | `workshop-monthly-budget` | A name for your budget — you will use this to find and delete it later |
| `BudgetLimit.Amount` | `1` | The budget limit in dollars — we set $1 so even tiny spending triggers the alert |
| `BudgetLimit.Unit` | `USD` | The currency (US Dollars) |
| `BudgetType` | `COST` | We are tracking actual dollar spending (not usage counts) |
| `TimeUnit` | `MONTHLY` | The budget resets at the start of each month |

---

### Step 6: Create the Notification Configuration File

Now create a second file that tells AWS Budgets **when** to send an alert and **where** to send it.

**Create a new file** called `notifications.json` in the same directory.

📋 Copy and paste this into the file, **replacing `<YOUR_TOPIC_ARN>`** with the TopicArn you saved in Step 2, then save:

```json
[
    {
        "Notification": {
            "NotificationType": "ACTUAL",
            "ComparisonOperator": "GREATER_THAN",
            "Threshold": 80,
            "ThresholdType": "PERCENTAGE"
        },
        "Subscribers": [
            {
                "SubscriptionType": "SNS",
                "Address": "<YOUR_TOPIC_ARN>"
            }
        ]
    }
]
```

**What does each part mean?**

| Field | Value | What It Means |
|-------|-------|---------------|
| `NotificationType` | `ACTUAL` | Check real spending (not forecasted/predicted spending) |
| `ComparisonOperator` | `GREATER_THAN` | Alert when spending goes **above** the threshold |
| `Threshold` | `80` | Alert at 80% of the budget limit |
| `ThresholdType` | `PERCENTAGE` | The 80 is a percentage, not a dollar amount |
| `SubscriptionType` | `SNS` | Send the alert to an SNS topic |
| `Address` | Your TopicArn | Which SNS topic to send to (the one with your email subscribed) |

> **In plain English:** "When my actual spending exceeds 80% of my $1 budget ($0.80), send an alert to my SNS topic, which will forward it to my email."

---

### Step 7: Create the Budget

Now you will create the budget using both JSON files.

📋 Copy and paste this command, **replacing `<YOUR_ACCOUNT_ID>`** with your 12-digit AWS account number:

**macOS / Linux:**

```bash
aws budgets create-budget \
    --account-id <YOUR_ACCOUNT_ID> \
    --budget file://budget.json \
    --notifications-with-subscribers file://notifications.json
```

**Windows (PowerShell):**

```powershell
aws budgets create-budget --account-id <YOUR_ACCOUNT_ID> --budget file://budget.json --notifications-with-subscribers file://notifications.json
```

> **🔄 Example:** If your account ID is `123456789012`:
> ```powershell
> aws budgets create-budget --account-id 123456789012 --budget file://budget.json --notifications-with-subscribers file://notifications.json
> ```

**What does this do?**
- `aws budgets create-budget` — tells AWS to create a new budget
- `--account-id` — which AWS account to create the budget in
- `--budget file://budget.json` — reads the budget settings from your file
- `--notifications-with-subscribers file://notifications.json` — reads the alert settings from your file

**✅ If the command succeeds, you will see no output** — the terminal will just show a new prompt. No output means success for this command.

> **🔧 If you see an error:** `An error occurred (DuplicateBudgetException)` — this means a budget with this name already exists. Delete it first with:
> ```
> aws budgets delete-budget --account-id <YOUR_ACCOUNT_ID> --budget-name workshop-monthly-budget
> ```
> Then try the create command again.

---

### Step 8: Verify the Budget Was Created

📋 Copy and paste this command, **replacing `<YOUR_ACCOUNT_ID>`**:

```
aws budgets describe-budgets --account-id <YOUR_ACCOUNT_ID> --output table
```

**What does this do?** This lists all budgets on your account in a readable table format.

**✅ You should see a table** that includes:
- `BudgetName`: `workshop-monthly-budget`
- `BudgetLimit.Amount`: `1.0`
- `TimeUnit`: `MONTHLY`

**✅ Checkpoint — Verify in the AWS Console:**
1. In the AWS Console, search for **Budgets** in the top search bar
2. Click **AWS Budgets**
3. You should see **workshop-monthly-budget** in the list
4. Click on it to see the details — you should see the $1.00 limit and the notification configuration

---

## What You Just Did

You set up **cost monitoring** on an AWS account — one of the first things any cloud engineer does when managing cloud infrastructure. Here is what you built:

1. **An SNS topic** — a notification channel that can send emails
2. **An email subscription** — your email is connected to that channel
3. **A cost budget** — AWS watches your spending and alerts you when it approaches $1

In a real production environment, these alerts prevent surprise bills and are part of the **AWS Well-Architected Framework's Cost Optimization pillar**. Companies set budgets on every AWS account to maintain cost visibility.

---

## Cert Prep Callout

**Target Certification:** AWS Solutions Architect – Associate (SAA)

This lab covers concepts from the **Cost Optimization** pillar of the AWS Well-Architected Framework. The SAA exam tests your ability to design cost-efficient architectures, including setting up billing alarms and budgets.

**Sample question type:** "A company wants to be notified when their monthly AWS spending exceeds a threshold. Which AWS service should they use?"

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `An error occurred (DuplicateBudgetException)` | A budget with this name already exists | Delete it first: `aws budgets delete-budget --account-id <YOUR_ACCOUNT_ID> --budget-name workshop-monthly-budget` then try again |
| Confirmation email not received | AWS sent it but it may be in spam | Check your spam/junk folder. Wait 2–3 minutes. If still nothing, run the subscribe command from Step 3 again. |
| `An error occurred (NotFoundException)` when creating budget | The TopicArn in your notifications.json is wrong | Open `notifications.json` and make sure the `Address` value exactly matches the TopicArn from Step 2. Check for typos or extra spaces. |
| `An error occurred (InvalidParameterException)` | Something in your JSON file is formatted incorrectly | Make sure you saved the files as `.json` (not `.json.txt`). Open them and verify the content matches the examples exactly. |

---

## Cleanup

**⚠️ Important:** Always clean up resources after completing a lab to avoid unexpected charges. Follow these steps in order.

### Step 1: Delete the Budget

📋 Copy and paste this command, **replacing `<YOUR_ACCOUNT_ID>`**:

```
aws budgets delete-budget --account-id <YOUR_ACCOUNT_ID> --budget-name "workshop-monthly-budget"
```

**✅ No output means success.**

### Step 2: Delete the SNS Topic

Deleting the topic automatically removes all subscriptions attached to it.

📋 Copy and paste this command, **replacing `<YOUR_TOPIC_ARN>`**:

```
aws sns delete-topic --topic-arn <YOUR_TOPIC_ARN> --region us-east-1
```

**✅ No output means success.**

### Step 3: Verify Everything Is Gone

📋 Run these two commands to confirm cleanup:

```
aws budgets describe-budgets --account-id <YOUR_ACCOUNT_ID> --output json
```

**✅ You should see** an empty `Budgets` list, or `workshop-monthly-budget` should not appear.

```
aws sns list-topics --region us-east-1
```

**✅ You should see** that `workshop-budget-alert` is no longer in the list.

**✅ Checkpoint — Verify in the AWS Console:**
1. Go to **AWS Budgets** — confirm `workshop-monthly-budget` is gone
2. Go to **SNS > Topics** — confirm `workshop-budget-alert` is gone

### Step 4: Delete the JSON Files

Remove the temporary files you created on your computer:

**macOS / Linux:**

```bash
rm budget.json notifications.json
```

**Windows (PowerShell):**

```powershell
Remove-Item budget.json, notifications.json
```

---

## Help

If you get stuck, post your question in the **Lab Help** channel on Microsoft Teams. Include:
1. The **command** you ran (copy and paste it)
2. The **full error message** you received (copy and paste it)
3. Which **step number** you are on
