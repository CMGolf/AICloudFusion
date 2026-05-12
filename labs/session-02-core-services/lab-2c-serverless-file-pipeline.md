# Lab 2C: Build a Serverless File Processing Pipeline

**Session:** 2 — Core AWS Services  
**Track:** Cloud Basics  
**Difficulty:** Advanced  
**Estimated Time:** 40–50 minutes

---

## Overview

In this lab, you will build an **automated file processing pipeline** using AWS Lambda and S3. When you upload a text file to an input bucket, a Lambda function will automatically trigger, process the file (convert it to uppercase and count the words), and save the result to an output bucket.

**What you will build:**
- An **input S3 bucket** — where you upload files
- An **output S3 bucket** — where processed results appear
- A **Lambda function** — serverless code that runs automatically when a file is uploaded
- An **S3 event trigger** — connects the bucket to the Lambda so it fires on every upload

This is a real-world pattern used in production systems: image resizing pipelines, log processing, data transformation, document conversion, and more. You will walk away with a working automated system that processes files without any servers to manage.

---

## Prerequisites

- ✅ Completed **Lab 1A** (AWS account, CLI configured)
- ✅ Familiarity with S3 (from Labs 1C and 2B)
- ✅ AWS CLI authenticated
- ✅ A text editor for creating files

---

## Cost Notice

| Service | What It Is | Free Tier Limit |
|---------|-----------|----------------|
| AWS Lambda | Serverless compute (runs code without servers) | 1 million requests/month + 400,000 GB-seconds Always Free |
| Amazon S3 | Cloud storage | 5 GB storage free for 12 months |
| IAM | Access management | Always Free |
| CloudWatch Logs | Lambda execution logs | 5 GB ingestion/month Always Free |

**Estimated cost for this lab: $0.00**

---

## Concepts

**AWS Lambda** is a serverless compute service. You write code, upload it to Lambda, and AWS runs it for you — no servers to manage, no operating systems to patch, no scaling to configure. You only pay for the time your code actually runs (and the first 1 million requests per month are free).

**Event-driven architecture** means your code runs in response to events. In this lab, the event is "a file was uploaded to S3." Lambda listens for that event and automatically runs your code. You don't need to poll, check, or schedule anything — it just happens.

**S3 Event Notifications** is the feature that connects S3 to Lambda. You configure a bucket to send a notification whenever a file is created, and Lambda receives that notification and runs your function.

**Why this matters for jobs:** Serverless and event-driven architectures are increasingly common. Companies use Lambda for everything from processing uploads to running APIs to automating workflows. Understanding how to wire services together (S3 → Lambda → S3) is a core cloud engineering skill.

---

## ⚠️ Placeholders in This Lab

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account number | `123456789012` |
| `<INPUT_BUCKET>` | A unique name for your input bucket | `jane-doe-workshop-input` |
| `<OUTPUT_BUCKET>` | A unique name for your output bucket | `jane-doe-workshop-output` |

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

### Step 2: Create the Input and Output S3 Buckets

📋 Copy and paste, **replacing the bucket names** with your own unique names:

```
aws s3 mb s3://<INPUT_BUCKET> --region us-east-1
```

```
aws s3 mb s3://<OUTPUT_BUCKET> --region us-east-1
```

**✅ You should see** `make_bucket:` for each.

> **📝 Write down your bucket names:**
> - Input bucket: ______________________________
> - Output bucket: ______________________________

---

### Step 3: Create the Lambda Execution Role

Lambda needs an IAM role that gives it permission to read from the input bucket, write to the output bucket, and write logs to CloudWatch.

**Create the trust policy file** (`lambda-trust-policy.json`):

📋 Copy and paste into a new file and save:

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

> **What does this do?** Tells AWS that Lambda functions are allowed to use this role.

**Create the role and attach policies.** 📋 Copy and paste:

```
aws iam create-role --role-name workshop-lambda-s3-role --assume-role-policy-document file://lambda-trust-policy.json
```

```
aws iam attach-role-policy --role-name workshop-lambda-s3-role --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
```

