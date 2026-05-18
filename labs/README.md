# AI Cloud Fusion — Lab Guides

Welcome to the AI Cloud Fusion workshop lab guides. These hands-on labs accompany the weekly workshop sessions and are designed to be completed independently using only the instructions in each guide.

## Getting Started

**New here?** Start with [Lab 1A](session-01-cloud-concepts/lab-1a-aws-cli-setup.md) — it walks you through creating your AWS account and setting up everything you need from scratch. No prior experience required.

## How to Use These Labs

Each session has its own folder containing three labs at different difficulty levels:

| Level | What to Expect |
|-------|---------------|
| **Lab A (Beginner)** | Foundational setup or introductory concepts |
| **Lab B (Intermediate)** | Core hands-on task with multiple AWS services |
| **Lab C (Advanced)** | Real-world engineering task — you build something practical |

Work through the labs in order (A → B → C). Each lab includes:

- 📋 Copy-and-paste commands with clear instructions
- 🔄 Placeholders clearly marked so you know what to replace
- 💡 Annotations explaining what each service does and why
- ✅ Console checkpoints to verify your work visually
- 🧹 Cleanup/decommission steps so you don't incur any costs

## Session Labs

| Session | Topic | Labs |
|---------|-------|------|
| 1 | Cloud Concepts & AWS Global Infrastructure | [Lab 1A: AWS Account & CLI Setup](session-01-cloud-concepts/lab-1a-aws-cli-setup.md) · [Lab 1B: Cost Budget & SNS Alert](session-01-cloud-concepts/lab-1b-cost-budget-sns-alert.md) · [Lab 1C: S3 Static Website](session-01-cloud-concepts/lab-1c-s3-static-website.md) |
| 2 | Core AWS Services | [Lab 2A: Launch EC2 Instance](session-02-core-services/lab-2a-launch-ec2-instance.md) · [Lab 2B: EC2 + S3 with IAM Role](session-02-core-services/lab-2b-ec2-s3-iam-role.md) · [Lab 2C: Serverless File Pipeline](session-02-core-services/lab-2c-serverless-file-pipeline.md) |
| 3 | Cloud Security Fundamentals | [Lab 3A: IAM Least Privilege](session-03-cloud-security/lab-3a-iam-least-privilege.md) · [Lab 3B: Groups, Custom Policy & MFA](session-03-cloud-security/lab-3b-groups-custom-policy-mfa.md) · [Lab 3C: Roles & CloudTrail Audit](session-03-cloud-security/lab-3c-roles-cloudtrail-audit.md) |

*More sessions coming soon.*

## Prerequisites

- An email address, phone number, and credit/debit card (for AWS account creation)
- A computer running Windows, macOS, or Linux
- No prior AWS or cloud experience needed — Lab 1A covers everything from scratch

## Cost

All new AWS accounts created in **May 2026** will automatically receive **$100 USD** in AWS credits after identity verification. These credits apply to all AWS services and are intended to support students throughout the 3‑month program. Most labs in this program use Always Free AWS resources. Each lab includes a Cost Notice listing every service used and confirming the **estimated cost is $0.00**. Some labs (such as those involving EC2) may run resources that incur minimal cost, but these should not meaningfully impact the $100 credit balance as long as cleanup is completed correctly. **Always complete the Cleanup section** at the end of each lab to ensure no resources are left running. Proper cleanup prevents unnecessary charges and ensures the AWS credits last for the full duration of the program.

## Need Help?

If you get stuck on any lab, post your question in the **Lab Help** channel on Microsoft Teams. Include:
1. The **command** you ran (copy and paste it)
2. The **full error message** you received
3. Which **step number** you are on
