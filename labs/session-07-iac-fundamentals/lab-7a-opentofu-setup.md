# Lab 7A: Setting Up Your Infrastructure as Code Foundation

**Session:** 7 — Infrastructure as Code with OpenTofu  
**Track:** Solutions Architecture  
**Difficulty:** Beginner  
**Estimated Time:** 40–45 minutes  
**Target Cert:** AWS Solutions Architect – Associate (SAA)

---

## Overview

In this lab, you will set up a **complete Infrastructure as Code (IaC) environment** from scratch. By the end, you will have:

1. **OpenTofu installed** — the tool that turns text files into real cloud infrastructure
2. **A remote state backend** — an S3 bucket and DynamoDB table that safely store your infrastructure state
3. **A dedicated IAM role** — a separate identity that OpenTofu uses to create resources (not your admin credentials)
4. **A professional repo structure** — environments (dev/staging/prod), modules, scripts, and policies folders
5. **A working `tofu init → plan → apply → destroy` cycle** — proof that everything connects

**The scenario:** You are a new Solutions Architect starting at a company. Before you can deploy any infrastructure, you need to set up the tooling and repo structure that your team will use for all future deployments. This is the real first task of any IaC project — and you are going to do it properly from day one.

**Why this matters:** In previous labs, you created resources with individual CLI commands (`aws s3 mb`, `aws lambda create-function`). That works for learning, but in real jobs, infrastructure is defined in **code files** that are version-controlled, reviewed, and deployed consistently. IaC means your entire infrastructure can be recreated from a Git repository — no manual clicking, no forgotten steps, no "works on my machine."

---

## Prerequisites

- ✅ Completed **Lab 1A** (AWS CLI installed and configured)
- ✅ AWS CLI authenticated — run `aws sts get-caller-identity` and confirm it returns your account info
- ✅ A text editor for creating files

---

## Cost Notice

| Service | What It Is | Cost |
|---------|-----------|------|
| Amazon S3 | State file storage | Free within 5 GB (12-month free tier) |
| Amazon DynamoDB | State lock table | Free within 25 GB + 25 WCU/25 RCU (Always Free) |
| AWS IAM | Role for OpenTofu | Always Free |

**Estimated cost for this lab: $0.00**

> **⚠️ Note:** The state backend (S3 bucket + DynamoDB table) and the IAM role you create in this lab are **kept after the lab ends** — they are the foundation for Labs 7B and 7C. Only the "test" S3 bucket deployed in the final step is destroyed.

---

## Concepts

**Infrastructure as Code (IaC)** means defining your cloud resources in text files (code) rather than creating them manually through the console or CLI. The code describes *what* you want (a bucket, a function, a database), and the tool creates it for you.

**OpenTofu** is an open-source IaC tool. You write `.tf` files that declare what infrastructure you want, and OpenTofu creates, updates, and deletes AWS resources to match your declarations. It is a fork of HashiCorp Terraform — they use the same language (HCL) and are interchangeable for our purposes.

**Declarative vs. Imperative:**
- **Imperative** (what you have been doing): "Create a bucket. Now enable versioning. Now add a policy." — a sequence of commands.
- **Declarative** (IaC): "I want a bucket with versioning and this policy." — you describe the end state, and the tool figures out the steps.

**State file** is a JSON file that records what OpenTofu has created. It is how OpenTofu knows what already exists so it can calculate what needs to change. Without the state file, OpenTofu would not know your infrastructure exists.

**Remote state backend** means storing the state file in S3 (not on your laptop). This way multiple people can work on the same infrastructure, and you do not lose the state if your computer dies.

**State locking** prevents two people from modifying the same infrastructure at the same time. A DynamoDB table acts as the lock — when one person runs `tofu apply`, the lock is held until they finish.

**Assume role** means OpenTofu uses a dedicated IAM role (not your personal admin credentials) to create resources. This is a security best practice — it limits what OpenTofu can do and creates an audit trail.

---

## ⚠️ Reminder: How to Read Commands in This Lab

Commands are inside gray code boxes. **📋 Copy and paste** them into your terminal.