```
aws iam attach-role-policy --role-name workshop-lambda-s3-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

> **What do these policies do?**
> - `AmazonS3FullAccess` — lets the Lambda read from and write to S3 buckets
> - `AWSLambdaBasicExecutionRole` — lets the Lambda write logs to CloudWatch (so you can see what happened)

**Wait 10 seconds** for IAM to propagate.

---

### Step 4: Write the Lambda Function Code

Create a file called `lambda_function.py` with the following code:

📋 Copy and paste this entire block into a new file and save it as `lambda_function.py`:

```python
import json
import boto3
import os

s3_client = boto3.client('s3')

def lambda_handler(event, context):
    # Get the bucket and file name from the S3 event
    source_bucket = event['Records'][0]['s3']['bucket']['name']
    source_key = event['Records'][0]['s3']['object']['key']
    
    # Only process .txt files
    if not source_key.endswith('.txt'):
        print(f"Skipping non-txt file: {source_key}")
        return {'statusCode': 200, 'body': 'Skipped'}
    
    # Download the file from the input bucket
    response = s3_client.get_object(Bucket=source_bucket, Key=source_key)
    original_text = response['Body'].read().decode('utf-8')
    
    # Process the file: convert to uppercase and count words
    processed_text = original_text.upper()
    word_count = len(original_text.split())
    
    # Build the output content
    output_content = f"=== PROCESSED FILE ===\n"
    output_content += f"Original file: {source_key}\n"
    output_content += f"Word count: {word_count}\n"
    output_content += f"=== CONTENT (UPPERCASE) ===\n"
    output_content += processed_text
    
    # Upload the processed file to the output bucket
    output_bucket = os.environ['OUTPUT_BUCKET']
    output_key = f"processed-{source_key}"
    
    s3_client.put_object(
        Bucket=output_bucket,
        Key=output_key,
        Body=output_content.encode('utf-8'),
        ContentType='text/plain'
    )
    
    print(f"Processed {source_key} -> {output_key}")
    
    return {
        'statusCode': 200,
        'body': json.dumps(f'Processed {source_key} successfully')
    }
```

> **What does this code do? (line by line)**
>
> 1. When a file is uploaded to the input bucket, S3 sends an "event" to Lambda with the bucket name and file name
> 2. The function downloads the file from S3
> 3. It converts all the text to UPPERCASE and counts the words
> 4. It creates a new file with the processed content plus a header showing the word count
> 5. It uploads the processed file to the output bucket with `processed-` added to the filename
>
> **You don't need to know Python to complete this lab** — just copy the code exactly as shown.

---

### Step 5: Package and Deploy the Lambda Function

Lambda functions need to be uploaded as a `.zip` file.

**macOS / Linux:**

```bash
zip lambda.zip lambda_function.py
```

**Windows (PowerShell):**

```powershell
Compress-Archive -Path lambda_function.py -DestinationPath lambda.zip -Force
```

**Now deploy the function.** 📋 Copy and paste, **replacing `<YOUR_ACCOUNT_ID>` and `<OUTPUT_BUCKET>`**:

**Windows (PowerShell):**

```powershell
aws lambda create-function --function-name workshop-file-processor --runtime python3.12 --role "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-lambda-s3-role" --handler lambda_function.lambda_handler --zip-file fileb://lambda.zip --environment "Variables={OUTPUT_BUCKET=<OUTPUT_BUCKET>}" --region us-east-1
```

**macOS / Linux:**

```bash
aws lambda create-function \
    --function-name workshop-file-processor \
    --runtime python3.12 \
    --role "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-lambda-s3-role" \
    --handler lambda_function.lambda_handler \
    --zip-file fileb://lambda.zip \
    --environment "Variables={OUTPUT_BUCKET=<OUTPUT_BUCKET>}" \
    --region us-east-1
```

> **What does this do?**
> - `--function-name` — names the function so you can find it later
> - `--runtime python3.12` — tells Lambda to use Python 3.12 to run the code
> - `--role` — which IAM role the function uses for permissions
> - `--handler lambda_function.lambda_handler` — which file and function to call (file: `lambda_function.py`, function: `lambda_handler`)
> - `--zip-file fileb://lambda.zip` — the code package to upload (`fileb://` means "binary file")
> - `--environment` — passes the output bucket name to the function as a configuration variable

**✅ You should see** a JSON response with the function details including `"State": "Active"`.

**✅ Checkpoint — In the AWS Console:**
1. Search for **Lambda** → click it
2. You should see **workshop-file-processor** in the functions list

---

