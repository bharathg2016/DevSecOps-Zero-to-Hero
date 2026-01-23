# TruffleHog 
---

## What Is TruffleHog?

TruffleHog scans Git repositories (including full history)
to find leaked secrets like AWS keys, tokens, and passwords.

---

## Step 1: Install TruffleHog

macOS:
```bash
brew install trufflehog
```

Linux:
```bash
pip install trufflehog
```

Verify:
```bash
trufflehog --version
```

---

## Step 2: Insecure Terraform code

```bash
mkdir terraform-trufflehog-demo
cd terraform-trufflehog-demo
git init
```

Create `provider.tf` (INSECURE – ON PURPOSE):

```bash
cat <<EOF > provider.tf
provider "aws" {
  access_key = "AKIAFAKEKEY123"
  secret_key = "fakeSecret123"
  region     = "us-east-1"
}
EOF
```

Commit it:
```bash
git add provider.tf
git commit -m "Add AWS provider with credentials"
```

🚨 The secret is now permanently in Git history.

---

## Step 3: Scan with TruffleHog

```bash
trufflehog git file://.
```

---

## What You Should See

- AWS credential detected
- Commit hash shown
- File: provider.tf
- Exact line number

---

## Step 4: Prove Deleting Is NOT Enough

```bash
rm provider.tf
git commit -am "remove credentials"
```

Scan again:
```bash
trufflehog git file://.
```

🚨 Secret is STILL detected.

---

## Key Takeaway

If a secret enters Git, assume compromise.
TruffleHog helps you detect this before attackers do.