| Placeholder | What to Replace It With | Example |
|-------------|------------------------|---------|
| `<YOUR_PROFILE_NAME>` | Your AWS CLI profile name from Lab 1A | `AdministratorAccess-123456789012` |
| `<YOUR_ACCOUNT_ID>` | Your 12-digit AWS account number | `123456789012` |

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

**✅ You should see** your account ID and role.

> **📝 Write down your Account ID** (the 12-digit number). You will need it in several steps.

---

### Step 2: Install OpenTofu

OpenTofu is a single executable file — no installer needed. You download it, put it in a folder, and tell your system where to find it.

**Windows (PowerShell):**

📋 Copy and paste these commands one at a time:

```powershell
Invoke-WebRequest -Uri "https://github.com/opentofu/opentofu/releases/download/v1.9.1/tofu_1.9.1_windows_amd64.zip" -OutFile "$env:TEMP\tofu.zip"
```

```powershell
Expand-Archive -Path "$env:TEMP\tofu.zip" -DestinationPath "$env:TEMP\tofu" -Force
```

```powershell
New-Item -ItemType Directory -Path "C:\tofu" -Force
```

```powershell
Copy-Item "$env:TEMP\tofu\tofu.exe" "C:\tofu\tofu.exe" -Force
```

```powershell
[Environment]::SetEnvironmentVariable("PATH", [Environment]::GetEnvironmentVariable("PATH", "User") + ";C:\tofu", "User")
```

```powershell
$env:PATH += ";C:\tofu"
```

> **What do these commands do?**
> 1. Downloads the OpenTofu zip file
> 2. Extracts it to a temp folder
> 3. Creates a permanent folder at `C:\tofu`
> 4. Copies the executable there
> 5. Adds `C:\tofu` to your PATH permanently (so future terminals find it)
> 6. Adds it to the current terminal's PATH (so it works right now)

**macOS (Homebrew):**

📋 Copy and paste:

```bash
brew install opentofu
```

**macOS / Linux (manual download):**

```bash
curl -Lo /tmp/tofu.zip "https://github.com/opentofu/opentofu/releases/download/v1.9.1/tofu_1.9.1_$(uname -s | tr '[:upper:]' '[:lower:]')_amd64.zip"
sudo unzip -o /tmp/tofu.zip -d /usr/local/bin/ tofu
```

**Verify installation:**

📋 Copy and paste:

```
tofu --version
```

**✅ You should see:**

```
OpenTofu v1.9.1
on windows_amd64
```

(or `linux_amd64` / `darwin_amd64` on Mac/Linux)

> **💡 If `tofu` is not recognized:** Close your terminal and open a new one, then try `tofu --version` again. The PATH change only takes effect in new terminals.

---

### Step 3: Create Your Project Folder

**Windows (PowerShell):**

📋 Copy and paste:

```powershell
mkdir ~\Desktop\workshop-iac
cd ~\Desktop\workshop-iac
```

**macOS / Linux:**

📋 Copy and paste:

```bash
mkdir ~/Desktop/workshop-iac
cd ~/Desktop/workshop-iac
```

**Verify:** `pwd` should show a path ending in `workshop-iac`.

---

## PART 1 — Bootstrap the State Backend

Before OpenTofu can manage infrastructure, it needs a place to store its state file. This is a chicken-and-egg problem: you cannot use OpenTofu to create the bucket that stores its own state. So you bootstrap these two resources manually (once), then everything else is managed by OpenTofu.

### Step 4: Create the State Bucket

📋 Copy and paste, **replacing `<YOUR_ACCOUNT_ID>`**:

**Windows (PowerShell):**
```powershell
aws s3 mb s3://workshop-tofu-state-<YOUR_ACCOUNT_ID> --region us-east-1
```

**macOS / Linux:**
```bash
aws s3 mb s3://workshop-tofu-state-<YOUR_ACCOUNT_ID> --region us-east-1
```

**✅ You should see:** `make_bucket: workshop-tofu-state-<YOUR_ACCOUNT_ID>`

> **💡 Why include the account ID?** S3 bucket names are globally unique. Including your account ID ensures no one else has the same name.

