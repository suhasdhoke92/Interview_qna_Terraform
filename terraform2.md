# Terraform Data Source — Interview Questions

A **data source** allows Terraform to **read information about an existing resource**. It does not create or manage the lifecycle of that resource.

---

## Q1. What is a Terraform data source?

**Answer:**

A data source is used to retrieve information about an existing infrastructure object.

Example:

```hcl
data "aws_vpc" "existing" {
  id = "vpc-123456"
}
```

Terraform reads the existing VPC and exposes its attributes:

```hcl
data.aws_vpc.existing.id
data.aws_vpc.existing.cidr_block
```

---

## Q2. What is the difference between a resource and a data source?

**Answer:**

```text
Resource      → Create/manage infrastructure
Data source   → Read existing information
```

Example resource:

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}
```

Example data source:

```hcl
data "aws_vpc" "existing" {
  id = "vpc-123456"
}
```

The first creates/manages the VPC. The second only reads an existing VPC.

---

## Q3. Give a real-world use case for a data source.

**Answer:**

Suppose the networking team already manages the production VPC, but your application team needs to create a subnet inside that VPC.

Instead of creating another VPC, use a data source:

```hcl
data "aws_vpc" "production" {
  tags = {
    Name = "production-vpc"
  }
}

resource "aws_subnet" "app" {
  vpc_id     = data.aws_vpc.production.id
  cidr_block = "10.0.10.0/24"
}
```

The VPC is managed elsewhere; Terraform only reads its ID.

---

## Q4. Can a data source create or delete infrastructure?

**Answer:**

No.

A data source is for **reading/querying** existing information.

```text
data → read
resource → create/update/delete
```

---

## Q5. What is a potential problem with a data source?

**Answer:**

The value returned by a data source can change.

For example:

```hcl
data "aws_ami" "latest" {
  most_recent = true
}
```

If a newer AMI becomes available, a resource using:

```hcl
ami = data.aws_ami.latest.id
```

may require an update or replacement depending on the resource and provider behavior.

**Senior-level follow-up:**

> Would you use `latest` automatically for a production AMI?

A good answer should discuss **controlled AMI versioning, testing, and avoiding unexpected production replacements**.
