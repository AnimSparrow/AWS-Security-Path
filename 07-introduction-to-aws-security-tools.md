<p align="center">
  <img src="banner-security-tools.svg" width="820" alt="Introduction to AWS Security Tools">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/EASY-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/OBSERVABILITY-1a0633?style=for-the-badge&labelColor=A479C4">
  <img src="https://img.shields.io/badge/BLUE_TEAM-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

## Overview

> [!NOTE]
> **TL;DR** — The four pillars of AWS security observability and how they chain together: **CloudTrail** (what happened), **GuardDuty** (something's wrong), **Inspector** (where the cracks are), **Detective** (solve the case). Hands-on: create a hardened CloudTrail trail, generate GuardDuty sample findings, run an Inspector scan, and launch a Detective investigation.

## The four pillars at a glance

| Service | Job | Feeds on | Answers |
|---|---|---|---|
| **CloudTrail** | API activity logging | — (it's the source) | *what happened, by whom, when* |
| **GuardDuty** | continuous threat detection | CloudTrail + VPC Flow + DNS logs | *is something wrong right now* |
| **Inspector** | vulnerability / config assessment | EC2, ECR, Lambda, CI/CD code | *where are the cracks (CVEs, exposure)* |
| **Detective** | investigation + visual analytics | GuardDuty + CloudTrail + VPC Flow | *full scope of an incident* |

> [!IMPORTANT]
> **The sequence is the point.** In a real incident: GuardDuty raises the alarm automatically → Detective builds the timeline to investigate the full scope. CloudTrail is the evidence underneath both; Inspector is the proactive side, finding weaknesses before they're exploited.

## Notes / key concepts

**CloudTrail** — records every API call (programmatic *and* console), stored in S3 and/or CloudWatch Logs. Foundation for monitoring, IR, and compliance (PCI-DSS, SOC 2). Key points: a trail is per-region but can be **multi-region**; default event history is **90 days** (use S3/CloudWatch for longer); logs are signed and support **log-file integrity validation**; the S3 bucket needs a policy granting CloudTrail write access.

**GuardDuty** — continuous detection using ML + threat intel feeds. Detector is **per-region**. Auto-ingests CloudTrail management events, VPC Flow Logs, DNS query logs. Findings carry **severity: Low / Medium / High**, the affected resource, and the actor (IP, IAM identity). Often centrally managed by a delegated admin (security account).

**Inspector** — automated assessment of EC2, ECR images, Lambda, and CI/CD code. Auto-discovers new resources and re-evaluates on change. Combines CVE data with real-time environmental factors (network reachability, exploitability) into a prioritized **risk score**. Findings include CVE IDs, severity, and remediation guidance.

**Detective** — builds a **behavior graph** (per-region) from GuardDuty + CloudTrail + VPC Flow Logs. Takes **24–48 hours** to build a meaningful baseline. Investigate via entity profiles (IPs, IAM users/roles, EC2) and manual investigations by ARN. Most useful when GuardDuty already has findings to chase.

## Lab — configuration highlights

**CloudTrail — hardened, multi-region trail via CLI:**

```bash
aws s3 ls          # find security-lab-cloudtrail-<id>-us-east-1
aws cloudtrail create-trail --name SecurityLabTrail \
  --s3-bucket-name "security-lab-cloudtrail-<id>-us-east-1" \
  --is-multi-region-trail --enable-log-file-validation
aws cloudtrail start-logging   --name SecurityLabTrail
aws cloudtrail get-trail-status --name SecurityLabTrail   # IsLogging: true
```

Best practices baked in: multi-region, log-file validation, SSE-KMS encryption, and a lifecycle rule to expire old logs.

**GuardDuty — generate sample findings:**

```bash
DETECTOR_ID=$(aws guardduty list-detectors --region us-east-1 --query 'DetectorIds[0]' --output text)
aws guardduty create-sample-findings --detector-id "$DETECTOR_ID" --region us-east-1 \
  --finding-types "Recon:EC2/PortProbeUnprotectedPort" \
                  "UnauthorizedAccess:IAMUser/ConsoleLoginSuccess.B"
```

**Inspector** — enable, activate EC2 scanning, wait ~3–5 min, then read findings *by instance*. The lab EC2 flags ports **22 (SSH)** and **80 (HTTP)** under **Network Reachability**.

**Detective** — enable the behavior graph, then run an investigation against your own ARN:

```bash
aws sts get-caller-identity   # grab the Arn, run investigation on the user (or a role)
```

## Mini-game — "Defending AWS: Match the Service" (3 levels)

- **L1 — match example to service:** "which IAM user called TerminateInstances, and when" → **CloudTrail**; "repeated logins from a known-bad IP, alert me" → **GuardDuty**; "visual timeline of what a flagged user did over 48h" → **Detective**.
- **L2 — service ↔ capability:** CloudTrail = *records every API call for audit/compliance*; GuardDuty = *threat intel + ML to detect malicious activity*; Inspector = *scans workloads for vulns and network exposure*.
- **L3 — services that work together:** initial alarm → **GuardDuty**; investigate full scope → **Detective**.

## Task answers

| # | Task | Question | Answer |
|---|------|----------|--------|
| 1 | Introduction | Feed me knowledge | *(no answer)* |
| 2 | CloudTrail | What CloudTrail records | `API activity` |
| 2 | CloudTrail | Name of the other trail in the dashboard | `thm-org-trail` |
| 3 | GuardDuty | One GuardDuty event source | `CloudTrail` |
| 3 | GuardDuty | The three severity levels | `Low, Medium, High` |
| 4 | Inspector | The two vulnerable ports (ascending) | `22,80` |
| 4 | Inspector | Vulnerability type they share | `Network Reachability` |
| 5 | Detective | Other IAM resource type in the dropdown | `aws role` |
| 5 | Detective | Min. time to build a baseline (hours) | `24` |
| 6 | Conclusion | Mini-game flag | `THM{DEFENCE_IN_*****}` |

## Lessons learned

- **Each tool does one job — mixing them up is the #1 IR mistake.** CloudTrail records, GuardDuty detects, Inspector assesses, Detective investigates.
- **Detection is only half of it.** GuardDuty tells you *something* happened; Detective's graph is what turns a single finding into the full incident scope.
- **CloudTrail is the bedrock.** Without a multi-region, validated, encrypted trail, both GuardDuty and Detective are working with less evidence — and so are you.
- **Inspector is the proactive pillar.** Network Reachability on 22/80 is exactly the kind of exposure you want flagged *before* GuardDuty sees someone probing it.

---

<p align="center">
  <a href="README.md">
    <img src="more_writeups.svg" width="360" alt="Back to AWS Security Path">
  </a>
</p>