**Enable versioning** (so you can recover previous state files if something goes wrong):

📋 Copy and paste, **replacing `<YOUR_ACCOUNT_ID>`**:

**Windows (PowerShell):**
```powershell
aws s3api put-bucket-versioning --bucket workshop-tofu-state-<YOUR_ACCOUNT_ID> --versioning-configuration Status=Enabled
```

**macOS / Linux:**
```bash
aws s3api put-bucket-versioning --bucket workshop-tofu-state-<YOUR_ACCOUNT_ID> --versioning-configuration Status=Enabled
```

**✅ No output means success.**

---

### Step 5: Create the State Lock Table

📋 Copy and paste:

```
aws dynamodb create-table --table-name terraform-locks --attribute-definitions AttributeName=LockID,AttributeType=S --key-schema AttributeName=LockID,KeyType=HASH --billing-mode PAY_PER_REQUEST --region us-east-1
```

**✅ You should see** JSON output with the table details. Look for `"TableStatus": "CREATING"`.

**Wait a few seconds**, then verify it is active:

```
aws dynamodb describe-table --table-name terraform-locks --query "Table.TableStatus" --output text --region us-east-1
```

**✅ You should see:** `ACTIVE`

> **What is this table for?** When you run `tofu apply`, OpenTofu writes a lock entry to this table. If someone else (or you in another terminal) tries to run `tofu apply` at the same time, they see the lock and wait. This prevents two people from changing the same infrastructure simultaneously.

---

## PART 2 — Create the OpenTofu Deploy Role

Instead of running OpenTofu with your full admin credentials, you will create a **dedicated IAM role** that OpenTofu assumes. This is a security best practice:
- It limits what OpenTofu can do (least privilege)
- It creates a clear audit trail in CloudTrail
- It mirrors how production CI/CD pipelines work

### Step 6: Create the Trust Policy

Open your text editor and create a **new, empty file**. 📋 Copy and paste this, **replacing `ACCOUNT_ID_HERE`** with your 12-digit account ID:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::ACCOUNT_ID_HERE:root"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
```

**Save the file as `tofu-trust-policy.json`** in your `workshop-iac` folder.

> **What does this file do?** It says "any identity in my AWS account can assume this role." Since you are the only person in your account, this effectively means "I can assume this role." In a team environment, you would restrict this further to specific users or CI/CD roles.

### Step 7: Create the Role

📋 Copy and paste:

```
aws iam create-role --role-name workshop-tofu-deploy-role --assume-role-policy-document file://tofu-trust-policy.json
```

**✅ You should see** JSON output with the role details.

### Step 8: Attach Permissions to the Role

The deploy role needs permission to manage the resources OpenTofu will create (S3 buckets in this lab) and to read/write the state lock table.

**Step 8a:** Open your text editor and create a **new, empty file**. 📋 Copy and paste this, **replacing `ACCOUNT_ID_HERE`** with your account ID:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "S3ManageWorkshopBuckets",
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": [
                "arn:aws:s3:::workshop-*",
                "arn:aws:s3:::workshop-*/*"
            ]
        },
        {
            "Sid": "DynamoDBStateLock",
            "Effect": "Allow",
            "Action": [
                "dynamodb:GetItem",
                "dynamodb:PutItem",
                "dynamodb:DeleteItem"
            ],
            "Resource": "arn:aws:dynamodb:us-east-1:ACCOUNT_ID_HERE:table/terraform-locks"
        }
    ]
}
```

**Replace `ACCOUNT_ID_HERE`** with your 12-digit account ID (1 place).

**Save the file as `tofu-permissions.json`** in your `workshop-iac` folder.

> **What does this file do?**
> - `S3ManageWorkshopBuckets` — OpenTofu can create, read, update, and delete any S3 bucket whose name starts with `workshop-`. This covers both the state bucket and any buckets you deploy.
> - `DynamoDBStateLock` — OpenTofu can read and write lock entries to the `terraform-locks` table (required for state locking).

**Step 8b:** Attach the policy to the role.

📋 Copy and paste:

