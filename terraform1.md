# Terraform Deep-Dive Interview Questions & Answers

This README is designed to test **real Terraform knowledge**, especially for Mid/Senior DevOps engineers. The questions focus on reasoning, troubleshooting, state management, production safety, modules, lifecycle, providers, security, and CI/CD rather than memorized definitions.

---

## 1. Terraform State

### Q1. Terraform wants to destroy and recreate a production resource even though you did not change the configuration. How do you investigate?

**Answer:**

Start with the plan and identify exactly which attribute forces replacement.

```bash
terraform plan
terraform state show <resource>
```

Then check:

1. Terraform configuration.
2. Current state.
3. Actual cloud resource.
4. Provider version.
5. Recent provider upgrades.
6. Drift/manual changes.
7. Computed/default attributes.
8. Resource schema changes.
9. Whether the resource address changed.
10. Whether lifecycle rules or dependencies are involved.

Useful commands:

```bash
terraform show
terraform state show <address>
terraform providers
terraform plan
terraform plan -refresh-only
```

Do **not** blindly use `-replace` or manually modify state. First determine why replacement is required.

---

### Q2. What happens if two engineers run `terraform apply` simultaneously against the same remote state?

**Answer:**

Terraform uses **state locking** when the backend supports it.

Normally:

- Run A acquires the lock.
- Run B waits or fails depending on backend/configuration.
- This prevents concurrent state modifications.

Without proper locking, concurrent applies can corrupt or overwrite state and cause conflicting infrastructure changes.

In an enterprise environment, remote state + locking is essential.

---

### Q3. The state file is deleted, but the infrastructure still exists. How do you recover?

**Answer:**

First check whether the backend has:

- State versioning.
- State history.
- Backups.
- Previous versions.

If the state cannot be recovered, Terraform can be rebuilt by importing existing resources:

```bash
terraform import <resource_address> <resource_id>
```

For many resources, automate the import process rather than manually rebuilding everything.

The important point: **do not recreate production infrastructure just because state is missing.**

---

### Q4. Explain these commands:

```bash
terraform state rm
terraform import
terraform apply -refresh-only
terraform state mv
```

**Answer:**

| Command | Purpose |
|---|---|
| `state rm` | Removes a resource from Terraform state without destroying the real resource |
| `import` | Adds an existing real resource into Terraform state |
| `apply -refresh-only` | Updates state to reflect real infrastructure without changing infrastructure |
| `state mv` | Changes the Terraform state address of a resource |

Example:

```bash
terraform state mv   module.old.aws_instance.app   module.new.aws_instance.app
```

This is useful when reorganizing modules without recreating resources.

---

### Q5. A resource exists in AWS but is not in Terraform state. What happens during `terraform apply`?

**Answer:**

Terraform considers the resource unmanaged.

If the configuration declares that resource, Terraform will normally attempt to create it.

If the resource already exists, the provider may return an "already exists" error.

The correct solution is generally to import the existing resource.

---

### Q6. A resource moves from one module to another and Terraform wants to destroy/recreate it. How do you prevent that?

**Answer:**

Use a state move or a `moved` block.

Example:

```hcl
moved {
  from = module.network.aws_subnet.app
  to   = module.infrastructure.aws_subnet.app
}
```

This tells Terraform that it is the **same resource at a new address**.

---

# 2. Dependency Graph

### Q7. How does Terraform know resource creation order?

**Answer:**

Terraform builds a dependency graph.

For example:

```hcl
resource "aws_instance" "app" {
  subnet_id = aws_subnet.app.id
}
```

The reference:

```hcl
aws_subnet.app.id
```

creates an **implicit dependency**.

Terraform therefore knows:

```text
VPC -> Subnet -> Instance
```

Resources without dependencies can be created in parallel.

---

### Q8. Difference between implicit dependency and `depends_on`?

**Answer:**

Implicit dependency:

```hcl
subnet_id = aws_subnet.app.id
```

Terraform automatically understands the dependency.

