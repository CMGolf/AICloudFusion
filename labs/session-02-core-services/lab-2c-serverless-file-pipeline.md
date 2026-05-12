# Lab 2C: Build a Serverless File Processing App with a Web Frontend

**Session:** 2 — Core AWS Services  
**Track:** Cloud Basics  
**Difficulty:** Advanced  
**Estimated Time:** 50–60 minutes

---

## Overview

In this lab, you will build a **complete serverless application** — a web page where you can upload text, have it automatically processed by AWS Lambda, and see the result displayed on the page. This ties together everything from Sessions 1 and 2: S3 static website hosting, Lambda functions, IAM roles, S3 event triggers, and presigned URLs.

**What you will build:**
- A **static website** hosted on S3 (like Lab 1C) with a file upload form
- A **presign Lambda function** that generates secure upload URLs for the browser
- A **Lambda Function URL** so the web page can call the presign function
- An **input S3 bucket** where uploaded files land
- A **processing Lambda function** that triggers automatically on upload, converts text to uppercase, and counts words
- An **output S3 bucket** where processed results are stored and publicly readable
- The web page **displays the processed result** and provides a download link

This is a real-world architecture pattern — serverless web applications that process user uploads without any servers running 24/7.

---

## Prerequisites

- ✅ Completed **Lab 1A** (AWS account, CLI configured)
- ✅ Familiarity with S3 static website hosting (Lab 1C)
- ✅ AWS CLI authenticated
- ✅ A text editor for creating files

---

## Cost Notice

| Service | What It Is | Free Tier Limit |
|---------|-----------|----------------|
| AWS Lambda | Serverless compute | 1 million requests/month Always Free |
| Amazon S3 | Cloud storage + website hosting | 5 GB storage free for 12 months |
| Lambda Function URLs | Public HTTP endpoint for Lambda | Included with Lambda free tier |
| IAM | Access management | Always Free |

**Estimated cost for this lab: $0.00**

---

## Concepts

**Lambda Function URL** is a simple way to give your Lambda function a public HTTP endpoint (a URL that anyone can call). Instead of setting up API Gateway (which is more complex), you create a Function URL and the browser can call it directly. We use this so the web page can ask Lambda for a presigned upload URL.

**Presigned URL** is a temporary URL that grants permission to upload a file to S3. Normally, S3 buckets are private. A presigned URL says "anyone with this link can upload one file, but only for the next 5 minutes." This is how the web page uploads files without needing AWS credentials in the browser.

**CORS (Cross-Origin Resource Sharing)** is a browser security feature. When your web page (hosted on one domain) tries to call a Lambda URL or fetch from S3 (different domains), the browser blocks it unless those services explicitly allow it. We configure CORS on our buckets and Lambda to permit these cross-origin requests.

---

