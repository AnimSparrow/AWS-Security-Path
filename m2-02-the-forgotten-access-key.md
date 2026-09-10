<p align="center">
  <img src="assets/banner-forgotten-access-key.svg" width="820" alt="The Forgotten Access Key">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/EASY-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/IAM-1a0633?style=for-the-badge&labelColor=A479C4">
  <img src="https://img.shields.io/badge/CREDENTIAL_HYGIENE-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

## Overview

> [!NOTE]
> **TL;DR** — An engineer's access key gets created for a deploy script and forgotten: still active, never rotated, sitting in a config file. You audit with the IAM credential report, find a stale never-used key and a user with no MFA, run the full remediation cycle (deactivate → wait → delete → rotate → MFA), then rebuild access properly with STS temporary credentials. Framed by the Uber 2016 breach.

## Real-world incident — Uber (2016)

A clean chain of credential failures: credential stuffing into GitHub (no MFA) → **AWS access keys hardcoded in a private repo** → authenticate to AWS → download 16 unencrypted DB backups from S3 → ransom, paid and covered up by the CSO.

> [!IMPORTANT]
> Every link was a credential-hygiene failure: long-lived keys with no rotation, keys in source code, no MFA, over-privileged access, no anomaly monitoring (nobody alerted on a bulk S3 download). The fix for the hardcoded-keys link is **Secrets Manager** (or, better, role-based temporary credentials).

## Identification — the credential report

The IAM credential report is the fast audit: a CSV snapshot of every user's credential state.

```bash
aws iam generate-credential-report
aws iam get-credential-report --query 'Content' --output text | base64 --decode > credential-report.csv
# key columns: mfa_active, access_key_N_active, _last_used_date, _last_rotated
```

Then drill the suspect user:

```bash
aws iam list-access-keys        --user-name dev-keyleaks    # TWO active keys
aws iam get-access-key-last-used --access-key-id $KEY2_ID   # no LastUsedDate = never used
aws iam list-mfa-devices        --user-name dev-keyleaks    # [] = no MFA
aws iam list-groups-for-user    --user-name dev-keyleaks    # AppDataReaders → scoped S3 read
```

**Findings:** `dev-keyleaks` has **no MFA**, a **stale key that was never used but is still active**, and read access to one S3 bucket (blast radius limited — but an attacker could silently read the whole bucket).

> [!TIP]
> Triage rules worth memorising: **age** — keys older than 90 days, never rotated; **usage gap** — keys unused 30+ days should be disabled; **monitoring** — use **CloudTrail** to alert on key-usage anomalies.

## Remediation — the safe cycle

Order matters: **deactivate → wait → delete** (never delete straight away — deactivation is reversible, deletion is not).

```bash
# 1. Deactivate the stale key (reversible: flip back to Active if something breaks)
aws iam update-access-key --user-name dev-keyleaks --access-key-id $KEY2_ID --status Inactive
# 2. Wait 24-72h, watch CloudTrail for calls using it
# 3. Delete (irreversible)
aws iam delete-access-key --user-name dev-keyleaks --access-key-id $KEY2_ID
# 4. Rotate the active key: create new → test → deactivate+delete old
aws iam create-access-key --user-name dev-keyleaks           # save the secret ONCE
aws sts get-caller-identity --profile dev-keyleaks           # verify new key works
aws iam update-access-key --user-name dev-keyleaks --access-key-id $KEY1_ID --status Inactive
aws iam delete-access-key --user-name dev-keyleaks --access-key-id $KEY1_ID
# 5. Enable MFA
aws iam enable-mfa-device --user-name dev-keyleaks --serial-number ...:mfa/dev-keyleaks-mfa \
  --authentication-code1 <c1> --authentication-code2 <c2>
```

> [!WARNING]
> **Gotcha I hit:** the verifier Lambda failed with `key_check: FAIL — expected 1 active key, found 2`. Rotation had created a new key while an old one was still active. The rule is **max 1 active key per user** (2 only *during* the rotation window). Deactivate + delete the leftover and it passes:
> ```
> active_key_count: 1, key_check: PASS, mfa_check: PASS, status: PASS
> ```
> Also saw `EntityAlreadyExists` on `create-virtual-mfa-device` — the device already existed from an earlier attempt; MFA still verified PASS, so it's harmless.

## Build it securely — temporary credentials

The real fix isn't a *better* long-lived key, it's *no* long-lived key. A role issues short-lived STS credentials, gated by MFA and capped by a permission boundary.

```bash
# trust policy requires MFA: "aws:MultiFactorAuthPresent": "true"
aws iam create-role --role-name DevS3ReadRole \
  --assume-role-policy-document file:///tmp/trust-policy.json \
  --permissions-boundary arn:aws:iam::${ACCOUNT_ID}:policy/Room22-DevS3ReadBoundary   # boundary is mandatory
aws iam put-role-policy --role-name DevS3ReadRole --policy-name S3ReadAccess ...       # scoped S3 read

# user assumes the role → temporary creds (needs an MFA code)
aws sts assume-role --role-arn ...:role/DevS3ReadRole --serial-number ...:mfa/dev-keyleaks-mfa --token-code <MFA>
```

**Why STS beats long-lived keys:** auto-expire (default **1 hour**), useless once expired, and the session token makes calls auditable in CloudTrail.

**Credential decision flow** (priority order):
1. **IAM role** — EC2 instance profiles, Lambda/ECS roles
2. **STS assume-role** — human users, cross-account
3. **SSO / Identity Center** — console/CLI humans
4. **Long-lived keys** — last resort, only with full hygiene (rotation schedule: max age 90d, unused 30d → deactivate, max 1 key, automated audit via AWS Config)

## Task answers

| # | Task | Question | Answer |
|---|------|----------|--------|
| 1 | Introduction | Remember, remember! | *(no answer)* |
| 2 | Real-World Incident | Service to fix hardcoded keys | `Secrets Manager` |
| 3 | Identification | Service to monitor key usage | `CloudTrail` |
| 3 | Identification | Other S3 actions in the group policy (alphabetical) | `GetBucketLocation,ListAllMyBuckets` |
| 4 | Remediation | What to always do before deleting keys | `wait` |
| 4 | Remediation | Flag | `THM{QUICKLY_FIND_MY_*****}` |
| 5 | Build It Securely | Default temp-credential lifetime (hours) | `1` |
| 5 | Build It Securely | Guardrail for long-lived keys | `key rotation schedule` |
| 5 | Build It Securely | Flag in the reports | `THM{FOUND_YOUR_***}` |
| 5 | Build It Securely | Flag from the helper function | `THM{NO_KEYS_******}` |
| 6 | Conclusion | The best access key... | *(no answer)* |

## Lessons learned

- **The credential report is the first line of audit** — one CSV shows MFA state, key age, last-used, and last-rotated for every user.
- **Old keys are silent vulnerabilities.** A never-used active key is pure risk with zero benefit.
- **Deactivate → wait → delete, never delete first.** Deactivation is your rollback; deletion is permanent and breaks anything still using the key.
- **Prefer temporary credentials.** STS expires automatically and is auditable; the best long-lived key is the one that doesn't exist.

---

<p align="center">
  <a href="README.md">
    <img src="more_writeups.svg" width="360" alt="Back to AWS Security Path">
  </a>
</p>
