<p align="center">
  <img src="assets/banner-leaky-bucket.svg" width="820" alt="The Leaky Bucket">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/EASY-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/S3-1a0633?style=for-the-badge&labelColor=A479C4">
  <img src="https://img.shields.io/badge/BLOCK_PUBLIC_ACCESS-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

## Overview

> [!NOTE]
> **TL;DR** — Carl made a bucket public again. You audit all three S3 access layers (Block Public Access, bucket policy, ACL), confirm anonymous access with `--no-sign-request`, lock it back down, and build a secure bucket from the ground up. Framed by the Verizon / Dow Jones / Pegasus S3 exposures.

## Real-world incidents — the "leaky bucket" pattern

- **Verizon** — a vendor left call-centre data public: ~6M records (names, addresses, account PINs).
- **Dow Jones** — a public bucket exposed 2.2M subscribers' personal and financial data.
- **Pegasus Airlines** — 6.5 TB, 23M files (flight charts, navigation material, crew data).

> [!IMPORTANT]
> Every case followed the same script: a bucket opened for a project → **Block Public Access not enabled (or turned off)** → no monitoring → exposure lasted months to years. Attackers did nothing clever; they ran public tools that scan AWS for open buckets. **Block Public Access** is the safety net that overrides any policy or ACL that would make a bucket public.

## The three access layers

| Layer | Controls | Overridden by |
|---|---|---|
| **ACL** | who can list/read the bucket and objects | Block Public Access |
| **Bucket policy** | fine-grained rules (per-action, per-principal) | Block Public Access |
| **Block Public Access** | account/bucket switch that blocks all public access | **nothing** |

## Identification — audit every layer

```bash
BUCKET_NAME=$(aws s3api list-buckets --query "Buckets[?contains(Name,'leaky')].Name" --output text)

aws s3api get-bucket-ownership-controls --bucket $BUCKET_NAME   # BucketOwnerPreferred = ACLs active
aws s3api get-public-access-block        --bucket $BUCKET_NAME   # all four false ← BAD
aws s3control get-public-access-block --account-id $ACCOUNT_ID   # account-level also false
aws s3api get-bucket-policy --bucket $BUCKET_NAME                # Principal:* + s3:GetObject ← public
aws s3api get-bucket-acl    --bucket $BUCKET_NAME                # AllUsers: READ ← public ACL grant

# what an attacker does — no credentials at all:
aws s3 ls s3://$BUCKET_NAME/ --no-sign-request                   # customer-records/, flag.txt, ...
aws s3 cp s3://$BUCKET_NAME/flag.txt - --no-sign-request         # the flag, anonymously
```

Three tell-tales of a leaky bucket: **all four BPA settings false** (account + bucket), an **over-broad policy** (`Principal:*`), and an **AllUsers/AuthenticatedUsers grant** in the ACL. The `customer-records/` prefix also exposes `export.csv` — real customer data, public, exactly like Verizon/Dow Jones.

## Remediation — close all three layers

```bash
# 1. Block Public Access (the overriding safety net)
aws s3api put-public-access-block --bucket $BUCKET_NAME --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
# 2. remove the wildcard-principal policy
aws s3api delete-bucket-policy --bucket $BUCKET_NAME
aws s3api get-bucket-policy --bucket $BUCKET_NAME   # → error NoSuchBucketPolicy (confirms removal)
# 3. reset the ACL to private
aws s3api put-bucket-acl --bucket $BUCKET_NAME --acl private
# verify: anonymous denied, authenticated still works
curl -s "https://$BUCKET_NAME.s3.amazonaws.com/flag.txt"   # AccessDenied
aws s3 cp s3://$BUCKET_NAME/flag.txt -                       # works (you have permission)
```

> [!TIP]
> **`NoSuchBucketPolicy` is the expected result** — it proves the policy is gone. The real test is that an anonymous `curl` / `--no-sign-request` returns **AccessDenied** while authenticated access still works, so you've closed the hole without breaking functionality.

## Build it securely

```bash
aws s3api create-bucket --bucket $SECURE_BUCKET --region us-east-1
aws s3api put-public-access-block --bucket $SECURE_BUCKET --public-access-block-configuration "...=true"
aws s3api put-bucket-ownership-controls --bucket $SECURE_BUCKET \
  --ownership-controls '{"Rules":[{"ObjectOwnership":"BucketOwnerEnforced"}]}'   # disable ACLs entirely
aws s3api put-bucket-policy --bucket $SECURE_BUCKET --policy file://secure-policy.json  # DenyNonTLS
aws s3control put-public-access-block --account-id $ACCOUNT_ID --public-access-block-configuration "...=true"
aws s3api put-bucket-logging --bucket $SECURE_BUCKET --bucket-logging-status "{...}"    # access logging
```

The checker validates four controls: Block Public Access, ACLs disabled (`BucketOwnerEnforced`), a **TLS-deny** policy (`aws:SecureTransport:false` → Deny), and access logging. The logging bucket's policy uses an `aws:SourceAccount` condition — the same confused-deputy guard seen with Flow Logs.

## Task answers

| # | Task | Question | Answer |
|---|------|----------|--------|
| 1 | Introduction | Let's patch some buckets! | *(no answer)* |
| 2 | Real-World Incident | Control that would have partially prevented these | `Block Public Access` |
| 3 | Identification | Flag from the public S3 object | `THM{CARL_MESSED_UP_*****}` |
| 3 | Identification | Other exposed file in customer-records | `export.csv` |
| 4 | Remediation | Error when verifying the policy was removed | `NoSuchBucketPolicy` |
| 4 | Remediation | Flag | `THM{LOCKED_DOWN_AND_*******}` |
| 5 | Build It Securely | Flag | `THM{SECURED_FROM_DAY_***}` |
| 6 | Conclusion | No more leaks | *(no answer)* |

## Lessons learned

- **Enable Block Public Access at the account level as the default baseline** — it overrides any policy or ACL that would go public.
- **A wildcard principal (`Principal:*`) should be an edge case, not the rule.**
- **Disable ACLs with `BucketOwnerEnforced`** on all new buckets — ACLs are legacy.
- **Monitor for public buckets regularly** (IAM Access Analyzer, AWS Config) — exposure that lasts months is the real damage.

---

<p align="center">
  <a href="README.md">
    <img src="assets/more_writeups.svg" width="360" alt="Back to AWS Security Path">
  </a>
</p>
