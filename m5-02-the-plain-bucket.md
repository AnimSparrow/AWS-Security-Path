<p align="center">
  <img src="assets/banner-plain-bucket.svg" width="820" alt="The Plain Bucket">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/EASY-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/SSE_KMS-1a0633?style=for-the-badge&labelColor=A479C4">
  <img src="https://img.shields.io/badge/ENCRYPTION-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

## Overview

> [!NOTE]
> **TL;DR** — S3 encrypts everything with SSE-S3 by default, but that gives no separate authorization boundary — anyone with `s3:GetObject` decrypts transparently. You identify SSE-S3, switch the bucket to SSE-KMS with a customer-managed key, prove a role with S3 read but no `kms:Decrypt` is now denied, and build a bucket that enforces KMS from day one. Framed by the 2022 LastPass breach.

## Real-world incident — LastPass (2022)

A chain: access to a dev environment (Aug 2022) → source code and docs → identifying one of four engineers with production access → exploiting a third-party media server on that engineer's **personal device** → capturing cloud credentials → downloading **encrypted customer vault backups and metadata** from S3.

> [!IMPORTANT]
> The vault buckets used **S3-managed encryption with no customer-managed KMS key**, so a single set of stolen S3 credentials was enough to read everything. A customer-managed KMS key with its own policy adds a **second authorization boundary**: stolen S3 creds without `kms:Decrypt` return ciphertext, not data.

## SSE-S3 vs SSE-KMS vs SSE-C

| | SSE-S3 | SSE-KMS | SSE-C |
|---|---|---|---|
| Key managed by | AWS (S3) | AWS (KMS) | customer (external) |
| Default | **yes (automatic)** | no (must enable) | no |
| Audit trail | no (basic) | **yes (CloudTrail)** | no |
| Access control | IAM / bucket policy | **IAM + KMS key policy** | client header |
| Best for | general use | compliance / enhanced | legal / legacy |

## Identification — SSE-S3 has no KMS boundary

```bash
aws s3api get-bucket-encryption --bucket "$BUCKET_NAME"
#   SSEAlgorithm: AES256   ← default SSE-S3, no explicit config
aws s3api head-object --bucket "$BUCKET_NAME" --key sample-data.txt \
  --query '{SSE:ServerSideEncryption,SSEKMSKeyId:SSEKMSKeyId}'
#   SSE: AES256, SSEKMSKeyId: null   ← S3-managed keys, zero KMS
aws s3 cp "s3://$BUCKET_NAME/flag.txt" -   # transparent, s3:read is enough
```

`AES256` = SSE-S3, `aws:kms` + a key id = SSE-KMS. Here everything is SSE-S3: encrypted at rest, but `s3:GetObject` decrypts automatically — the exact LastPass gap. A `pii/customer-export.csv` sits in the bucket protected only by S3-managed keys.

## Remediation — enable SSE-KMS with a customer-managed key

```bash
KEY_ARN=$(aws kms describe-key --key-id "alias/lab-cmk" --query "KeyMetadata.Arn" --output text)
aws s3api put-bucket-encryption --bucket "$BUCKET_NAME" --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"aws:kms","KMSMasterKeyID":"'$KEY_ARN'"},"BucketKeyEnabled":true}]}'

# new object gets aws:kms; the OLD object is still AES256 (not retroactive)
aws s3 cp ./kms-test.txt "s3://$BUCKET_NAME/kms-test.txt"
aws s3api head-object --bucket "$BUCKET_NAME" --key kms-test.txt --query '{SSE:ServerSideEncryption}'  # aws:kms

# a role with s3:read but NO kms:Decrypt is now blocked
aws s3 cp "s3://$BUCKET_NAME/kms-test.txt" -   # → AccessDenied
```

> [!IMPORTANT]
> Two things to remember. **Default encryption only applies to new writes** — it does not re-encrypt existing objects (the old `sample-data.txt` stays AES256 until rewritten). And **`BucketKeyEnabled: true`** is a cost best practice: the S3 Bucket Key cuts the number of direct KMS requests.

With SSE-KMS, `s3:GetObject` alone is not enough — decryption needs **`kms:Decrypt`** on the key. That's the second gate LastPass lacked.

## Build it securely — enforce KMS with a resource-based policy

```bash
# customer-managed key with alias + rotation
NEW_KEY_ID=$(aws kms create-key --description "..." --query 'KeyMetadata.KeyId' --output text)
aws kms create-alias --alias-name alias/room52-secure-data-key --target-key-id "$NEW_KEY_ID"
aws kms enable-key-rotation --key-id "$NEW_KEY_ID"
# bucket: BucketOwnerEnforced + default SSE-KMS + a bucket policy that enforces KMS
aws s3api put-bucket-policy --bucket "$SECURE_BUCKET" --policy file://enforce-kms.json
```

The **bucket policy is a resource-based policy** (attached to the resource, not an identity). It stacks four `Deny` statements: missing encryption header, wrong algorithm (force `aws:kms`), non-TLS transport, and wrong KMS key id. Default encryption sets it automatically; the policy *enforces* that nobody can put an unencrypted or wrongly-encrypted object — defense in depth. Test: `put-object` without `--server-side-encryption aws:kms --ssekms-key-id` → **AccessDenied**.

## Task answers

| # | Task | Question | Answer |
|---|------|----------|--------|
| 1 | Introduction | Encryption is everywhere | *(no answer)* |
| 2 | Real-World Incident | Control that would have reduced the exfiltration | `customer-managed KMS` |
| 3 | Identification | Flag | `THM{IS_MY_KMS_**}` |
| 4 | Remediation | Extra permission needed to decrypt (SSE-KMS) | `kms:Decrypt` |
| 4 | Remediation | Flag | `THM{I_CANT_READ_****}` |
| 5 | Build It Securely | Policy type that denies put-object without the key | `resource-based` |
| 5 | Build It Securely | Flag | `THM{LOCKED_STOCKED_AND_****}` |
| 6 | Conclusion | Second security? | *(no answer)* |

## Lessons learned

- **SSE-S3 is a baseline; SSE-KMS is a second authorization boundary** — it requires both S3 and KMS key permissions to read data.
- **Default bucket encryption is not retroactive** — existing objects keep their old encryption until rewritten.
- **Enforce encryption with a bucket policy**, don't just set the default — stack denies for missing/wrong algorithm, wrong key, and non-TLS.
- **Layer the sensitive-bucket pattern:** Block Public Access + `BucketOwnerEnforced` + default SSE-KMS + enforcement policy + key rotation and monitoring.

---

<p align="center">
  <a href="README.md">
    <img src="assets/more_writeups.svg" width="360" alt="Back to AWS Security Path">
  </a>
</p>