Explicit dependency:

```hcl
depends_on = [
  aws_vpc.main
]
```

Use `depends_on` when the dependency exists logically but is not visible through a resource reference.

---

### Q9. When can `depends_on` make Terraform worse?

**Answer:**

Overusing it can make the dependency graph unnecessarily restrictive.

For example, making an entire module depend on another module can cause Terraform to wait for resources that do not actually need to wait.

This can:

- Reduce parallelism.
- Increase plan/apply time.
- Create unnecessary replacement behavior.
- Make the dependency graph harder to understand.

Use it only when the dependency cannot be expressed naturally through references.

---

### Q10. What happens when a data-source value changes?

```hcl
resource "aws_instance" "app" {
  ami = data.aws_ami.latest.id
}
```

**Answer:**

If the selected AMI changes and the provider marks `ami` as requiring replacement, Terraform will propose:

```text
destroy old instance
create new instance
```

The exact behavior depends on the resource schema.

This is why "latest" data sources can be dangerous in production unless replacement is intentionally controlled.

---

# 3. Lifecycle

### Q11. What does `create_before_destroy` do?

```hcl
lifecycle {
  create_before_destroy = true
}
```

**Answer:**

Terraform attempts to create the replacement before destroying the old resource.

This can reduce downtime.

However, it can fail when:

- The cloud resource name must be unique.
- Quotas are reached.
- The platform does not allow two versions simultaneously.
- Dependencies prevent parallel existence.
- The replacement requires an exclusive IP/name/resource.

It is **not a universal zero-downtime switch**.

---

### Q12. What does `prevent_destroy` do?

```hcl
lifecycle {
  prevent_destroy = true
}
```

**Answer:**

Terraform will reject an operation that requires destruction of that resource.

It is useful for critical resources such as production databases.

It does not prevent someone from deleting the resource directly through the cloud console/API.

---

### Q13. What does `ignore_changes` do?

```hcl
lifecycle {
  ignore_changes = [tags]
}
```

**Answer:**

Terraform stops trying to reconcile changes to the specified attribute.

If someone manually changes tags, Terraform will not attempt to revert them.

This is useful when another system legitimately manages part of a resource.

It can also hide drift, so it should be used carefully.

---

### Q14. Why is `ignore_changes = all` usually dangerous?

**Answer:**

Because Terraform effectively stops managing changes to the resource.

You can end up with:

```text
Terraform configuration
        !=
Real infrastructure
```

without Terraform showing useful corrective actions.

It can turn Terraform into little more than a record of resource ownership.

---

# 4. Drift Detection

### Q15. Someone manually changes an instance from `t3.medium` to `t3.large`. What happens?

**Answer:**

If Terraform refreshes the resource and the attribute is managed, Terraform detects the difference.

Example:

```text
Configuration: t3.medium
State/Infrastructure: t3.large
```

Terraform may propose changing it back to:

```text
t3.medium
```

The exact plan depends on the resource schema.

---

### Q16. Difference between `plan` and `plan -refresh-only`?

**Answer:**

Normal:

```bash
terraform plan
```

Refreshes state and then determines whether infrastructure must change to match configuration.

Refresh-only:

```bash
terraform plan -refresh-only
```

Shows changes that would update Terraform state to reflect the real infrastructure without proposing normal configuration-driven infrastructure changes.

Useful for investigating drift.

---

### Q17. Terraform says "No changes" even though infrastructure changed. Why?

**Answer:**

Possible reasons:

1. Attribute is not managed by Terraform.
2. Attribute is computed.
3. `ignore_changes` is configured.
4. Provider does not expose the change.
5. Resource was changed outside Terraform in a way the provider does not detect.
6. Terraform state/backend is stale or incorrect.
7. The observed change is not represented in the Terraform resource schema.

A strong engineer should investigate the provider schema and state rather than assuming Terraform is always correct.

---

# 5. Modules

### Q18. What makes a Terraform module reusable?

**Answer:**

