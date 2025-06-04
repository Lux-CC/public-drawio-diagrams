# Architecture Design Recommendation: Terraform over AWS CDK  
_For teams with less than 3 years experience on AWS_

---
## 📚 Table of Contents

- [🌐 What is Infrastructure as Code and why do you need it?](#-what-is-infrastructure-as-code-and-why-do-you-need-it)
- [⚖️ Why We Recommend Terraform (Over AWS CDK)](#️-why-we-recommend-terraform-over-aws-cdk)
  - [1. 📉 Learning Curve & Operational Risk](#1--learning-curve--operational-risk)
  - [2. 🔧 Easier Operations & Handling Manual Changes](#2--easier-operations--handling-manual-changes)
  - [3. 🚢 Cleaner Container Deployment Flows](#3--cleaner-container-deployment-flows-eg-ecs-on-fargate)
  - [4. 🚪 Flexible Foundation with Future Options](#4--flexible-foundation-with-future-options)
- [✅ Summary Table](#-summary-table)
- [🛠️ Approaches for Serverless Workloads](#️-approaches-for-serverless-workloads)
- [📎 Addendum](#-addendum)
  - [Addendum A: ⚠️ Downsides of Terraform (and How to Work Around Them)](#addendum-a-️-downsides-of-terraform-and-how-to-work-around-them)
  - [Addendum B: Example of an extremely simple CDK & Terraform app](#addendum-b-example-of-an-extremely-simple-cdk--terraform-app)
    - [🧰 Option 1: CDK (TypeScript)](#-option-1-cdk-typescript)
    - [🧰 Option 2: Terraform](#-option-2-terraform)
    - [🔍 Summary: Plan / Diff Experience](#-summary-plan--diff-experience)
    - [🧩 Takeaway](#-takeaway)

## 🌐 What is Infrastructure as Code and why do you need it?

Infrastructure as Code (IaC) means managing your cloud infrastructure (servers, networks, databases, etc.) using code instead of manual steps in the AWS console. For developers coming from backend, frontend, or PHP backgrounds, it brings benefits similar to using version control and package managers for application code:

- 🕐 **Rebuild entire environments from scratch in under an hour** rather than spending days manually configuring everything.
- 🧪 **Create exact copies of your infrastructure across dev, staging, and prod**, eliminating environment drift.
- ✅ **Review and test changes before applying**, enabling safer deployments.
- 📚 **Track infrastructure history in Git**, making it easy to audit or revert changes.

The most widely used tools for Infrastructure as Code on AWS are:

- **Terraform** – an open-source, cloud-agnostic tool by HashiCorp
- **CloudFormation** – AWS's native IaC service. Many tools, including CDK, ultimately compile down to CloudFormation templates, which are what actually get deployed to your account.


Here’s how you would create a simple S3 bucket using both:

### 🧾 Terraform

```hcl
resource "aws_s3_bucket" "example_bucket" {
  bucket = "my-simple-bucket"
}
```

### 🧾 CloudFormation (YAML)

```yaml
Resources:
  ExampleBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: my-simple-bucket
```

As you can see, **both can define the same resource**: an S3 bucket. You can deploy these templates to a dev AWS account to test, and once you're confident, reuse the exact same template to deploy to production.


---

## ⚖️ Why We Recommend Terraform (Over AWS CDK)

When choosing a foundation for infrastructure automation, we evaluated **Terraform** and **AWS CDK**. While CDK allows writing infrastructure in languages like TypeScript or Python, we recommend **Terraform** for teams new to AWS, especially when containerized workloads are used for web applications.

---

### 1. 📉 Learning Curve & Operational Risk

| Factor              | Terraform                                                   | AWS CDK                                                                 |
|---------------------|-------------------------------------------------------------|--------------------------------------------------------------------------|
| Setup effort        | Easy to get started and reason about                        | Fast to scaffold projects, but setup depends on language/tooling choices |
| Required knowledge  | Basic AWS services + HCL (HashiCorp Configuration Language) | CloudFormation, CDK concepts, and full TypeScript or Python ecosystem knowledge (including build tooling, dependency management, etc.) |
| Transparency        | Declarative and explicit (you see exactly what's created)   | Abstracted away — requires debugging through synthesized CloudFormation |

- CDK **generates CloudFormation** templates behind the scenes, which have trickier change management.
- To safely troubleshoot or maintain CDK infrastructure, **you must understand CloudFormation deeply** — which adds significant risk for new teams. Treating CDK like a black box often leads to unexpected outages and long time to recovery after a failure occurs.
- Terraform is **more explicit and predictable**, making onboarding and troubleshooting much easier.

---

### 2. 🔧 Easier Operations & Handling Manual Changes

Manual changes to cloud infrastructure happen outside of the IaC templates — especially in early stages or mixed-skill teams.

- **Terraform detects these changes** and helps you reconcile them cleanly using `terraform plan`, `terraform refresh`, or `terraform import`.
- You can also mark specific fields to **ignore changes**, giving you flexibility to handle temporary manual fixes or external automation.

> Example: If someone manually updates a security group or IAM role, Terraform will detect it and guide you through syncing it safely.

- **CDK**, by contrast, is fragile when resources are changed outside of its control — often causing errors or requiring destructive redeploys.

---

### 3. 🚢 Cleaner Container Deployment Flows (e.g. ECS on Fargate)

When deploying containerized apps, you often want to:

- Update the **container image** (e.g. via CI/CD)
- Leave the **infrastructure** unchanged

With **Terraform**, you can define:

```hcl
lifecycle {
  ignore_changes = ["image"]
}
```

This lets you **push a new container version without triggering a full infra redeploy**.

In **CDK**, there's no equivalent out of the box. You need to:

- Build the container
- Push it to AWS' elastic container registry (ECR)
- Update the CDK stack with the new image reference
- Deploy the whole infrastructure stack again

This takes more time, adds complexity, and increases the risk of unrelated infra changes being deployed accidentally.

---

### 4. 🚪 Flexible Foundation with Future Options

Once you master Terraform:

- You can **always add CDK** later for standalone, isolated use cases (e.g. full-stack teams experimenting with Lambda). More on this later though.
- But the reverse is harder — CDK makes it difficult to extract or refactor later due to tight coupling with CloudFormation.

Terraform gives you a **strong baseline** that’s modular, scalable, and compatible with a wide range of tools.

---

## ✅ Summary Table

| Criteria                | Terraform                         | AWS CDK                                     |
|-------------------------|-----------------------------------|---------------------------------------------|
| 🧠 Learning curve       | Easier for teams new to AWS       | Fast start, but high long-term risk         |
| 🔍 Debugging visibility | Clear and consistent              | Opaque, relies on CloudFormation knowledge  |
| 🔄 Manual change safety | Supports drift recovery & imports | Fragile; fails on manual infra edits        |
| 🚢 Container workflows  | Can decouple image updates        | Requires full-stack redeploy                |
| 🛠 Long-term flexibility| CDK can be layered on later       | Hard to extract once adopted                |

---





If your project is:

- **Infra-heavy** (ECS, RDS, networking, etc.) → stick with **Terraform**
- **Purely serverless** → consider **Serverless Framework** or **CDK**
- **A mix** → use Terraform for infra, and Serverless/SAM/CDK for isolated serverless stacks

The goal is to avoid drift and maintain clarity, while also letting your team move fast.


# 📎 Addendum

## Addemdum A: ⚠️ Downsides of Terraform (and How to Work Around Them)

While Terraform offers strong stability, transparency, and control, it does come with trade-offs:

### 📄 Verbosity — Especially for Serverless Workloads

Terraform can be **verbose**, particularly when working with serverless components like:

- Lambda functions
- IAM Policies
- API Gateway
- Step Functions
- EventBridge rules

These setups often require multiple interconnected resources with low-level wiring, making the Terraform code **longer and harder to read** than equivalent CDK or Serverless Framework configurations.

For example, deploying a simple Lambda+API in Terraform often requires managing IAM roles, policies, method integrations, permissions, and deployment stages manually.

> Personal note from Lux Cloud Consulting: I personally still use CDK for almost all my personal projects — I'm much faster with it. But I also have **7 years of experience** (and failures) with both CDK and CloudFormation, which means:
> - I know how to safely override CDK behavior.
> - I can read and patch raw CloudFormation.
> - I even understand the AWS API calls made by CloudFormation under the hood, so I know which fields are dangerous and which aren't.
> This makes **correcting and deploying infrastructure configurations much less risky for me** than for teams newer to AWS.


### 🛠️ Approaches for Serverless Workloads

Terraform can feel **verbose and low-level** when building serverless applications like Lambda functions, API Gateway, or Step Functions — especially compared to tools that abstract these patterns.

If a part of your project is **serverless-heavy** and does not have strong coupling to non-serverless components, consider these more developer-friendly options:

- Use **Terraform** for foundational infrastructure (e.g. VPCs, ECS, databases, networking).
- Use a faster framework for serverless components, such as:
  - [**Serverless Framework**](https://www.serverless.com/) – intuitive, widely used, great plugin ecosystem
  - [**AWS SAM**](https://docs.aws.amazon.com/serverless-application-model/) – AWS-native, deploys quickly via `sam deploy`

These frameworks are optimized for typical serverless use cases, allowing you to spin up complete Lambda + API Gateway stacks in **under 30 lines of YAML**.

The **Terraform ecosystem is improving** too, thanks to projects like:

- [`serverless.tf`](https://github.com/antonbabenko/serverless.tf) – opinionated wrappers to reduce boilerplate
- [`terraform-aws-modules`](https://github.com/terraform-aws-modules/) – well-maintained modules with sane defaults

> ✅ **Hybrid recommendation:** Use Terraform where it shines (networking, persistence, containers), and plug in serverless-specific tooling where it makes your team more productive.

```bash
# Example: deploy core infra with Terraform
terraform apply

# Then deploy independent serverless logic with Serverless Framework
sls deploy
```

This approach keeps your infrastructure modular, minimizes drift, and lets your team move fast where it matters most.

---

### ✅ Recommendation


## Addendum B: Example of an extremely simple CDK & Terraform app

Note this is not a realistic scenario, but it can give you an idea of how a CDK app compares to a terraform template.

This example creates an **S3 bucket** named `my-log-bucket` with:

- Versioning enabled  
- Block public access  
- Tag: `Environment = Dev`

---

### 🧰 Option 1: CDK (TypeScript)

#### Code

```ts
import * as cdk from 'aws-cdk-lib';
import { Bucket, BlockPublicAccess } from 'aws-cdk-lib/aws-s3';

const app = new cdk.App();
new cdk.Stack(app, 'LogBucketStack', {
  env: { region: 'us-east-1' },
}).node.addConstruct(new Bucket(this, 'MyLogBucket', {
  bucketName: 'my-log-bucket',
  versioned: true,
  blockPublicAccess: BlockPublicAccess.BLOCK_ALL,
  removalPolicy: cdk.RemovalPolicy.RETAIN,
  tags: { Environment: 'Dev' },
}));
```

#### DevOps Commands

```bash
npm install
cdk bootstrap
cdk diff      # Shows planned changes
cdk deploy    # Deploys via CloudFormation
```

#### `cdk diff` Output (Simplified)

```diff
Resources
[+] AWS::S3::Bucket MyLogBucket MyLogBucketXXXXX
[+] AWS::S3::BucketPolicy MyLogBucket/Policy PolicyXXXXX
```

> CDK does **not show field-level changes** clearly unless you manually dig into synthesized CloudFormation or logs. Drift is hard to detect from this diff alone.

---

### 🧰 Option 2: Terraform

#### Code

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "log_bucket" {
  bucket = "my-log-bucket"

  versioning {
    enabled = true
  }

  tags = {
    Environment = "Dev"
  }

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_public_access_block" "log_bucket_block" {
  bucket                  = aws_s3_bucket.log_bucket.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

#### DevOps Commands

```bash
terraform init
terraform plan    # Shows proposed changes
terraform apply   # Deploys changes
```

#### `terraform plan` Output (Simplified)

```
Terraform will perform the following actions:

  # aws_s3_bucket.log_bucket will be created
  + resource "aws_s3_bucket" "log_bucket" {
      + bucket = "my-log-bucket"
      + versioning {
          + enabled = true
        }
      + tags = {
          + "Environment" = "Dev"
        }
    }

  # aws_s3_bucket_public_access_block.log_bucket_block will be created
  + resource "aws_s3_bucket_public_access_block" "log_bucket_block" {
      + block_public_acls       = true
      + block_public_policy     = true
      + ignore_public_acls      = true
      + restrict_public_buckets = true
    }

Plan: 2 to add, 0 to change, 0 to destroy.
```

---

### 🔍 Summary: Plan / Diff Experience

| Aspect                 | CDK                             | Terraform                             |
|------------------------|----------------------------------|----------------------------------------|
| Plan/diff detail       | Basic resource-level summary     | Field-level, explicit diff             |
| Drift detection        | Often only visible via errors    | First-class via `terraform plan`      |
| Output clarity         | Hidden behind CDK abstractions   | Clear and readable                    |
| Day-to-day usability   | Requires CloudFormation knowledge| Works with minimal AWS internals      |

---

### 🧩 Takeaway

When you're building and operating infrastructure:

- **Terraform gives developers full control and visibility** into what’s changing — critical for confidence, especially on real environments.
- **CDK adds convenience for building**, but at the cost of **visibility, safety, and day-to-day ops debugging** — especially when changes happen outside the IaC (e.g. manual tweaks).

For this reason, we recommend **Terraform** as the default infrastructure-as-code tool for teams starting out on AWS.