### Step 6: Grant S3 Permission to Invoke the Lambda

S3 needs explicit permission to trigger your Lambda function.

📋 Copy and paste, **replacing `<INPUT_BUCKET>`**:

```
aws lambda add-permission --function-name workshop-file-processor --statement-id s3-trigger --action lambda:InvokeFunction --principal s3.amazonaws.com --source-arn arn:aws:s3:::<INPUT_BUCKET> --region us-east-1
```

> **What does this do?** Tells Lambda "allow S3 to invoke this function, but only from this specific bucket."

---

### Step 7: Configure S3 to Trigger Lambda on File Upload

Create a file called `s3-notification.json` that tells S3 to notify Lambda when a `.txt` file is uploaded.

📋 Copy and paste into a new file, **replacing `<YOUR_ACCOUNT_ID>`**, and save:

```json
{
    "LambdaFunctionConfigurations": [
        {
            "LambdaFunctionArn": "arn:aws:lambda:us-east-1:<YOUR_ACCOUNT_ID>:function:workshop-file-processor",
            "Events": ["s3:ObjectCreated:*"],
            "Filter": {
                "Key": {
                    "FilterRules": [
                        {
                            "Name": "suffix",
                            "Value": ".txt"
                        }
                    ]
                }
            }
        }
    ]
}
```

> **What does this do?**
> - `Events: s3:ObjectCreated:*` — trigger on any new file upload
> - `Filter: suffix .txt` — only trigger for files ending in `.txt`

**Apply the notification configuration.** 📋 Copy and paste, **replacing `<INPUT_BUCKET>`**:

```
aws s3api put-bucket-notification-configuration --bucket <INPUT_BUCKET> --notification-configuration file://s3-notification.json --region us-east-1
```

**✅ No output means success.** Your pipeline is now wired up!

---

### Step 8: Test the Pipeline!

Create a test file and upload it to the input bucket. The Lambda should automatically process it and put the result in the output bucket.

📋 Create a file called `my-test-file.txt` with some text:

```
echo "Cloud computing is transforming how businesses operate. AWS provides over 200 services that help companies build, deploy, and scale applications in the cloud. In this workshop, we are learning the fundamentals." > my-test-file.txt
```

> **Windows PowerShell:**
> ```powershell
> "Cloud computing is transforming how businesses operate. AWS provides over 200 services that help companies build, deploy, and scale applications in the cloud. In this workshop, we are learning the fundamentals." | Out-File -Encoding utf8 my-test-file.txt
> ```

**Upload it to the input bucket.** 📋 Copy and paste, **replacing `<INPUT_BUCKET>`**:

```
aws s3 cp my-test-file.txt s3://<INPUT_BUCKET>/my-test-file.txt
```

**Wait 5–10 seconds** for Lambda to process the file.

**Check the output bucket.** 📋 Copy and paste, **replacing `<OUTPUT_BUCKET>`**:

```
aws s3 ls s3://<OUTPUT_BUCKET>/
```

**✅ You should see:** `processed-my-test-file.txt`

**Download and view the processed file.** 📋 Copy and paste, **replacing `<OUTPUT_BUCKET>`**:

```
aws s3 cp s3://<OUTPUT_BUCKET>/processed-my-test-file.txt -
```

> (The `-` at the end means "print to the terminal instead of saving to a file")

**✅ You should see something like:**

```
=== PROCESSED FILE ===
Original file: my-test-file.txt
Word count: 35
=== CONTENT (UPPERCASE) ===
CLOUD COMPUTING IS TRANSFORMING HOW BUSINESSES OPERATE. AWS PROVIDES OVER 200 SERVICES THAT HELP COMPANIES BUILD, DEPLOY, AND SCALE APPLICATIONS IN THE CLOUD. IN THIS WORKSHOP, WE ARE LEARNING THE FUNDAMENTALS.
```

> **🎉 Your pipeline works!** You uploaded a file, Lambda automatically processed it, and the result appeared in the output bucket — all without you doing anything. This is event-driven, serverless computing in action.

---

### Step 9: Test with Another File

Try uploading a different file to confirm the pipeline handles multiple files:

```
echo "This is a second test file to prove the pipeline works repeatedly." > second-test.txt
```

```
aws s3 cp second-test.txt s3://<INPUT_BUCKET>/second-test.txt
```