A good module should have:

- Clear inputs.
- Clear outputs.
- Strong variable types.
- Sensible defaults.
- Minimal hard-coded environment assumptions.
- Documentation.
- Versioning.
- Predictable naming.
- Minimal hidden dependencies.

Bad module:

```text
Huge module
Everything hard-coded
Environment-specific logic everywhere
```

Good module:

```text
Inputs -> predictable resources -> outputs
```

---

### Q19. A module is used by 50 applications and you need to add a required variable. What do you do?

**Answer:**

Do not immediately make the variable mandatory if that will break 50 consumers.

Safer approaches:

1. Introduce a default.
2. Release a new module version.
3. Communicate the breaking change.
4. Migrate consumers gradually.
5. Remove the default in a future major version if appropriate.

Use semantic versioning for reusable modules.

---

### Q20. How should enterprise Terraform modules be versioned?

**Answer:**

Use explicit versions.

For example:

```hcl
module "network" {
  source = "git::https://github.com/company/terraform-network.git?ref=v2.3.1"
}
```

Avoid pointing production directly at:

```text
main
```

because the module can change without the consuming configuration changing.

---

# 6. `count` vs `for_each`

### Q21. What is the problem with this?

```hcl
resource "aws_instance" "app" {
  count = length(var.instances)
}
```

Suppose:

```text
["web", "api", "worker"]
```

becomes:

```text
["api", "worker"]
```

**Answer:**

Resources are indexed:

```text
app[0]
app[1]
app[2]
```

Removing the first element shifts indexes.

Terraform can therefore think existing resources have changed identity.

That can result in unnecessary replacement or modification.

---

### Q22. How does `for_each` help?

**Answer:**

Use stable keys:

```hcl
resource "aws_instance" "app" {
  for_each = {
    web    = "t3.medium"
    api    = "t3.small"
    worker = "t3.large"
  }

  instance_type = each.value
}
```

Resources become:

```text
aws_instance.app["web"]
aws_instance.app["api"]
aws_instance.app["worker"]
```

Removing `web` does not shift the identity of `api` or `worker`.

---

### Q23. When would you use `count` instead of `for_each`?

**Answer:**

Use `count` when resources are essentially identical and controlled by a simple numeric condition.

Example:

```hcl
count = var.create_resource ? 1 : 0
```

Use `for_each` when resources have stable identities or different configuration.

---

# 7. Providers and Versions

### Q24. What does this mean?

```hcl
version = "~> 5.0"
```

**Answer:**

It allows compatible versions in the 5.x range while preventing an upgrade to 6.x.

It is different from:

```hcl
>= 5.0
```

which can allow future major versions.

For production, understand exactly what version constraints permit.

---

### Q25. Difference between `terraform init` and `terraform init -upgrade`?

**Answer:**

```bash
terraform init
```

Initializes providers/modules and normally respects the existing dependency selections/lock file.

```bash
terraform init -upgrade
```

actively looks for newer provider/module versions permitted by the configuration and updates dependency selections.

---

### Q26. What is `.terraform.lock.hcl`?

**Answer:**

It records selected provider versions and checksums.

It helps ensure that different environments use verified provider packages consistently.

Normally it should be committed to source control.

---

# 8. Security

### Q27. If a variable is marked sensitive, is the secret secure?

```hcl
variable "db_password" {
  sensitive = true
}
```

**Answer:**

No.

`sensitive = true` mainly prevents Terraform from displaying the value in normal CLI output.

The secret can still exist in:

- Terraform state.
- Plan files.
- Backend storage.
- CI/CD systems.
- Provider APIs.
- Logs if handled incorrectly.

You still need secure secret management and encrypted remote state.

---

### Q28. Where can Terraform secrets exist?

**Answer:**

Potential locations include:

```text
terraform.tfstate
terraform.tfplan
Terraform Cloud/Enterprise state
CI/CD environment variables
Provider API requests
Logs
Shell history
```

Treat Terraform state as sensitive.