```
aws iam put-role-policy --role-name workshop-tofu-deploy-role --policy-name tofu-deploy-permissions --policy-document file://tofu-permissions.json
```

**✅ No output means success.**

**⏳ Wait 10 seconds** for the role to propagate.

---

## PART 3 — Scaffold the IaC Repo Structure

Now you will create the folder structure that a professional IaC project uses. This structure separates **environments** (dev, staging, prod) from **reusable modules**, and includes helper scripts and a policy directory for future governance.

### Step 9: Create the Folder Structure

**Windows (PowerShell):**

📋 Copy and paste:

```powershell
mkdir infra\environments\dev
mkdir infra\environments\staging
mkdir infra\environments\prod
mkdir infra\modules
mkdir infra\scripts
mkdir infra\policies
```

**macOS / Linux:**

📋 Copy and paste:

```bash
mkdir -p infra/environments/dev
mkdir -p infra/environments/staging
mkdir -p infra/environments/prod
mkdir -p infra/modules
mkdir -p infra/scripts
mkdir -p infra/policies
```

**Your folder structure now looks like this:**

```
workshop-iac/
├── tofu-trust-policy.json     (from Step 6)
├── tofu-permissions.json      (from Step 8)
└── infra/
    ├── environments/
    │   ├── dev/               ← Development environment configs
    │   ├── staging/           ← Staging environment configs
    │   └── prod/              ← Production environment configs
    ├── modules/               ← Reusable infrastructure modules
    ├── scripts/               ← Helper scripts (init, plan, apply)
    └── policies/              ← Governance policies (future use)
```

> **What is each folder for?**
>
> | Folder | Purpose |
> |--------|---------|
> | `environments/dev` | Configuration for the development environment — this is where you experiment |
> | `environments/staging` | Configuration for staging — mirrors production for testing |
> | `environments/prod` | Configuration for production — real users, real data |
> | `modules/` | Reusable building blocks (e.g., "a Lambda function" module you use in all environments) |
> | `scripts/` | Shell scripts that wrap common commands (`init.sh`, `plan.sh`, `apply.sh`) |
> | `policies/` | Governance rules that prevent bad deployments (e.g., "no public S3 buckets") |
>
> **Why separate environments?** Each environment has its own state file, its own variables (different bucket names, different sizes), but shares the same modules. You develop in `dev`, test in `staging`, and deploy to `prod` with confidence because they use identical code.

---

### Step 10: Create the Dev Environment Configuration

You will now create four files in the `infra/environments/dev/` folder. These are the core of any OpenTofu environment.

**Step 10a: Create `backend.tf`** — tells OpenTofu where to store its state

Open your text editor and create a **new, empty file**. 📋 Copy and paste this, **replacing `ACCOUNT_ID_HERE`** with your account ID:

```hcl
terraform {
  backend "s3" {
    bucket         = "workshop-tofu-state-ACCOUNT_ID_HERE"
    key            = "dev/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

**Save the file as `backend.tf`** in `infra/environments/dev/`.

> **What does this file do?**
> - `bucket` — the S3 bucket you created in Step 4 (stores the state file)
> - `key` — the path within the bucket (`dev/` means each environment has its own state)
> - `dynamodb_table` — the lock table from Step 5
> - `encrypt` — the state file is encrypted at rest in S3

---

**Step 10b: Create `variables.tf`** — declares what inputs this configuration accepts

Open your text editor and create a **new, empty file**. 📋 Copy and paste:

```hcl
variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
  default     = "dev"
}

variable "aws_region" {
  description = "AWS region to deploy resources"
  type        = string
  default     = "us-east-1"
}

variable "project" {
  description = "Project name used for resource naming"
  type        = string
}

