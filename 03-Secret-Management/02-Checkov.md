# Checkov 

---

## Problem 

Terraform can deploy insecure infrastructure without warnings:
- Public S3 buckets
- SSH open to the internet
- No encryption

---

## What Is Checkov?

Checkov scans Terraform code for security misconfigurations
BEFORE infrastructure is deployed.

---

### Step 1: Install Checkov

```bash
pip install checkov
```

Verify:
```bash
checkov --version
```

---

## Step 2: Create Insecure Terraform Code

```bash
mkdir terraform-checkov-demo
cd terraform-checkov-demo
```

Create `main.tf` (INSECURE ON PURPOSE):

```bash
cat <<EOF > main.tf
resource "aws_s3_bucket" "public_bucket" {
  bucket = "my-public-demo-bucket"
  acl    = "public-read"
}

resource "aws_security_group" "open_ssh" {
  name = "open-ssh"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
EOF
```

---

## Step 3: Scan with Checkov

```bash
checkov -d .
```

---

### What You Should See

❌ Public S3 bucket  
❌ SSH open to 0.0.0.0/0  

Terraform did NOT warn you.
Checkov did.

---

## Step 4: Fix the Terraform Code

```bash
cat <<EOF > main.tf
resource "aws_s3_bucket" "secure_bucket" {
  bucket = "my-secure-demo-bucket"

  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }
}

resource "aws_security_group" "restricted_ssh" {
  name = "restricted-ssh"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]
  }
}
EOF
```

---

## Step 5: Re-Scan

```bash
checkov -d .
```

✅ No critical findings.

---

## Key Takeaway

Terraform builds infrastructure.
Checkov secures it before deployment.
