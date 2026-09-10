<p align="center">
  <img src="assets/banner-over-privileged-user.svg" width="820" alt="The Over-Privileged User">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/EASY-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/IAM-1a0633?style=for-the-badge&labelColor=A479C4">
  <img src="https://img.shields.io/badge/LEAST_PRIVILEGE-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

## Overview

> [!NOTE]
> **TL;DR** — A developer ("Carl") was handed full `*:*` admin "temporarily" and never scoped down. Playing security analyst, you audit the account with the IAM CLI, spot the over-permissive policy, strip it, rebuild access as a scoped group policy, and add a permission boundary as a hard ceiling. Framed by the Code Spaces breach — the company that admin sprawl killed.

## Real-world incident — Code Spaces (2014)

A DDoS was the opening move; while staff fought the traffic flood, the attacker was already inside the AWS console. Staff changed panel passwords, but the attacker had **already created backdoor IAM logins**, saw the recovery attempts, and escalated to destruction: deleted all EBS snapshots, all S3 buckets, all AMIs, and multiple EC2 instances. **Within 12 hours the company was gone** — production, backups, and off-site backups destroyed.

> [!IMPORTANT]
> The core failure wasn't the DDoS — it was that **one compromised identity had unrestricted administrative access**. No permission boundaries, no explicit deny on destructive actions, no MFA on sensitive ops, no IAM-change monitoring. This is the entire case for least privilege in one incident.

## Identification — auditing the user

The methodology: list users → check attached policies → **check inline policies** (they don't show in the managed list) → check group membership.

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws iam list-users
aws iam list-attached-user-policies --user-name carl-the-dev     # AWS201-DevCarlAdmin
aws iam get-policy-version \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/AWS201-DevCarlAdmin --version-id v1
#   → "Action": "*", "Resource": "*", "Effect": "Allow"   ← RED FLAG
aws iam list-user-policies    --user-name carl-the-dev           # inline: none
aws iam list-groups-for-user  --user-name carl-the-dev           # groups: none
```

**Findings:** Carl holds `*:*` (full admin) attached **directly to the user** — not via a group. If his credentials leak, it's Code Spaces all over again: delete any bucket, terminate any EC2, wipe CloudTrail, read any secret.

> [!TIP]
> A quick side-channel to confirm intended scope: `aws iam list-user-tags --user-name carl-the-dev` returns `Role: Developer` — his *intended* role, useful when reasoning about what he should actually have.
>
> Also watch for **misleading policy names**: `list-attached-group-policies` can show `PolicyName: AdministratorAccess` on an ARN that actually points to a scoped `AppAccess` policy. Read the policy *content*, never trust the name.

## Remediation — scope it down

Evaluation order to keep in mind: **explicit deny → explicit allow → implicit deny.**

```bash
# 1. Rip off the god-mode policy → user drops to implicit deny (zero perms)
aws iam detach-user-policy --user-name carl-the-dev \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/AWS201-DevCarlAdmin

# 2. Scoped policy (AppAccess): only what a dev needs —
#    s3:GetObject + s3:ListBucket on thm-app-data-*, ec2:Describe*, CloudWatch Logs read

# 3. Attach via a GROUP, not the user directly
aws iam create-group        --group-name Developers
aws iam attach-group-policy --group-name Developers --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/AppAccess
aws iam add-user-to-group   --group-name Developers --user-name carl-the-dev

# 4. Validate with the Policy Simulator
aws iam simulate-custom-policy --policy-input-list "$POLICY_DOC" \
  --action-names "s3:ListBucket" "s3:GetObject" \
  --resource-arns "arn:aws:s3:::thm-app-data-${ACCOUNT_ID}"
#   ListBucket → allowed, GetObject → allowed
#   s3:DeleteBucket → implicitDeny  (not in the policy = falls through to implicit deny)
```

## Build it securely — three principles

1. **Group-based model** — define company roles as groups (`Accounting`, `Developers`); onboard = add to group, offboard = remove from group.
2. **Least-privilege policies** — specific actions on specific resource ARNs, nothing broader.
3. **Permission boundaries** — an IAM policy that caps the *maximum* permissions an identity can have, **even if a more permissive policy is later attached**. The effective permissions are the intersection.

```bash
aws iam put-user-permissions-boundary --user-name carl-the-dev \
  --permissions-boundary arn:aws:iam::${ACCOUNT_ID}:policy/CarlBoundary
aws iam get-user --user-name carl-the-dev --query "User.PermissionsBoundary"
```

## Task answers

| # | Task | Question | Answer |
|---|------|----------|--------|
| 1 | Introduction | Let's get to work | *(no answer)* |
| 2 | Real-World Incident | Initial attack type | `ddos` |
| 2 | Real-World Incident | Misconfiguration behind the destruction | `unrestricted administrative access` |
| 3 | Identification | Carl's role | `developer` |
| 3 | Identification | Group `ci-deployer` belongs to | `Deployers` |
| 4 | Remediation | S3 actions in the policy (alphabetical, no `s3:`) | `getobject,listbucket` |
| 4 | Remediation | Simulator result for `s3:DeleteBucket` | `implicitDeny` |
| 5 | Build It Securely | Mechanism that caps max permissions | `Permission Boundaries` |
| 6 | Conclusion | I will scope it now | *(no answer)* |

## Lessons learned

- **Never grant persistent admin to a human.** Carl's `*:*` was one leaked credential away from a Code Spaces repeat.
- **Attach via groups, not directly.** Direct attachment doesn't scale and makes offboarding error-prone.
- **Audit inline policies and tags too** — inline policies hide from the managed-policy list, and names can lie; read the content.
- **Permission boundaries are the seatbelt.** Even a future mistake that over-grants gets clamped to the boundary's intersection.

---

<p align="center">
  <a href="README.md">
    <img src="more_writeups.svg" width="360" alt="Back to AWS Security Path">
  </a>
</p>
