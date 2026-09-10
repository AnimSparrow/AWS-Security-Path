<p align="center">
  <img src="assets/banner-silence-of-the-iams.svg" width="820" alt="The Silence of the IAMs">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/EASY-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/DETECTION-1a0633?style=for-the-badge&labelColor=A479C4">
  <img src="https://img.shields.io/badge/CLOUDTRAIL-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

## Overview

> [!NOTE]
> **TL;DR** — The detection room. After a foothold, attackers persist through IAM changes (new keys, policies attached to low-priv users, new users). You hunt an already-planted change in CloudTrail, revert it, then build a full IAM monitoring baseline: EventBridge → SNS alerting, a dedicated log-tampering rule, and a CloudWatch burst alarm. Framed by the Ubiquiti insider breach.

## Real-world incident — Ubiquiti (2020)

An insider — a senior engineer — used his own credentials to reach an admin key, masked himself behind a Surfshark VPN, and over two weeks cloned 1,000+ GitHub repos, pulled secrets from Secrets Manager, and downloaded 1,400+ config files. To cover his tracks he **reduced CloudWatch log retention to one day** so records auto-deleted. He then posed as an external hacker and demanded ransom — and was caught only when a VPN disconnect briefly exposed his home IP in the access logs. Six years, federal prison.

> [!IMPORTANT]
> CloudTrail was **enabled** the whole time — but nobody was watching. The failures were all detective: no alerts on high-risk IAM calls, undetected log tampering, no anomaly alerting on bulk Secrets Manager access. Logging without alerting is silence. Hence the room's whole thesis.

## High-risk IAM events to watch

| Event | Risk | Why |
|---|---|---|
| `AttachUserPolicy` / `AttachRolePolicy` / `AttachGroupPolicy` | Critical | instant privesc (e.g. attaching `AdministratorAccess`) |
| `PutUserPolicy` / `PutRolePolicy` | Critical | inline policy — harder to discover |
| `CreateUser` | High | persistence |
| `CreateAccessKey` | High | new programmatic creds, no console login |
| `CreateLoginProfile` | High | adds console access to a programmatic-only user |
| `UpdateAssumeRolePolicy` | High | cross-account persistence |
| `DeleteTrail` / `StopLogging` / `PutRetentionPolicy` | Critical | defense evasion (the Ubiquiti move) |
| `DeactivateMFADevice` | High | weakens account access |

## Identification — hunt the change in CloudTrail

```bash
# confirm the trail is actually logging
aws cloudtrail get-trail-status --name lab-audit-trail    # IsLogging: true

# targeted hunts
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=AttachUserPolicy
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=CreateAccessKey

# broad IAM write sweep
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=iam.amazonaws.com \
  --query "Events[?contains(EventName,'Attach')||contains(EventName,'Create')||contains(EventName,'Put')||contains(EventName,'Update')||contains(EventName,'Delete')]"
```

**Found:** `AdministratorAccess` attached **directly** to the low-priv user `app-deployer`, plus a fresh access key created for it. Both events were recorded — and silent.

> [!TIP]
> When reading an event, note **`userIdentity`** (who), **`requestParameters`** (what), **`sourceIPAddress`** (where), **`eventTime`** (when). And remember: **IAM is global — its events land in `us-east-1`** regardless of your working region.

## Remediation — revert, then build the alerting pipeline

```bash
# revert the attack
aws iam detach-user-policy --user-name app-deployer --policy-arn arn:aws:iam::aws:policy/AdministratorAccess
aws iam update-access-key  --user-name app-deployer --access-key-id $ROGUE_KEY_ID --status Inactive
aws iam delete-access-key  --user-name app-deployer --access-key-id $ROGUE_KEY_ID

# CloudTrail → EventBridge → SNS pipeline
SNS_ARN=$(aws sns create-topic --name iam-change-alerts --query TopicArn --output text)
aws events put-rule    --name iam-high-risk-changes --event-pattern file://iam-event-pattern.json --state ENABLED
aws events put-targets --rule iam-high-risk-changes --targets "Id=sns-iam-alerts,Arn=$SNS_ARN"
# + SNS topic policy allowing events.amazonaws.com to Publish
```