## ⚠️ Placeholders in This Lab

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account number | `123456789012` |
| `<INPUT_BUCKET>` | Unique name for input bucket | `jane-doe-pipeline-input` |
| `<OUTPUT_BUCKET>` | Unique name for output bucket | `jane-doe-pipeline-output` |
| `<WEBSITE_BUCKET>` | Unique name for website bucket | `jane-doe-pipeline-website` |
| `<FUNCTION_URL>` | The Lambda Function URL (you'll get this in Step 8) | `https://abc123.lambda-url.us-east-1.on.aws/` |

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

### Step 2: Create Three S3 Buckets

You need three buckets: one for the website, one for file uploads (input), and one for processed results (output).

📋 Copy and paste, **replacing the bucket names** with your own unique names:

```
aws s3 mb s3://<INPUT_BUCKET> --region us-east-1
aws s3 mb s3://<OUTPUT_BUCKET> --region us-east-1
aws s3 mb s3://<WEBSITE_BUCKET> --region us-east-1
```

---

### Step 3: Create the Lambda IAM Role

Both Lambda functions need a role that gives them permission to access S3 and write logs. You will create this role now.

**Step 3a: Create the trust policy file**

Open your text editor and create a **new file**. 📋 Copy and paste this into the file:

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

**Save the file as `lambda-trust-policy.json`** in the same folder where you are running your terminal commands.

> **What is this file?** It tells AWS "Lambda functions are allowed to use this role." Without it, Lambda can't assume the role's permissions.

---

**Step 3b: Create the role and attach permissions**

📋 Copy and paste these three commands into your terminal, **one at a time**, pressing Enter after each:

```
aws iam create-role --role-name workshop-pipeline-role --assume-role-policy-document file://lambda-trust-policy.json
```

> **What does this do?** Creates a new IAM role called `workshop-pipeline-role`.

```
aws iam attach-role-policy --role-name workshop-pipeline-role --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
```

> **What does this do?** Gives the role permission to read from and write to any S3 bucket.

```
aws iam attach-role-policy --role-name workshop-pipeline-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

> **What does this do?** Gives the role permission to write logs to CloudWatch (so you can see what happened when the function runs).

**⏳ Wait 10 seconds** before continuing to the next step. IAM changes take a moment to propagate across AWS.

---

### Step 4: Create the Presign Lambda Function

Now you will create the first Lambda function. This function's job is simple: when the web page asks for an upload link, this function generates a temporary URL that allows the upload.

**You do NOT need to know Python to complete this step.** You are just copying pre-written code into a file. The code is provided for you — just follow the steps exactly.

---

**Step 4a: Open your text editor and create a new file**

Open your text editor (VS Code, Notepad, or any editor). Create a **new, empty file**.

> **💡 Important:** Make sure you are saving files in the **same folder** where you have been running your terminal commands. If you're not sure which folder that is, type `pwd` in your terminal (macOS/Linux) or check the path shown in your PowerShell prompt (e.g., `C:\Users\YourName`).

---

**Step 4b: Copy and paste the code below into the file**

📋 Copy this **entire block** of code and paste it into your new file. Do not change anything — copy it exactly as shown:

```python
import json
import boto3
import os

s3_client = boto3.client('s3')

def lambda_handler(event, context):
    params = event.get('queryStringParameters', {}) or {}
    filename = params.get('filename', 'upload.txt')
    
    input_bucket = os.environ['INPUT_BUCKET']
    
    presigned_url = s3_client.generate_presigned_url(
        'put_object',
        Params={
            'Bucket': input_bucket,
            'Key': filename,
            'ContentType': 'text/plain'
        },
        ExpiresIn=300
    )
    
    return {
        'statusCode': 200,
        'body': json.dumps({'uploadUrl': presigned_url, 'filename': filename})
    }
```

---

**Step 4c: Save the file as `presign_function.py`**

Save the file with the **exact name** `presign_function.py`. The name matters — Lambda uses it to find the code.

> **⚠️ Common mistakes:**
> - Make sure the file extension is `.py` (not `.py.txt` or `.txt`)
> - Make sure there are no extra spaces or blank lines at the beginning of the file
> - If using Notepad on Windows, change "Save as type" to "All Files" before saving, otherwise it may add `.txt` to the end

---

**Step 4d: What does this code do? (optional reading)**

You don't need to understand the code to complete this lab, but here's what each part does if you're curious:

| Line(s) | What It Does |
|---------|-------------|
| `import json, boto3, os` | Loads libraries that the code needs (JSON handling, AWS SDK, environment variables) |
| `s3_client = boto3.client('s3')` | Creates a connection to the S3 service |
| `def lambda_handler(event, context):` | This is the function that Lambda calls when triggered. Think of it as the "main" function. |
| `params = event.get(...)` | Gets the filename from the web page's request |
| `input_bucket = os.environ['INPUT_BUCKET']` | Reads the input bucket name from a configuration variable (you'll set this when deploying) |
| `s3_client.generate_presigned_url(...)` | Generates a temporary URL that allows uploading one file to S3 |
| `ExpiresIn=300` | The URL is valid for 300 seconds (5 minutes) |
| `return {...}` | Sends the URL back to the web page |

---

**Step 4e: Package the code into a zip file**

Lambda requires code to be uploaded as a `.zip` file. You will zip the file you just created.

**macOS / Linux:**

📋 Copy and paste this command into your terminal:

```bash
zip presign.zip presign_function.py
```

**Windows (PowerShell):**

📋 Copy and paste this command into your terminal:

```powershell
Compress-Archive -Path presign_function.py -DestinationPath presign.zip -Force
```

> **What does this do?** Creates a file called `presign.zip` that contains your `presign_function.py`. This is the package you will upload to Lambda.

**✅ You should now have two files in your folder:** `presign_function.py` and `presign.zip`

---

**Step 4f: Deploy the function to AWS Lambda**

Now you will upload the zip file to Lambda and create the function.

📋 Copy and paste this command, **replacing `<YOUR_ACCOUNT_ID>` and `<INPUT_BUCKET>`** with your actual values:

**Windows (PowerShell):**

```powershell
aws lambda create-function --function-name workshop-presign --runtime python3.12 --role "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-pipeline-role" --handler presign_function.lambda_handler --zip-file fileb://presign.zip --environment "Variables={INPUT_BUCKET=<INPUT_BUCKET>}" --region us-east-1
```

**macOS / Linux:**

```bash
aws lambda create-function \
    --function-name workshop-presign \
    --runtime python3.12 \
    --role "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-pipeline-role" \
    --handler presign_function.lambda_handler \
    --zip-file fileb://presign.zip \
    --environment "Variables={INPUT_BUCKET=<INPUT_BUCKET>}" \
    --region us-east-1
```

> **What does each part mean?**
> - `--function-name workshop-presign` — names the function so you can find it later
> - `--runtime python3.12` — tells Lambda to use Python to run the code
> - `--role "arn:aws:iam::..."` — which permissions the function has (the role you created in Step 3)
> - `--handler presign_function.lambda_handler` — tells Lambda "look in the file `presign_function.py` and call the function named `lambda_handler`"
> - `--zip-file fileb://presign.zip` — the code package to upload (`fileb://` means "binary file from my computer")
> - `--environment "Variables={INPUT_BUCKET=...}"` — passes your input bucket name to the function as a configuration variable (this is how the code knows which bucket to generate URLs for)

**✅ You should see** a JSON response with details about the function, including `"State": "Active"`.

**✅ Checkpoint — In the AWS Console:**
1. Search for **Lambda** → click it
2. Make sure you're in the **us-east-1** region
3. You should see **workshop-presign** in the functions list

---

### Step 5: Create the Processing Lambda Function

Now you will create the second Lambda function. This one triggers automatically when a file is uploaded to the input bucket. It reads the file, converts the text to uppercase, counts the words, and saves the result to the output bucket.

Again — **you do NOT need to know Python.** Just copy the code exactly as shown.

---

**Step 5a: Open your text editor and create a new file**

Create another **new, empty file** in your text editor. This will be a separate file from the one you created in Step 4.

---

**Step 5b: Copy and paste the code below into the file**

📋 Copy this **entire block** of code and paste it into your new file:

```python
import json
import boto3
import os

s3_client = boto3.client('s3')

def lambda_handler(event, context):
    source_bucket = event['Records'][0]['s3']['bucket']['name']
    source_key = event['Records'][0]['s3']['object']['key']
    
    if not source_key.endswith('.txt'):
        return {'statusCode': 200, 'body': 'Skipped'}
    
    response = s3_client.get_object(Bucket=source_bucket, Key=source_key)
    original_text = response['Body'].read().decode('utf-8')
    
    processed_text = original_text.upper()
    word_count = len(original_text.split())
    
    output_content = f"=== PROCESSED FILE ===\n"
    output_content += f"Original file: {source_key}\n"
    output_content += f"Word count: {word_count}\n"
    output_content += f"=== CONTENT (UPPERCASE) ===\n"
    output_content += processed_text
    
    output_bucket = os.environ['OUTPUT_BUCKET']
    output_key = f"processed-{source_key}"
    
    s3_client.put_object(
        Bucket=output_bucket,
        Key=output_key,
        Body=output_content.encode('utf-8'),
        ContentType='text/plain'
    )
    
    return {'statusCode': 200, 'body': json.dumps(f'Processed {source_key}')}
```

---

**Step 5c: Save the file as `process_function.py`**

Save the file with the **exact name** `process_function.py` in the **same folder** as your other files.

> **⚠️ Same warnings as before:** Make sure the extension is `.py`, not `.py.txt`. Make sure there are no extra spaces at the beginning.

---

**Step 5d: What does this code do? (optional reading)**

| Line(s) | What It Does |
|---------|-------------|
| `source_bucket = event['Records'][0]...` | When S3 triggers this function, it sends information about which file was uploaded. This line reads the bucket name and filename. |
| `if not source_key.endswith('.txt'):` | Only process `.txt` files — skip everything else |
| `s3_client.get_object(...)` | Downloads the uploaded file from S3 |
| `original_text.upper()` | Converts all text to UPPERCASE |
| `len(original_text.split())` | Counts the number of words |
| `output_content = ...` | Builds the processed output with a header showing the filename and word count |
| `output_bucket = os.environ['OUTPUT_BUCKET']` | Reads the output bucket name from configuration |
| `s3_client.put_object(...)` | Uploads the processed result to the output bucket |
| `output_key = f"processed-{source_key}"` | Names the output file `processed-` plus the original filename |

**In plain English:** "When a .txt file is uploaded to the input bucket, download it, convert it to uppercase, count the words, add a header, and save the result to the output bucket."

---

**Step 5e: Package the code into a zip file**

**macOS / Linux:**

📋 Copy and paste:

```bash
zip process.zip process_function.py
```

**Windows (PowerShell):**

📋 Copy and paste:

```powershell
Compress-Archive -Path process_function.py -DestinationPath process.zip -Force
```

**✅ You should now have these files in your folder:** `presign_function.py`, `presign.zip`, `process_function.py`, `process.zip`

---

**Step 5f: Deploy the function to AWS Lambda**

📋 Copy and paste, **replacing `<YOUR_ACCOUNT_ID>` and `<OUTPUT_BUCKET>`**:

**Windows (PowerShell):**

```powershell
aws lambda create-function --function-name workshop-processor --runtime python3.12 --role "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-pipeline-role" --handler process_function.lambda_handler --zip-file fileb://process.zip --environment "Variables={OUTPUT_BUCKET=<OUTPUT_BUCKET>}" --region us-east-1
```

**macOS / Linux:**

```bash
aws lambda create-function \
    --function-name workshop-processor \
    --runtime python3.12 \
    --role "arn:aws:iam::<YOUR_ACCOUNT_ID>:role/workshop-pipeline-role" \
    --handler process_function.lambda_handler \
    --zip-file fileb://process.zip \
    --environment "Variables={OUTPUT_BUCKET=<OUTPUT_BUCKET>}" \
    --region us-east-1
```

> **What's different from the presign function?**
> - Different function name (`workshop-processor`)
> - Different handler (`process_function.lambda_handler` — points to the `process_function.py` file)
> - Different environment variable (`OUTPUT_BUCKET` instead of `INPUT_BUCKET`)

**✅ You should see** a JSON response with `"State": "Active"`.

**✅ Checkpoint — In the AWS Console:**
1. Go to **Lambda** → **Functions**
2. You should now see **two functions**: `workshop-presign` and `workshop-processor`

---

### Step 6: Create a Function URL for the Presign Lambda

This gives the presign function a public URL that the web page can call.

📋 Copy and paste:

```
aws lambda create-function-url-config --function-name workshop-presign --auth-type NONE --cors "AllowOrigins=*,AllowMethods=GET,AllowHeaders=*" --region us-east-1
```

> **What does `--auth-type NONE` mean?** It makes the URL publicly accessible without authentication. This is fine for our use case — the presign function only generates upload URLs, it doesn't expose any sensitive data.

> **What does `--cors` do?** It tells the browser "yes, web pages from other domains are allowed to call this URL." Without this, the browser would block the request.

**✅ You should see** a JSON response with a `FunctionUrl` field. **Copy this URL.**

> **📝 Write down your Function URL:** ______________________________
>
> It looks like: `https://abc123xyz.lambda-url.us-east-1.on.aws/`

**Grant public access:**

```
aws lambda add-permission --function-name workshop-presign --statement-id FunctionURLAllowPublicAccess --action lambda:InvokeFunctionUrl --principal "*" --function-url-auth-type NONE --region us-east-1
```

> **⏳ Wait 1–2 minutes** for the permission to propagate before testing.

---

### Step 7: Configure the S3 Event Trigger

Connect the input bucket to the processing Lambda so it fires automatically on upload.

**Grant S3 permission to invoke the Lambda.** 📋 Replace `<INPUT_BUCKET>`:

```
aws lambda add-permission --function-name workshop-processor --statement-id s3-trigger --action lambda:InvokeFunction --principal s3.amazonaws.com --source-arn arn:aws:s3:::<INPUT_BUCKET> --region us-east-1
```

**Create `s3-notification.json`.** 📋 Replace `<YOUR_ACCOUNT_ID>`:

```json
{
    "LambdaFunctionConfigurations": [
        {
            "LambdaFunctionArn": "arn:aws:lambda:us-east-1:<YOUR_ACCOUNT_ID>:function:workshop-processor",
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

📋 Apply it, **replacing `<INPUT_BUCKET>`**:

```
aws s3api put-bucket-notification-configuration --bucket <INPUT_BUCKET> --notification-configuration file://s3-notification.json --region us-east-1
```

---

### Step 8: Configure CORS on the S3 Buckets

The browser needs permission to upload to the input bucket and read from the output bucket.

**Create `cors.json`:**

```json
{
    "CORSRules": [
        {
            "AllowedHeaders": ["*"],
            "AllowedMethods": ["PUT", "GET"],
            "AllowedOrigins": ["*"],
            "ExposeHeaders": []
        }
    ]
}
```

📋 Apply to both buckets, **replacing the bucket names**:

```
aws s3api put-bucket-cors --bucket <INPUT_BUCKET> --cors-configuration file://cors.json --region us-east-1
```

```
aws s3api put-bucket-cors --bucket <OUTPUT_BUCKET> --cors-configuration file://cors.json --region us-east-1
```

---

### Step 9: Make the Output Bucket Publicly Readable

So the web page can fetch and display the processed results.

📋 Replace `<OUTPUT_BUCKET>`:

```
aws s3api put-public-access-block --bucket <OUTPUT_BUCKET> --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"
```

**Create `output-policy.json`.** 📋 Replace `<OUTPUT_BUCKET>`:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::<OUTPUT_BUCKET>/*"
        }
    ]
}
```

📋 Apply it:

```
aws s3api put-bucket-policy --bucket <OUTPUT_BUCKET> --policy file://output-policy.json
```

---

### Step 10: Create and Deploy the Website

**Set up the website bucket** (same as Lab 1C):

📋 Replace `<WEBSITE_BUCKET>`:

```
aws s3 website s3://<WEBSITE_BUCKET> --index-document index.html --error-document index.html
```

```
aws s3api put-public-access-block --bucket <WEBSITE_BUCKET> --public-access-block-configuration "BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false"
```

**Create `website-policy.json`.** 📋 Replace `<WEBSITE_BUCKET>`:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::<WEBSITE_BUCKET>/*"
        }
    ]
}
```

```
aws s3api put-bucket-policy --bucket <WEBSITE_BUCKET> --policy file://website-policy.json
```

**Create `index.html`.** 📋 Copy this entire file, **replacing `<FUNCTION_URL>` and `<OUTPUT_BUCKET>`** with your actual values:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Cloud File Processor</title>
    <style>
        body { font-family: Arial, sans-serif; max-width: 700px; margin: 50px auto; padding: 20px; background: #f0f4f8; }
        h1 { color: #232f3e; }
        .card { background: white; border-radius: 8px; padding: 30px; box-shadow: 0 2px 8px rgba(0,0,0,0.1); margin: 20px 0; }
        textarea { width: 100%; height: 150px; padding: 12px; border: 2px solid #ddd; border-radius: 6px; font-size: 16px; resize: vertical; }
        button { background: #ff9900; color: white; border: none; padding: 12px 24px; font-size: 16px; border-radius: 6px; cursor: pointer; margin-top: 10px; }
        button:hover { background: #ec7211; }
        button:disabled { background: #ccc; cursor: not-allowed; }
        .status { margin-top: 15px; padding: 12px; border-radius: 6px; display: none; }
        .status.success { display: block; background: #d4edda; color: #155724; border: 1px solid #c3e6cb; }
        .status.error { display: block; background: #f8d7da; color: #721c24; border: 1px solid #f5c6cb; }
        .status.loading { display: block; background: #fff3cd; color: #856404; border: 1px solid #ffeeba; }
        .result { margin-top: 15px; padding: 15px; background: #1a1a2e; color: #00ff88; border-radius: 6px; white-space: pre-wrap; font-family: monospace; font-size: 14px; }
        .badge { background: #232f3e; color: white; padding: 4px 10px; border-radius: 4px; font-size: 12px; }
        a.download-link { display: inline-block; margin-top: 10px; background: #232f3e; color: white; padding: 10px 20px; border-radius: 6px; text-decoration: none; }
        a.download-link:hover { background: #37475a; }
    </style>
</head>
<body>
    <h1>&#9729; Cloud File Processor</h1>
    <p>Upload a text file and watch it get processed automatically by AWS Lambda.</p>
    <p><span class="badge">Serverless</span> <span class="badge">S3</span> <span class="badge">Lambda</span></p>

    <div class="card">
        <h2>Upload a Text File</h2>
        <p>Type or paste some text below, then click Upload. The serverless pipeline will process it automatically.</p>
        <textarea id="fileContent" placeholder="Type your text here..."></textarea>
        <br>
        <label for="fileName">File name: </label>
        <input type="text" id="fileName" value="my-upload.txt" style="padding: 8px; border: 2px solid #ddd; border-radius: 4px;">
        <br>
        <button id="uploadBtn" onclick="uploadFile()">Upload & Process</button>
        <div id="status" class="status"></div>
    </div>

    <div id="resultSection" class="card" style="display:none;">
        <h2>&#9989; Processed Result</h2>
        <p>Lambda processed your file. Here is the output:</p>
        <div id="result" class="result"></div>
        <a id="downloadLink" class="download-link" href="#" target="_blank">&#128229; View Processed File</a>
    </div>

    <div class="card">
        <h2>How It Works</h2>
        <ol>
            <li>You type text and click Upload</li>
            <li>The web page asks Lambda for a <strong>presigned URL</strong> (a temporary upload link)</li>
            <li>The file is uploaded directly to the <strong>S3 input bucket</strong></li>
            <li>S3 automatically triggers a <strong>Lambda function</strong></li>
            <li>Lambda processes the file (converts to uppercase, counts words)</li>
            <li>The result is saved to the <strong>S3 output bucket</strong></li>
            <li>The processed file is displayed above and available for viewing</li>
        </ol>
        <p><strong>All of this happens without any servers running 24/7.</strong> Lambda only runs when a file is uploaded.</p>
    </div>

    <script>
        const PRESIGN_URL = '<FUNCTION_URL>';
        const OUTPUT_BUCKET = '<OUTPUT_BUCKET>';
        const OUTPUT_REGION = 'us-east-1';

        async function uploadFile() {
            const content = document.getElementById('fileContent').value;
            const filename = document.getElementById('fileName').value;
            const statusEl = document.getElementById('status');
            const btn = document.getElementById('uploadBtn');

            if (!content.trim()) {
                statusEl.className = 'status error';
                statusEl.textContent = 'Please enter some text to upload.';
                return;
            }

            btn.disabled = true;
            statusEl.className = 'status loading';
            statusEl.textContent = 'Getting upload URL from Lambda...';

            try {
                const presignResponse = await fetch(PRESIGN_URL + '?filename=' + encodeURIComponent(filename));
                const presignData = await presignResponse.json();

                statusEl.textContent = 'Uploading file to S3...';

                const uploadResponse = await fetch(presignData.uploadUrl, {
                    method: 'PUT',
                    body: content,
                    headers: { 'Content-Type': 'text/plain' }
                });

                if (!uploadResponse.ok) throw new Error('Upload failed: ' + uploadResponse.status);

                statusEl.className = 'status loading';
                statusEl.textContent = 'File uploaded! Waiting for Lambda to process...';

                const outputUrl = 'https://' + OUTPUT_BUCKET + '.s3.' + OUTPUT_REGION + '.amazonaws.com/processed-' + filename;

                await new Promise(resolve => setTimeout(resolve, 5000));

                const resultResponse = await fetch(outputUrl);
                if (resultResponse.ok) {
                    const resultText = await resultResponse.text();
                    document.getElementById('result').textContent = resultText;
                    document.getElementById('downloadLink').href = outputUrl;
                    document.getElementById('resultSection').style.display = 'block';
                    statusEl.className = 'status success';
                    statusEl.textContent = 'File "' + filename + '" processed successfully! See the result below.';
                } else {
                    document.getElementById('downloadLink').href = outputUrl;
                    document.getElementById('result').textContent = 'Processing may still be in progress. Click the link below to check.';
                    document.getElementById('resultSection').style.display = 'block';
                    statusEl.className = 'status success';
                    statusEl.textContent = 'File uploaded! Processing may take a few more seconds. Try the link below.';
                }
            } catch (error) {
                statusEl.className = 'status error';
                statusEl.textContent = 'Error: ' + error.message;
            }

            btn.disabled = false;
        }
    </script>
</body>
</html>
```

**Upload the website.** 📋 Replace `<WEBSITE_BUCKET>`:

```
aws s3 cp index.html s3://<WEBSITE_BUCKET>/index.html --content-type "text/html" --region us-east-1
```

---

### Step 11: Test Your Application!

Your website URL is:

```
http://<WEBSITE_BUCKET>.s3-website-us-east-1.amazonaws.com
```

1. Open this URL in your browser
2. Type some text in the text area
3. Click **Upload & Process**
4. Wait 5 seconds — the processed result should appear on the page
5. Click **View Processed File** to see the raw output in a new tab

**✅ Checkpoint:** You should see your text converted to UPPERCASE with a word count header.

> **🎉 You just built a complete serverless web application!** A website that accepts user input, processes it with Lambda, and displays the result — all without a single server running 24/7.

---

## What You Just Did

You built a **full-stack serverless application** using 5 AWS services working together:

1. **S3 (Website Bucket)** — hosts your web page
2. **Lambda (Presign Function)** — generates secure upload URLs on demand
3. **S3 (Input Bucket)** — receives uploaded files
4. **Lambda (Processor Function)** — automatically processes files when they arrive
5. **S3 (Output Bucket)** — stores and serves processed results

This architecture is used in production by companies for image processing, document conversion, data pipelines, and more. The key insight: **no servers are running when no one is using the app.** You only pay for actual usage.

---

## Cert Prep Callout

**Target Certification:** AWS Solutions Architect – Associate (SAA)

This lab covers event-driven architectures, Lambda triggers, S3 event notifications, and presigned URLs — all heavily tested on the SAA exam.

**Sample question type:** "A company needs to automatically process images when users upload them through a web application. The solution should scale automatically and minimize operational overhead. Which architecture should they use?"

---

## Troubleshooting

| Issue | How to Fix It |
|-------|---------------|
| "Failed to fetch" when clicking Upload | The Function URL permission may not have propagated yet. Wait 2 minutes and try again. |
| CORS error in browser console | Make sure you ran the CORS configuration on both the input and output buckets (Step 8). |
| "Access-Control-Allow-Origin contains multiple values" | Your Lambda code is returning CORS headers AND the Function URL is adding them. Remove CORS headers from the Lambda code — the Function URL config handles it. |
| File uploads but no result appears | Check Lambda logs: Console → Lambda → workshop-processor → Monitor → View CloudWatch Logs. The processing Lambda may have errored. |
| 403 on the output file | Make sure you completed Step 9 (public access block disabled + bucket policy applied on the output bucket). |

---

## Cleanup

**⚠️ Important:** Clean up all resources.

### Step 1: Remove S3 Notification

📋 Replace `<INPUT_BUCKET>`:

**Windows:**
```powershell
aws s3api put-bucket-notification-configuration --bucket <INPUT_BUCKET> --notification-configuration "{}" --region us-east-1
```

**macOS / Linux:**
```bash
aws s3api put-bucket-notification-configuration --bucket <INPUT_BUCKET> --notification-configuration '{}' --region us-east-1
```

### Step 2: Delete Lambda Functions

```
aws lambda delete-function --function-name workshop-presign --region us-east-1
aws lambda delete-function --function-name workshop-processor --region us-east-1
```

### Step 3: Empty and Delete All Buckets

📋 Replace all bucket names:

```
aws s3 rm s3://<INPUT_BUCKET> --recursive
aws s3 rm s3://<OUTPUT_BUCKET> --recursive
aws s3 rm s3://<WEBSITE_BUCKET> --recursive
aws s3 rb s3://<INPUT_BUCKET>
aws s3 rb s3://<OUTPUT_BUCKET>
aws s3 rb s3://<WEBSITE_BUCKET>
```

### Step 4: Delete the IAM Role

```
aws iam detach-role-policy --role-name workshop-pipeline-role --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess
aws iam detach-role-policy --role-name workshop-pipeline-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name workshop-pipeline-role
```

### Step 5: Delete Local Files

**macOS / Linux:**

```bash
rm presign_function.py process_function.py presign.zip process.zip lambda-trust-policy.json s3-notification.json cors.json output-policy.json website-policy.json index.html
```

**Windows (PowerShell):**

```powershell
Remove-Item presign_function.py, process_function.py, presign.zip, process.zip, lambda-trust-policy.json, s3-notification.json, cors.json, output-policy.json, website-policy.json, index.html
```

### Step 6: Verify Cleanup

**✅ Checkpoint:**
1. **Lambda** → Functions → both functions are gone
2. **S3** → all three buckets are gone
3. **IAM** → Roles → `workshop-pipeline-role` is gone

---

## Help

If you get stuck, post your question in the **Lab Help** channel on Microsoft Teams. Include:
1. The **command** you ran
2. The **full error message**
3. Which **step number** you are on
