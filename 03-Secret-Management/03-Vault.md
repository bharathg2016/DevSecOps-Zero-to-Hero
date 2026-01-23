# Vault Production Setup on EC2 + GitHub Actions 

GitHub Actions → Vault (EC2) → AWS (temporary credentials)

---

## Step 1: Launch EC2 for Vault

- AMI: Ubuntu 22.04
- Instance type: t2.micro
- Security Group:
  - SSH (22) → your IP
  - Vault (8200) → GitHub IPs (demo: 0.0.0.0/0)

SSH:
```bash
ssh -i your-key.pem ubuntu@EC2_PUBLIC_IP
```

---

## Step 2: Install Vault

```bash
sudo apt update
sudo apt install -y unzip wget
wget https://releases.hashicorp.com/vault/1.15.5/vault_1.15.5_linux_amd64.zip
unzip vault_1.15.5_linux_amd64.zip
sudo mv vault /usr/local/bin/
vault version
```

---

## Step 3: Configure Vault

```bash
sudo mkdir -p /opt/vault/data /etc/vault
sudo chown -R ubuntu:ubuntu /opt/vault /etc/vault
```

```bash
cat <<EOF > /etc/vault/vault.hcl
storage "file" {
  path = "/opt/vault/data"
}

listener "tcp" {
  address = "0.0.0.0:8200"
  tls_disable = 1
}

ui = true
EOF
```

---

## Step 4: Start Vault

```bash
vault server -config=/etc/vault/vault.hcl
```

---

## Step 5: Initialize & Unseal Vault

New terminal:
```bash
export VAULT_ADDR="http://127.0.0.1:8200"
vault operator init
```

Save:
- Unseal keys
- Root token

Unseal (run 3 times):
```bash
vault operator unseal
```

Login:
```bash
vault login <ROOT_TOKEN>
```

---

## Step 6: Enable AWS Secrets Engine

```bash
vault secrets enable aws
```

```bash
vault write aws/config/root \
  access_key=REAL_AWS_KEY \
  secret_key=REAL_AWS_SECRET \
  region=us-east-1
```

---

## Step 7: Create AWS Role

```bash
cat <<EOF > aws-policy.json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["ec2:*", "s3:*"],
    "Resource": "*"
  }]
}
EOF
```

```bash
vault write aws/roles/terraform-role \
  credential_type=iam_user \
  policy_document=@aws-policy.json \
  default_ttl=15m \
  max_ttl=1h
```

---

## Step 8: Enable GitHub OIDC Auth

```bash
vault auth enable jwt
```

```bash
vault write auth/jwt/config \
  oidc_discovery_url="https://token.actions.githubusercontent.com" \
  bound_issuer="https://token.actions.githubusercontent.com"
```

---

## Step 9: Create GitHub Role

```bash
cat <<EOF > terraform-policy.hcl
path "aws/creds/terraform-role" {
  capabilities = ["read"]
}
EOF
```

```bash
vault policy write terraform terraform-policy.hcl
```

```bash
vault write auth/jwt/role/github-actions \
  role_type="jwt" \
  user_claim="repository" \
  bound_claims='{
    "repository": "ORG/REPO",
    "ref": "refs/heads/main"
  }' \
  policies="terraform" \
  ttl="15m"
```

---

## Step 10: GitHub Actions Workflow

Create `.github/workflows/terraform.yml`:

```yaml
name: Secure Terraform Pipeline

on:
  push:
    branches: [ "main" ]

permissions:
  id-token: write
  contents: read

jobs:
  terraform:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4

    - name: Authenticate to Vault
      uses: hashicorp/vault-action@v2
      with:
        url: http://EC2_PUBLIC_IP:8200
        method: jwt
        role: github-actions

    - name: Terraform Init
      run: terraform init

    - name: Checkov Scan
      run: checkov -d .

    - name: Terraform Apply
      run: terraform apply -auto-approve
```

---