variable "tofu_role_arn" {
  description = "ARN of the IAM role OpenTofu assumes to manage resources"
  type        = string
}
```

**Save the file as `variables.tf`** in `infra/environments/dev/`.

> **What does this file do?** It declares the inputs (variables) that this configuration needs. Think of it like function parameters — you define what inputs are expected, and the actual values go in a separate file (`terraform.tfvars`).

---

**Step 10c: Create `terraform.tfvars`** — sets the actual values for variables

Open your text editor and create a **new, empty file**. 📋 Copy and paste this, **replacing `ACCOUNT_ID_HERE`** with your account ID:

```hcl
environment   = "dev"
aws_region    = "us-east-1"
project       = "workshop-iac"
tofu_role_arn = "arn:aws:iam::ACCOUNT_ID_HERE:role/workshop-tofu-deploy-role"
```

**Save the file as `terraform.tfvars`** in `infra/environments/dev/`.

> **What does this file do?** It provides the actual values for each variable. OpenTofu automatically reads this file when you run commands. Notice `tofu_role_arn` points to the role you created in Step 7 — this is how OpenTofu knows which role to assume.

---

**Step 10d: Create `main.tf`** — the provider configuration and your first resource

Open your text editor and create a **new, empty file**. 📋 Copy and paste:

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region

  assume_role {
    role_arn = var.tofu_role_arn
  }

  default_tags {
    tags = {
      Environment = var.environment
      Project     = var.project
      ManagedBy   = "opentofu"
    }
  }
}

# ============================================================
# Proof-of-life: a simple S3 bucket managed by OpenTofu
# ============================================================
data "aws_caller_identity" "current" {}

resource "aws_s3_bucket" "test" {
  bucket = "${var.project}-${var.environment}-test-${data.aws_caller_identity.current.account_id}"
}
```

**Save the file as `main.tf`** in `infra/environments/dev/`.

> **What does each section do?**
>
> | Section | Purpose |
> |---------|---------|
> | `terraform { required_version }` | Ensures you're running a compatible version of OpenTofu |
> | `required_providers` | Declares that we need the AWS provider (the plugin that knows how to talk to AWS) |
> | `provider "aws"` | Configures how OpenTofu connects to AWS — which region, which role to assume |
> | `assume_role { role_arn }` | OpenTofu will assume the deploy role instead of using your admin credentials directly |
> | `default_tags` | Every resource created gets these tags automatically — makes it easy to identify what OpenTofu manages |
> | `data "aws_caller_identity"` | Reads your account ID so we can use it in the bucket name |
> | `resource "aws_s3_bucket"` | Declares "I want an S3 bucket with this name" — OpenTofu will create it |

---

**Step 10e: Create `outputs.tf`** — defines what values to display after `tofu apply`

Open your text editor and create a **new, empty file**. 📋 Copy and paste:

```hcl
output "test_bucket_name" {
  description = "Name of the test S3 bucket created by OpenTofu"
  value       = aws_s3_bucket.test.bucket
}

output "test_bucket_arn" {
  description = "ARN of the test S3 bucket"
  value       = aws_s3_bucket.test.arn
}
```

**Save the file as `outputs.tf`** in `infra/environments/dev/`.

> **What does this file do?** After `tofu apply` finishes, it prints these values so you know what was created. Outputs are also how one module passes information to another (you will use this in Lab 7B).

---

**Your `infra/environments/dev/` folder should now contain:**

```
infra/environments/dev/
├── backend.tf         (where state is stored)
├── main.tf            (provider + resources)
├── outputs.tf         (what to display after apply)
├── terraform.tfvars   (actual variable values)
└── variables.tf       (variable declarations)
```

> **💡 This is the standard pattern.** Every environment (dev, staging, prod) has the same five files. The only differences between environments are in `backend.tf` (different state key) and `terraform.tfvars` (different values like names and sizes).

---

### Step 11: Create a `.gitignore`

In a real project, this repo would be version-controlled with Git. The `.gitignore` file tells Git which files to NOT track (local state, temporary files, sensitive data).

Open your text editor and create a **new, empty file**. 📋 Copy and paste:

```
# OpenTofu local files
.terraform/
*.tfstate
*.tfstate.backup
*.tfplan
.terraform.lock.hcl

# Sensitive variable overrides
*.auto.tfvars
override.tf
override.tf.json

# OS files
.DS_Store
Thumbs.db
```

**Save the file as `.gitignore`** in the root `workshop-iac/` folder (not inside `infra/`).