---

### Q29. Someone commits `terraform.tfstate` to Git. What do you do?

**Answer:**

Assume secrets may have been exposed.

Immediately:

1. Remove it from the repository.
2. Determine whether secrets were present.
3. Rotate exposed credentials.
4. Check Git history.
5. Move state to a secure remote backend.
6. Restrict access.
7. Add `.gitignore` rules.
8. Review repository access/logs.

Simply deleting the latest file is insufficient because Git history may still contain it.

---

# 9. Import and Migration

### Q30. You have 500 manually created Azure resources. Management wants Terraform management. What is your strategy?

**Answer:**

Do not blindly import everything at once.

Use phases:

```text
Inventory
   ↓
Categorize
   ↓
Write Terraform configuration
   ↓
Import
   ↓
Plan
   ↓
Reconcile differences
   ↓
Validate
   ↓
Move to CI/CD
```

Start with lower-risk resources and establish a repeatable import process.

---

### Q31. After importing a resource, Terraform shows 30 changes. Is the import broken?

**Answer:**

No.

Import primarily establishes the relationship between:

```text
Terraform address
        ↕
Existing cloud resource
```

It does not automatically create perfect Terraform configuration.

You still need to reconcile configuration with the actual resource.

---

# 10. Terraform Cloud / Enterprise

### Q32. Why use remote state in an enterprise?

**Answer:**

Benefits include:

- Centralized state.
- Locking.
- Access control.
- State versioning.
- Collaboration.
- Auditability.
- Remote execution options.
- Reduced risk of local state loss.

---

### Q33. What is a Terraform Enterprise workspace?

**Answer:**

A workspace is a unit where Terraform configuration, variables, state, runs, and execution settings are managed.

A common enterprise pattern is:

```text
workspace
   ↓
environment
   ↓
state
   ↓
Terraform runs
```

Do not confuse workspace with the Terraform CLI working directory.

---

### Q34. Why use Terraform Enterprise agents?

**Answer:**

Agents are useful when Terraform execution needs access to private infrastructure or networks that cannot be reached directly from the public Terraform service.

For example:

```text
Terraform Enterprise
        ↓
Agent
        ↓
Private network
        ↓
Azure/AWS/on-prem
```

The agent executes Terraform in the organization's network context.

---

# 11. Production Scenarios

### Q35. Terraform suddenly wants to destroy 147 production resources. What are your first steps?

**Answer:**

**Do not apply.**

Investigate:

```bash
terraform plan
terraform show
terraform state list
terraform state show <resource>
terraform providers
```

Then check:

1. Git diff.
2. State version/history.
3. Backend/workspace.
4. Provider version.
5. Variable values.
6. Environment/account/subscription.
7. Resource addresses.
8. Module changes.
9. State corruption/movement.
10. Recent provider/module upgrades.

A very common real-world mistake is running Terraform against the **wrong workspace/account/subscription**.

---

### Q36. Terraform apply fails halfway. What happens to state?

**Answer:**

Terraform updates state as resources are successfully created/changed.

Therefore, state can represent a partially completed deployment.

Do not simply rerun blindly.

First:

```bash
terraform plan
```

Understand what Terraform believes exists and what remains to be created/changed.

Then correct the underlying failure and continue.

---

### Q37. Terraform is stuck at:

```text
Acquiring state lock...
```

What do you investigate?

**Answer:**

Check:

1. Whether another Terraform run is active.
2. CI/CD jobs.
3. Terraform Enterprise runs.
4. Backend lock state.
5. Failed/abandoned runs.
6. Network/backend availability.

Only use force-unlock after confirming that the lock is genuinely stale.

```bash
terraform force-unlock <LOCK_ID>
```

Never force-unlock blindly.

---

### Q38. Two pipelines run against the same workspace. Pipeline A plans, Pipeline B applies, then A applies the old plan. What can go wrong?

**Answer:**

The saved plan is based on a particular state/configuration.