Wait 5–10 seconds, then check:

```
aws s3 ls s3://<OUTPUT_BUCKET>/
```

**✅ You should see both** `processed-my-test-file.txt` and `processed-second-test.txt`.

---

## What You Just Did

You built a **serverless file processing pipeline** — a real-world architecture pattern used by companies for:
- **Image processing** — resize uploaded photos automatically
- **Log analysis** — process log files as they arrive
- **Data transformation** — convert CSV files to JSON, clean data, etc.
- **Document processing** — extract text from PDFs, generate summaries

Your pipeline:
1. Listens for file uploads to an S3 bucket (event-driven)
2. Automatically triggers a Lambda function (serverless — no servers to manage)
3. Processes the file (converts to uppercase, counts words)
4. Saves the result to a separate output bucket

You did this with zero servers, zero infrastructure management, and zero cost (within free tier). The Lambda function only runs when a file is uploaded — the rest of the time, nothing is running and nothing costs money.

---

## Cert Prep Callout

**Target Certification:** AWS Solutions Architect – Associate (SAA)

Lambda and S3 event notifications are frequently tested on the SAA exam. Understanding event-driven architectures and when to use Lambda vs. EC2 is a core exam topic.

**Sample question type:** "A company needs to automatically resize images when they are uploaded to S3. Which AWS service should they use to process the images?"

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| No file appears in the output bucket after uploading | Lambda may not have triggered, or it errored | Check Lambda logs: AWS Console → Lambda → workshop-file-processor → Monitor tab → View CloudWatch Logs |
| `An error occurred (InvalidArgument)` when setting notification | The Lambda ARN in `s3-notification.json` is wrong | Verify your account ID is correct in the file. Make sure there are no extra spaces. |
| `ResourceConflictException` when creating the function | A function with this name already exists | Delete it first: `aws lambda delete-function --function-name workshop-file-processor --region us-east-1` then try again |
| Lambda logs show "Access Denied" | The Lambda role doesn't have S3 permissions | Verify `AmazonS3FullAccess` is attached to `workshop-lambda-s3-role` |

---

## Cleanup

**⚠️ Important:** Clean up all resources to prevent any charges.

### Step 1: Remove the S3 Event Notification

📋 Copy and paste, **replacing `<INPUT_BUCKET>`**:

**Windows (PowerShell):**

```powershell
aws s3api put-bucket-notification-configuration --bucket <INPUT_BUCKET> --notification-configuration "{}" --region us-east-1
```

**macOS / Linux:**

```bash
aws s3api put-bucket-notification-configuration --bucket <INPUT_BUCKET> --notification-configuration '{}' --region us-east-1
```

### Step 2: Delete the Lambda Function

```
aws lambda delete-function --function-name workshop-file-processor --region us-east-1
```

### Step 3: Empty and Delete Both S3 Buckets

📋 Copy and paste, **replacing both bucket names**:

```
aws s3 rm s3://<INPUT_BUCKET> --recursive
```

```
aws s3 rm s3://<OUTPUT_BUCKET> --recursive
```

```
aws s3 rb s3://<INPUT_BUCKET>
```

```
aws s3 rb s3://<OUTPUT_BUCKET>
```

### Step 4: Delete the IAM Role

```
aws iam detach-role-policy --role-name workshop-lambda-s3-role --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
```

```
aws iam detach-role-policy --role-name workshop-lambda-s3-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

```
aws iam delete-role --role-name workshop-lambda-s3-role
```

### Step 5: Delete Local Files

**macOS / Linux:**

```bash
rm lambda_function.py lambda.zip lambda-trust-policy.json s3-notification.json my-test-file.txt second-test.txt
```

**Windows (PowerShell):**

```powershell
Remove-Item lambda_function.py, lambda.zip, lambda-trust-policy.json, s3-notification.json, my-test-file.txt, second-test.txt
```

### Step 6: Verify Cleanup

**✅ Checkpoint:**
1. **Lambda** → Functions → `workshop-file-processor` is gone
2. **S3** → both buckets are gone
3. **IAM** → Roles → `workshop-lambda-s3-role` is gone

---

## Help

If you get stuck, post your question in the **Lab Help** channel on Microsoft Teams. Include:
1. The **command** you ran
2. The **full error message**
3. Which **step number** you are on