> **What does this file do?** It prevents you from accidentally committing state files (which may contain secrets), lock files, and OS junk to Git. The actual state lives safely in S3 — you never commit it to the repo.

---

## PART 4 — Initialize and Test

### Step 12: Initialize OpenTofu

This is the moment of truth. `tofu init` downloads the AWS provider plugin and connects to your S3 backend.

📋 Copy and paste:

**Windows (PowerShell):**
```powershell
cd infra\environments\dev
tofu init
```

**macOS / Linux:**
```bash
cd infra/environments/dev
tofu init
```

**✅ You should see:**

```
Initializing the backend...

Initializing provider plugins...
- Installing hashicorp/aws v5.x.x...
- Installed hashicorp/aws v5.x.x

OpenTofu has been successfully initialized!
```

> **🎉 This means:**
> - OpenTofu found your `backend.tf` and connected to the S3 bucket
> - It downloaded the AWS provider plugin (the code that knows how to create AWS resources)
> - The state lock table is ready
> - You can now run `plan` and `apply`

> **💡 If you get an error** about "Error configuring S3 Backend": double-check the bucket name in `backend.tf` matches the bucket you created in Step 4 (including your account ID).

---

### Step 13: Run `tofu plan` — Preview What Will Be Created

📋 Copy and paste:

```
tofu plan
```

**✅ You should see something like:**

```
data.aws_caller_identity.current: Reading...
data.aws_caller_identity.current: Read complete after 0s

OpenTofu will perform the following actions:

  # aws_s3_bucket.test will be created
  + resource "aws_s3_bucket" "test" {
      + bucket    = "workshop-iac-dev-test-123456789012"
      + tags_all  = {
          + "Environment" = "dev"
          + "ManagedBy"   = "opentofu"
          + "Project"     = "workshop-iac"
        }
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

> **What just happened?** OpenTofu:
> 1. Assumed your deploy role (you can see it working in the background)
> 2. Read the current state (empty — nothing exists yet)
> 3. Compared the state to your `main.tf` declarations
> 4. Calculated what needs to change: "1 resource needs to be created"
>
> **Nothing was created yet.** The plan is a preview. You review it, confirm it looks right, then apply.
>
> This is the core safety mechanism of IaC: **you always see what will happen before it happens.**

---

### Step 14: Run `tofu apply` — Create the Infrastructure

📋 Copy and paste:

```
tofu apply
```

OpenTofu will show the plan again and ask for confirmation. **Type `yes` and press Enter.**

**✅ You should see:**

```
aws_s3_bucket.test: Creating...
aws_s3_bucket.test: Creation complete after 1s

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

test_bucket_name = "workshop-iac-dev-test-123456789012"
test_bucket_arn = "arn:aws:s3:::workshop-iac-dev-test-123456789012"
```

> **🎉 You just created real AWS infrastructure from a text file!** The bucket exists, it has the correct tags, and OpenTofu's state file in S3 records that it was created.

**✅ Console Checkpoint:**

1. Go to [https://console.aws.amazon.com/](https://console.aws.amazon.com/)
2. Search for **S3** and click it
3. Find `workshop-iac-dev-test-<YOUR_ACCOUNT_ID>` in the list
4. Click on it → **Properties** tab → scroll to **Tags**
5. You should see: `Environment=dev`, `Project=workshop-iac`, `ManagedBy=opentofu`

> **💡 These tags came from `default_tags` in your provider configuration.** Every resource OpenTofu creates gets these automatically — no need to add them manually to each resource.

---

### Step 15: Run `tofu destroy` — Remove the Test Resource

The test bucket was just to prove everything works. Let's remove it cleanly.

📋 Copy and paste:

```
tofu destroy
```

OpenTofu will show what it plans to destroy. **Type `yes` and press Enter.**

**✅ You should see:**

```
aws_s3_bucket.test: Destroying...
aws_s3_bucket.test: Destruction complete after 0s