If the state has changed between planning and applying, Terraform can reject the plan rather than applying it.

More broadly, CI/CD should serialize operations against the same state/workspace and avoid stale plans.

Use pipeline concurrency controls and the backend's state locking.

---

# 12. Saved Plans

### Q39. Why use:

```bash
terraform plan -out=tfplan
terraform apply tfplan
```

**Answer:**

It allows a controlled workflow:

```text
Plan
 ↓
Review
 ↓
Approval
 ↓
Apply exactly the reviewed plan
```

This is common in production CI/CD.

The plan file itself can contain sensitive information, so protect it.

---

### Q40. Can a saved plan become invalid?

**Answer:**

Yes.

The plan is based on a particular state/configuration/provider context.

If state changes before applying it, Terraform may refuse to apply it.

This is why production pipelines should prevent conflicting runs between plan and apply.

---

# 13. Unknown Values

### Q41. How does Terraform handle values that don't exist during plan?

**Answer:**

Terraform has the concept of **unknown values**.

For example:

```hcl
resource "aws_instance" "app" {
  # ID does not exist until creation
}

resource "aws_eip" "app" {
  instance = aws_instance.app.id
}
```

During planning, Terraform may know:

```text
aws_instance.app.id = (known after apply)
```

Terraform can still build the dependency graph even though the actual value is not known yet.

---

### Q42. Difference between `null`, unknown, and empty string?

**Answer:**

```text
null
```

Generally means no value / absence of a value.

```text
""
```

is a known string with zero characters.

An **unknown value** means Terraform cannot determine the value during planning, but expects it to become known during apply.

These are not interchangeable.

---

# 14. Final Senior-Level Challenge

### Q43. Terraform wants to replace a production database. Management says "just apply." What do you do?

**Answer:**

Do not immediately apply.

First identify the exact replacement trigger.

Example:

```text
-/+ aws_database_instance.prod
```

Then determine:

1. Which attribute causes replacement?
2. Is the configuration intentional?
3. Did a provider upgrade cause it?
4. Is there drift?
5. Did the resource address change?
6. Is the state correct?
7. Can a `moved` block/state move solve it?
8. Can the change be performed safely?
9. Can `create_before_destroy` work?
10. What is the downtime impact?
11. What is the backup/restore strategy?
12. What is the rollback plan?

The correct answer is not simply **"don't apply."**

The candidate should be able to explain **why Terraform wants replacement and how to safely resolve it**.

---

# Interview Scoring Guide

For a Senior DevOps/Terraform candidate:

| Area | Weight |
|---|---:|
| State & state recovery | 20% |
| Dependency & lifecycle | 15% |
| Modules & reusable design | 15% |
| Drift/import/migration | 15% |
| Providers/versioning | 10% |
| Security/secrets | 10% |
| CI/CD & remote execution | 10% |
| Basic Terraform syntax | 5% |

## Candidate Levels

### Junior

Can explain:

- Resources
- Variables
- Outputs
- Providers
- Basic `plan`/`apply`
- Simple modules

### Mid-Level

Can explain:

- State
- Remote backend
- `for_each`
- `count`
- Lifecycle
- Modules
- Provider versions
- Drift
- Import
- CI/CD

### Senior

Should comfortably reason about:

- State recovery
- State locking
- Provider upgrade impact
- Dependency graph
- Complex module design
- Drift
- Import/migration
- Saved plans
- Production safety
- Terraform Enterprise/Cloud
- CI/CD concurrency
- Secrets
- Disaster recovery

### Strong Senior / Terraform SME

Should be able to debug a scenario where:

```text
Terraform wants to destroy production
            ↓
Determine why
            ↓
Validate state
            ↓
Validate provider
            ↓
Validate configuration
            ↓
Validate actual infrastructure
            ↓
Determine safe remediation
            ↓
Execute with controlled rollout
```

The strongest signal is **not how many Terraform commands they remember**. It is whether they understand Terraform's state model and can safely diagnose why Terraform wants to change production.
