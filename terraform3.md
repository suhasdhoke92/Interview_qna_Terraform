# Terraform Interview Traps — `plan` vs `refresh-only`

These questions are designed to test whether a candidate actually understands Terraform's **state, configuration, drift, and refresh behavior**, rather than simply memorizing commands.

---

## Interview Trap 1 — Drift

### Ask the candidate

> Someone manually changes an EC2 instance from `t3.medium` to `t3.large`. I run `terraform plan -refresh-only`. Will Terraform change the EC2 back to `t3.medium`?

### Correct Answer

**No.**

`terraform plan -refresh-only` is for reconciling Terraform's **state with the actual infrastructure**. It does not perform the normal configuration-driven correction.

### Follow-up

> What command would you normally use if you want Terraform to detect the drift and propose changing the EC2 back?

### Answer

```bash
terraform plan
```

---

# Interview Trap 2 — `terraform apply -refresh-only`

### Ask the candidate

> If `terraform plan -refresh-only` shows drift, can I run `terraform apply -refresh-only` to change the EC2 from `t3.large` back to `t3.medium`?

### Correct Answer

**No.**

`terraform apply -refresh-only` applies the **state refresh**, not the normal configuration changes.

The actual EC2 instance remains:

```text
t3.large
```

The purpose is to make Terraform's state reflect what actually exists.

---

# Interview Trap 3 — State vs Configuration

Give the candidate:

```text
Terraform configuration → t3.medium
Terraform state         → t3.medium
AWS infrastructure      → t3.large
```

### Ask

> Which one is wrong?

### Strong Answer

The actual infrastructure has **drifted from the Terraform configuration**.

The state may also be stale until Terraform refreshes it.

### Follow-up

> After a refresh, what could happen to Terraform state?

### Answer

Terraform can update the state to reflect the actual value reported by AWS:

```text
Terraform configuration → t3.medium
Terraform state         → t3.large
AWS infrastructure      → t3.large
```

Now state reflects reality, while configuration still represents the desired state.

---

# Interview Trap 4 — `-refresh=false`

### Ask

> What happens if I run:

```bash
terraform plan -refresh=false
```

while someone has manually changed the EC2 instance?

### Correct Answer

Terraform does **not refresh the state from the provider before planning**.

For example, if Terraform state still says:

```text
t3.medium
```

but AWS is actually:

```text
t3.large
```

Terraform may not detect that manual change during that plan because it is relying on the existing state.

### Important

Do not confuse:

```bash
terraform plan -refresh-only
```

with:

```bash
terraform plan -refresh=false
```

They have very different purposes.

---

# Interview Trap 5 — `ignore_changes`

Give the candidate:

```hcl
resource "aws_instance" "app" {
  instance_type = "t3.medium"

  lifecycle {
    ignore_changes = [instance_type]
  }
}
```

Someone manually changes the instance to:

```text
t3.large
```

### Ask

> Will `terraform plan` propose changing it back?

### Correct Answer

**No.**

Because:

```hcl
ignore_changes = [instance_type]
```

tells Terraform not to manage changes to that attribute.

### Follow-up

> What happens with `terraform plan -refresh-only`?

Terraform can still refresh state to reflect the real value, but `ignore_changes` prevents the normal configuration-driven correction for that attribute.

---

# Interview Trap 6 — Is `refresh-only` a Drift Detection Command?

### Ask

> Is `terraform plan -refresh-only` a drift detection command?

### Strong Answer

**Yes, but that description is incomplete.**

It is useful for identifying differences between:

```text
Terraform state
       vs
Real infrastructure
```

and planning state updates.

However, normal:

```bash
terraform plan
```

is what you use when you want Terraform to determine:

> "What infrastructure changes are required to make reality match my configuration?"

---

# Mental Model

## Normal Plan

```text
Terraform Configuration
          ↓
       Terraform
          ↓
Actual Infrastructure
          ↓
"What infrastructure changes are required?"
```

## Refresh-Only Plan

```text
Terraform State
      ↓
   Refresh
      ↓
Actual Infrastructure
      ↓
"What state information needs to be updated?"
```

---

# Quick Comparison

| Command | Main Purpose |
|---|---|
| `terraform plan` | Determine infrastructure changes required to match configuration |
| `terraform plan -refresh-only` | Determine state changes needed to reflect actual infrastructure |
| `terraform apply` | Apply the normal Terraform plan |
| `terraform apply -refresh-only` | Apply state refresh/reconciliation |
| `terraform plan -refresh=false` | Plan without refreshing state first |

---

# Best Final Interview Question

Ask:

> **Configuration says `t3.medium`, state says `t3.medium`, and AWS says `t3.large`. What happens with `terraform plan`, `terraform plan -refresh-only`, and `terraform plan -refresh=false`?**

### Expected Answer

| Command | Expected behavior |
|---|---|
| `terraform plan` | Refreshes state, detects the actual drift, and normally proposes changing infrastructure back toward configuration |
| `terraform plan -refresh-only` | Focuses on reconciling Terraform state with the actual infrastructure; it does not propose the normal configuration-driven correction |
| `terraform plan -refresh=false` | Does not refresh from AWS first, so it may continue using the stale state and miss the manual change |

This is a strong interview question because the candidate has to understand **configuration vs state vs reality**, not just memorize Terraform commands.