Destroy complete! Resources: 1 destroyed.
```

> **What just happened?** OpenTofu:
> 1. Read the state file (knew the bucket existed)
> 2. Calculated what needs to be removed (the bucket)
> 3. Deleted the bucket from AWS
> 4. Updated the state file (bucket no longer tracked)
>
> Go back to the S3 console and refresh — the test bucket is gone. **One command cleaned up everything.** Compare this to the manual cleanup steps in Labs 6A, 6B, and 6C — no more deleting versions, detaching policies, or forgetting a resource.

---

### Step 16: Remove the Test Resource from Your Code

Since you destroyed the test bucket, remove it from `main.tf` so your code matches reality.

Open `infra/environments/dev/main.tf` in your text editor. **Delete** the last two blocks (the `data "aws_caller_identity"` and `resource "aws_s3_bucket"` blocks).

Your `main.tf` should now look like this:

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region

  assume_role {
    role_arn = var.tofu_role_arn
  }

  default_tags {
    tags = {
      Environment = var.environment
      Project     = var.project
      ManagedBy   = "opentofu"
    }
  }
}
```

Also open `outputs.tf` and **delete all content** (or delete the file). You will add real outputs in Lab 7B.

**Save both files.**

> **💡 This is the IaC discipline:** code and infrastructure stay in sync. If a resource is destroyed, the code is updated to match. The repo always represents the truth of what is deployed.

---

## What You Just Did

You set up a **complete Infrastructure as Code foundation** — the same setup a real engineering team uses before deploying their first line of infrastructure:

| What You Built | Why It Matters |
|---------------|----------------|
| Installed OpenTofu | The tool that converts text files into real cloud resources |
| S3 state bucket (versioned) | Safely stores what OpenTofu has created — shared, durable, recoverable |
| DynamoDB lock table | Prevents two people from modifying infrastructure at the same time |
| Dedicated deploy IAM role | Separation of duties — OpenTofu has limited, auditable permissions |
| Professional repo structure | Dev/staging/prod environments, modules, scripts — ready to scale |
| Working init → plan → apply → destroy cycle | The core IaC workflow, proven end to end |

**Key takeaways:**
- **IaC is declarative** — you describe what you want, not how to get there. OpenTofu figures out the steps.
- **Plan before apply** — you always see what will change before it happens. No surprises.
- **State is sacred** — the state file is the source of truth for what exists. It lives in S3, versioned, encrypted, and locked.
- **Environments share code** — dev, staging, and prod use the same modules with different variables. Change once, deploy consistently.
- **Destroy is one command** — no more manual cleanup with 15 delete commands. OpenTofu tracks everything.

> **💡 What persists after this lab:** The state bucket, lock table, IAM role, and your repo scaffolding all stay in place. Labs 7B and 7C will deploy real resources into this same structure. Do NOT delete these bootstrap resources.

---

## Cert Prep Callout

**Target Certification:** AWS Solutions Architect – Associate (SAA)

The SAA exam tests:
- Understanding of Infrastructure as Code concepts and benefits
- S3 bucket versioning (your state bucket uses it)
- DynamoDB table configuration (on-demand billing mode)
- IAM roles and the principle of least privilege
- The relationship between state management and infrastructure consistency

**Sample question type:** "A team uses OpenTofu to manage infrastructure across multiple environments. How should they configure state storage to prevent conflicts?"  
**Answer:** Use an S3 backend with versioning enabled and a DynamoDB table for state locking. Each environment should use a separate state key within the same bucket.

---

## Troubleshooting

| Issue | What It Means | How to Fix It |
|-------|--------------|---------------|
| `tofu` is not recognized | OpenTofu is not in your PATH | Close and reopen your terminal. On Windows, verify `C:\tofu\tofu.exe` exists. |
| `Error configuring S3 Backend` | The state bucket name doesn't match | Open `backend.tf` and verify the bucket name matches exactly what you created in Step 4 |
| `Cannot assume IAM Role` | Trust policy or credentials issue | Verify your AWS session is active (`aws sts get-caller-identity`). If expired, re-login with SSO. |
| `Error acquiring the state lock` | A previous command was interrupted and left a stale lock | Run `tofu force-unlock <LOCK_ID>` using the lock ID shown in the error message |
| `Error creating S3 Bucket: AccessDenied` | The deploy role's permissions don't cover the bucket name | Verify `tofu-permissions.json` has `"Resource": ["arn:aws:s3:::workshop-*", ...]` and your bucket name starts with `workshop-` |
| `BucketAlreadyExists` when creating the state bucket | Someone else has that name | Add extra characters to the bucket name (e.g., append `-01`) and update `backend.tf` to match |
| Plan shows no changes but you expected a resource | The resource code may not be saved | Make sure `main.tf` is saved, and you're in the right directory (`infra/environments/dev/`) |

