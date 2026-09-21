<p align="center">
  <img src="assets/banner-shared-snapshot.svg" width="820" alt="The Shared Snapshot">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/EASY-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/EBS_SNAPSHOTS-1a0633?style=for-the-badge&labelColor=A479C4">
  <img src="https://img.shields.io/badge/DATA_EXPOSURE-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

## Overview

> [!NOTE]
> **TL;DR** — An EBS snapshot can be shared with the whole world (`createVolumePermission: Group=all`) and copied silently by anyone scanning the public catalog. You find a public, unencrypted snapshot, make it private, create an encrypted copy that can only be shared with specific accounts, and build an encrypted-and-private-by-default snapshot workflow. Framed by Bishop Fox's 2019 DEF CON research.

## Real-world incident — Bishop Fox / DEF CON 27 (2019)

Researcher Ben Morris built a scanner against the **AWS Public Snapshot Catalog** (the API needs only a standard account). Across all regions it returned **100,000+ publicly exposed snapshots**. The team copied a sample, attached them as EBS volumes, and mounted them — and because they were **unencrypted**, the data was plaintext immediately: AWS access keys, secret keys, SSH private keys, TLS/SSL certificates, proprietary source code. The owners had no indication their data was accessed.

> [!IMPORTANT]
> Three failures: a snapshot set public with **`createVolumePermission: Group=all`** (persists indefinitely once set), **no encryption** (public = plaintext to anyone who copies it), and **no detection** — copying a public snapshot generates **no CloudTrail event in the source account**, so owners relying on their own logs can't see it.

## Identification — two findings on one snapshot

```bash
aws ec2 describe-snapshots --owner-ids self --output table
#   Encrypted: False                ← finding 1: unencrypted
#   DataClassification: Sensitive   ← the tag says it holds sensitive data
aws ec2 describe-snapshot-attribute --snapshot-id $SNAPSHOT_ID --attribute createVolumePermission
#   "CreateVolumePermissions": [{"Group": "all"}]   ← finding 2: publicly shared
```

**`Encrypted: False` + `Group: all`** is the DEF CON combination: a public, unencrypted snapshot anyone can copy and read in plaintext. `Group=all` is the snapshot equivalent of S3's `Principal:*` / AllUsers — one setting makes the resource globally public.

## Remediation — private, then encrypted copy

```bash
# 1. remove public sharing
aws ec2 modify-snapshot-attribute --snapshot-id $SNAPSHOT_ID \
  --attribute createVolumePermission --operation-type remove --group-names all
aws ec2 describe-snapshot-attribute --snapshot-id $SNAPSHOT_ID --attribute createVolumePermission
#   CreateVolumePermissions: []   ← private
# (share securely with a SPECIFIC account instead of the world, if needed)
#   ... --operation-type add --user-ids <TARGET_ACCOUNT_ID>

# 2. encrypted copy (you can't encrypt a snapshot in place — copy it)
ENCRYPTED_SNAP=$(aws ec2 copy-snapshot --source-region "$AWS_REGION" \
  --source-snapshot-id $SNAPSHOT_ID --encrypted --kms-key-id alias/lab-ebs-key \
  --query SnapshotId --output text)
aws ec2 describe-snapshots --snapshot-ids $ENCRYPTED_SNAP --query "Snapshots[0].{Encrypted:Encrypted}"  # true
```

> [!WARNING]
> Removing `Group=all` stops future exposure, but if the snapshot was public you must **assume the data leaked and rotate the secrets** it held (access keys, passwords, SSH keys) — exactly the Bishop Fox findings. You **can't encrypt a snapshot in place**; you create an encrypted copy, which can then be shared safely with specific accounts, and AWS blocks any attempt to `Group=all` a custom-KMS snapshot.

## Build it securely — encrypted + private by default

```bash
# dedicated KMS key with alias + rotation
EBS_KEY_ID=$(aws kms create-key --description "..." --key-usage ENCRYPT_DECRYPT --query KeyMetadata.KeyId --output text)
aws kms create-alias --alias-name "alias/ebs-data-key" --target-key-id $EBS_KEY_ID
aws kms enable-key-rotation --key-id $EBS_KEY_ID
# encrypted volume → snapshot inherits encryption, private by default
VOLUME_ID=$(aws ec2 create-volume --size 1 --availability-zone $AZ --encrypted --kms-key-id $EBS_KEY_ID ...)
SECURE_SNAP=$(aws ec2 create-snapshot --volume-id $VOLUME_ID ...)
```

A snapshot from an encrypted volume inherits encryption and is **private by default**. Audit regularly:

```bash
# public snapshots (Group=all) — should return nothing
aws ec2 describe-snapshots --owner-ids self --query "Snapshots[*].SnapshotId" --output text | tr '\t' '\n' | while read s; do
  p=$(aws ec2 describe-snapshot-attribute --snapshot-id $s --attribute createVolumePermission \
      --query "CreateVolumePermissions[?Group=='all']" --output text)
  [ -n "$p" ] && echo "PUBLIC: $s"; done
# unencrypted snapshots
aws ec2 describe-snapshots --owner-ids self --filters "Name=encrypted,Values=false" --output table
```

In production, use the AWS Config rule **`ec2-ebs-snapshot-public-restorable-check`** for continuous automated detection.

## Task answers

| # | Task | Question | Answer |
|---|------|----------|--------|
| 1 | Introduction | Who needs snapshots anyway? | *(no answer)* |
| 2 | Real-World Incident | `createVolumePermission` value that makes a snapshot public | `Group=all` |
| 3 | Identification | Flag | `THM{SNAPSHOTS_TRAVEL_****}` |
| 4 | Remediation | Flag | `THM{PRIVATE_AND_*********}` |
| 5 | Build It Securely | Flag | `THM{AUDIT_SNAPSHOTS_*********}` |
| 6 | Conclusion | There goes my snapshot | *(no answer)* |

## Lessons learned

- **A public EBS snapshot is as dangerous as a public S3 bucket, minus even the URL barrier** — it's in the public AWS catalog and can be copied silently.
- **Encrypt all volumes and snapshots with a customer-managed KMS key** — encrypted snapshots can't be copied without the key, and AWS won't let you make a custom-KMS snapshot public.
- **There's no record of who copied a shared snapshot** — treat any public snapshot as an active exposure and rotate its secrets.
- **Audit continuously** with `ec2-ebs-snapshot-public-restorable-check`, and default new volumes/snapshots to encrypted.

---

<p align="center">
  <a href="README.md">
    <img src="assets/more_writeups.svg" width="360" alt="Back to AWS Security Path">
  </a>
</p>
