# Choosing a Terraform Deployment Flow – From Dev to Prod

*For backend developers getting started with AWS*

---

## 🌟 Introduction: Why Deployment Strategy Matters (Especially with IaC)

As a backend developer using Terraform, you're not just defining infrastructure — you're deploying it safely and consistently.

Your **deployment flow** (how you move changes from dev to prod) directly impacts:

* Speed of iteration
* Team collaboration
* Risk of production mistakes

This guide outlines two realistic Terraform deployment flows, how to structure environments, and how to align your Git strategy with infrastructure rollout.

---

## 🚀 Scenario 1: Minimum Setup – One Environment Per Account

### Description

Start with a basic deployment flow that doesn't rely on staging names at all. Each AWS account represents a clean environment (dev, acc, prod), and all infrastructure is deployed with static names (e.g., `my-app-bucket`). This setup keeps things very simple for teams just starting out.

* Use **separate AWS accounts** for `dev`, `acc`, and `prd`
* Apply the **same Terraform code** with different variable sets (`.tfvars`) or backends

### Git Branch Strategy

```
feature/* ➔ main
```

* `feature/*` → can be deployed **manually from a developer's laptop** or via **CI/CD pipeline**, targeting the **AWS dev account** for fast iteration
* `main`:

  * Deployed exclusively by CI/CD — begins with the **AWS dev account** to perform a basic smoke test and verify that the merge to `main` hasn’t unintentionally broken anything
  * Then promotes to:

    * **AWS acc account** (for in-depth functional testing)
    * **AWS prod account**

### Benefits

* ✅ Simple to understand and implement
* ✅ No naming collisions or state overlap
* ✅ Easier rollback between clean environments

### Drawbacks

* ❌ Only one shared `dev` environment → potential blocking
* ❌ No parallel testing or previews per PR
* ❌ Infra drift more likely if manual edits are made

---

## 🧪 Scenario 2: Advanced Setup – Stage-Based Parallel Deployments

As your team scales and multiple developers need to test infrastructure in parallel, you’ll want to introduce the concept of `stageName` — a suffix or identifier that distinguishes resource instances per feature branch or environment. This makes it possible to deploy **multiple copies of the same stack** within a single AWS account.

### Description

* Use **single AWS dev account**, but deploy **multiple isolated environments** in parallel via `stageName`
* Still use separate accounts for `acc` and `prd`

### Git Branch Strategy

```
feature/* ➔ main
```

* `feature/*` → deploys to **AWS dev account**, using `stageName = feature-X`
* `main` branch:

  * Deploys to **AWS dev account**, using `stageName = dev`
  * Then promotes to:

    * **AWS acc account** with `stageName = acc`
    * **AWS prod account** with `stageName = prd`

### Requirements

* Parameterized resource names: `my-bucket-${stageName}`
* Isolated state per feature or CI environment
* Auto-cleanup pipelines or scheduled teardown when feature stages have finished
* Optional: shared or seeded data stores for feature environments

### Benefits

* ✅ Parallel testing (PR previews)
* ✅ Fast feedback on infra changes
* ✅ No need for more AWS accounts

### Drawbacks

* ❌ Naming complexity and resource limits in a single account
* ❌ Requires stronger CI discipline (e.g. destroy feature stacks)
* ❌ More complex state management

---

## 💡 Hybrid Recommendation (Best of Both Worlds)

Start simple, but structure your code with flexibility:

* Add a `stageName` variable from day one
* Avoid hardcoded resource names
* Structure `dev`, `acc`, and `prd` with the same module inputs, but different values

```hcl
resource "aws_s3_bucket" "app" {
  bucket = "my-app-${var.stageName}"
}
```

This allows you to:

* Start with 1 deploy per account
* Add preview stages later without refactoring
* Prepare for more advanced patterns like cell-based deployments

---

## 📂 Git + Terraform Best Practices

| Tip                                   | Description                           |
| ------------------------------------- | ------------------------------------- |
| Keep `*.tfvars` files in Git          | Consistent environment configuration  |
| Use backend configs per env           | Avoids state overlap                  |
| Add `stageName` to all resource names | Enables parallel deployments          |
| Automate `terraform plan` for PRs     | Catch issues before they reach `main` |
| Use CI/CD to run `terraform apply`    | Avoids manual deployment risk         |
| Add cleanup jobs for feature stages   | Prevent AWS quota exhaustion          |

---

## 🧩 Conclusion

* Start with the **minimum setup**: 1 environment per AWS account
* Add a `stageName` early so you can **scale your workflow later**
* Align Git flow with your Terraform rollout:

  * `feature/*` branches = isolated changes
  * `main` or `dev` branches = promotion pipelines

Once your team grows or needs faster iteration, you can evolve to parallel deployments or even dynamic stage previews with little code refactoring.

Terraform scales well — **if you set the foundation right.**

---

## 📦 Addendum: Going Further with CI/CD Builds

As your Terraform deployment matures, you may want to include more advanced checks and build steps in your CI/CD pipelines:

### 🔍 `terraform plan` and `plan diff` for Pull Requests

* Run `terraform plan` for each PR against the dev workspace
* Upload the plan output as an artifact or comment it back into the PR
* Use tools like [infracost](https://www.infracost.io/) to highlight cost changes

### ✅ Validate and Lint

* Use `terraform validate` to check syntax and config errors
* Use `tflint` or `checkov` for static analysis and security checks

### 🔬 Automated Tests (Optional)

* Use `terratest` (Go) or `kitchen-terraform` (Ruby) to write end-to-end tests
* For example: provision infra → hit an API endpoint → verify status code

### 🔐 Secret Management

* Never store secrets in `.tfvars` or `terraform.tfstate`
* Use systems like AWS SSM Parameter Store, Secrets Manager, or Vault

### 🧪 Mocking Shared Infrastructure in Dev

* Use mocks or pre-provision shared resources like RDS, S3, and VPC
* Or treat shared resources as separate modules with data source lookups

### 📊 Observability

* Export Terraform outputs to your APM or dashboard system (e.g., Datadog tags, Slack alerts)
* Track changes over time and who triggered them