## Build it securely — an IAM monitoring baseline

**Classify by severity, route separately, add burst detection.**

- **Critical** (immediate): Attach*/Put* policy, `CreateUser`, `UpdateAssumeRolePolicy`, `DeleteTrail`/`StopLogging`, `PutRetentionPolicy`
- **High** (1–2h): `CreateAccessKey`, `CreateLoginProfile`, `DeactivateMFADevice`, Detach/DeleteUserPolicy
- **Medium** (24h): `CreateRole`, `CreateGroup`, `AddUserToGroup`

```bash
# per-severity EventBridge rules → different channels (Critical→all, High→Slack, Medium→email)
aws events put-rule --name iam-critical-changes --event-pattern file://iam-critical-pattern.json --state ENABLED

# dedicated log-tampering rule (defense-evasion detection)
#   DeleteTrail, StopLogging, UpdateTrail, PutEventSelectors
aws events put-rule --name audit-trail-tampering --event-pattern file://audit-tampering-pattern.json --state ENABLED

# CloudWatch: metric filter + burst alarm (>3 high-risk IAM events / 5 min)
aws logs put-metric-filter --log-group-name aws-cloudtrail-logs --filter-name IAMWriteEventCount \
  --filter-pattern '{ ($.eventSource="iam.amazonaws.com") && (($.eventName="AttachUserPolicy")||($.eventName="CreateUser")|| ...) }' \
  --metric-transformations metricName=IAMHighRiskEventCount,metricNamespace=SecurityMetrics,metricValue=1,defaultValue=0
aws cloudwatch put-metric-alarm --alarm-name iam-change-burst --metric-name IAMHighRiskEventCount \
  --namespace SecurityMetrics --statistic Sum --period 300 --threshold 3 \
  --comparison-operator GreaterThanThreshold --evaluation-periods 1 --alarm-actions $SNS_ARN
```

> [!IMPORTANT]
> Two complementary layers: **EventBridge** fires per-event (deterministic, immediate), while the **CloudWatch alarm** catches *bursts* — "five IAM changes in a minute" — even when each call looks individually fine. The dedicated **tampering rule** is the direct answer to what Ubiquiti missed.

## Task answers

| # | Task | Question | Answer |
|---|------|----------|--------|
| 1 | Introduction | Something feels off... | *(no answer)* |
| 2 | Real-World Incident | How the attacker removed evidence | `log tampering` |
| 3 | Identification | User that created an access key (no account-ID suffix) | `Room24-RogueKeyCreator` |
| 3 | Identification | Source of the AttachUserPolicy event | `iam.amazonaws.com` |
| 4 | Remediation | Flag | `THM{SILENT_CHANGES_********}` |
| 5 | Build It Securely | Flag | `THM{WATCHING_THE_*******}` |
| 6 | Conclusion | You cannot silence the IAMs | *(no answer)* |

## Lessons learned

- **Logging ≠ detection.** CloudTrail records everything; without EventBridge/SNS/CloudWatch on top, no one hears the alarm — exactly Ubiquiti's gap.
- **IAM persistence is the attacker's first move.** New keys, direct `AdministratorAccess` on a low-priv user, and new users are the signatures to alert on.
- **Log tampering is its own high-severity signal.** `StopLogging` / `DeleteTrail` / retention cuts deserve a dedicated rule.
- **Deterministic + behavioural together.** Pair EventBridge's exact-match alerting with a CloudWatch burst alarm (and GuardDuty) for defense in depth.

---

<p align="center">
  <a href="README.md">
    <img src="assets/more_writeups.svg" width="360" alt="Back to AWS Security Path">
  </a>
</p>