---

## Cleanup

> **⚠️ IMPORTANT: Do NOT delete the state bucket, DynamoDB table, or IAM role!** These are used in Labs 7B and 7C. Only clean up if you are NOT continuing with the rest of Session 7.

### If you ARE continuing to Lab 7B/7C (recommended):

**No cleanup needed.** Keep everything in place:
- ✅ State bucket stays (`workshop-tofu-state-<YOUR_ACCOUNT_ID>`)
- ✅ Lock table stays (`terraform-locks`)
- ✅ Deploy role stays (`workshop-tofu-deploy-role`)
- ✅ Your repo scaffolding stays (`workshop-iac/`)
- The test S3 bucket was already destroyed in Step 15

### If you are DONE with Session 7 and want to clean up everything:

**Step 1: Delete the state bucket**

📋 Copy and paste, **replacing `<YOUR_ACCOUNT_ID>`**:

**Windows (PowerShell):**
```powershell
aws s3 rm s3://workshop-tofu-state-<YOUR_ACCOUNT_ID> --recursive
```

Then delete all versions (the bucket has versioning enabled):

```powershell
$versions = aws s3api list-object-versions --bucket workshop-tofu-state-<YOUR_ACCOUNT_ID> --output json | ConvertFrom-Json
foreach ($v in $versions.Versions) { aws s3api delete-object --bucket workshop-tofu-state-<YOUR_ACCOUNT_ID> --key $v.Key --version-id $v.VersionId | Out-Null }
foreach ($m in $versions.DeleteMarkers) { aws s3api delete-object --bucket workshop-tofu-state-<YOUR_ACCOUNT_ID> --key $m.Key --version-id $m.VersionId | Out-Null }
aws s3 rb s3://workshop-tofu-state-<YOUR_ACCOUNT_ID>
```

**macOS / Linux:**
```bash
aws s3 rm s3://workshop-tofu-state-<YOUR_ACCOUNT_ID> --recursive
aws s3api delete-objects --bucket workshop-tofu-state-<YOUR_ACCOUNT_ID> --delete "$(aws s3api list-object-versions --bucket workshop-tofu-state-<YOUR_ACCOUNT_ID> --query '{Objects: Versions[].{Key:Key,VersionId:VersionId}}' --output json)"
aws s3api delete-objects --bucket workshop-tofu-state-<YOUR_ACCOUNT_ID> --delete "$(aws s3api list-object-versions --bucket workshop-tofu-state-<YOUR_ACCOUNT_ID> --query '{Objects: DeleteMarkers[].{Key:Key,VersionId:VersionId}}' --output json)"
aws s3 rb s3://workshop-tofu-state-<YOUR_ACCOUNT_ID>
```

**Step 2: Delete the DynamoDB lock table**

```
aws dynamodb delete-table --table-name terraform-locks --region us-east-1
```

**Step 3: Delete the IAM role**

```
aws iam delete-role-policy --role-name workshop-tofu-deploy-role --policy-name tofu-deploy-permissions
```

```
aws iam delete-role --role-name workshop-tofu-deploy-role
```

**Step 4: Delete local files**

**Windows (PowerShell):**
```powershell
cd ~\Desktop
Remove-Item -Recurse -Force ~\Desktop\workshop-iac
```

**macOS / Linux:**
```bash
cd ~/Desktop
rm -rf ~/Desktop/workshop-iac
```

---

## Help

If you get stuck, post your question in the **Lab Help** channel on Microsoft Teams. Include:
1. The **step number** you are on
2. The **command** you ran (copy and paste it)
3. The **full error message** you received (copy and paste it)
4. Your **operating system** (Windows, Mac, or Linux)
