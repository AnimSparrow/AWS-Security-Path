<p align="center">
  <img src="assets/banner-invisible-network.svg" width="820" alt="The Invisible Network">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/EASY-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/VPC_FLOW_LOGS-1a0633?style=for-the-badge&labelColor=A479C4">
  <img src="https://img.shields.io/badge/VISIBILITY-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

## Overview

> [!NOTE]
> **TL;DR** — A SOC alert fires, you open VPC Flow Logs to investigate… and there are none. You learn what Flow Logs capture (and don't), enable them at the VPC level with an IAM role and CloudWatch Logs destination, query with Logs Insights, and build a secure logging baseline with the `aws:SourceAccount` guard. Framed by the Marriott/Starwood breach that ran undetected for four years.

## Real-world incident — Marriott / Starwood (2014–2018)

Attackers breached Starwood in 2014. Marriott acquired Starwood in 2016 and **inherited the already-compromised network** with no network security review. For ~4 years attackers exfiltrated the SPG reservation database (names, passports, cards) over sustained outbound connections — traffic that Flow Logs would have shown, but **no logs were being collected**. Discovered September 2018 via an alert on an unusual DB query; disclosed November 2018.

> [!IMPORTANT]
> There was **no traffic baseline**, so there was no way to tell when abnormal activity began. Flow Logs from day one establish the baseline that enables **anomaly detection** — turning attacker dwell time from years into hours.

## What Flow Logs capture

Fields include `srcaddr`, `dstaddr`, `srcport`, `dstport`, `protocol` (6=TCP, 17=UDP, 1=ICMP), `packets`, `bytes`, `start`, `end`, **`action`** (ACCEPT/REJECT per SG+NACL), `log-status`.

- **Three levels:** ENI, Subnet, VPC.
- **Three destinations:** CloudWatch Logs (real-time query/alerting), S3 (cheap long-term), Kinesis Firehose (streaming to third parties).

> [!IMPORTANT]
> Flow Logs record **metadata only, not packet contents**. They exclude Amazon DNS, DHCP, traffic to the VPC router/reserved addresses, and **traffic to IMDS (`169.254.169.254`)** — so the credential theft in SCARLETEEL/Capital One wouldn't show there, but the *exfiltration to external IPs* would.

## Identification — confirm the gap

```bash
aws ec2 describe-flow-logs --filter "Name=resource-id,Values=$VPC_ID" --output table
#   empty table = no logging configured for this VPC
```

Two instances are running and generating traffic with zero network-layer visibility. Any connection — legitimate or malicious — leaves no trace.

## Remediation — enable Flow Logs → CloudWatch

```bash
# 1. log group + retention
aws logs create-log-group --log-group-name /vpc/net-lab-flow-logs
aws logs put-retention-policy --log-group-name /vpc/net-lab-flow-logs --retention-in-days 5
# 2. IAM role: trust vpc-flow-logs.amazonaws.com + a policy allowing logs:CreateLogStream/PutLogEvents/...
aws iam create-role --role-name net-lab-flow-logs-role --assume-role-policy-document file://trust.json --permissions-boundary $BOUNDARY
aws iam put-role-policy --role-name net-lab-flow-logs-role --policy-name FlowLogsToCloudWatch --policy-document file://perms.json
# 3. enable at VPC level, ALL traffic
ROLE_ARN=$(aws iam get-role --role-name net-lab-flow-logs-role --query "Role.Arn" --output text)
aws ec2 create-flow-logs --resource-type VPC --resource-ids $VPC_ID --traffic-type ALL \
  --log-destination-type cloud-watch-logs --log-group-name /vpc/net-lab-flow-logs --deliver-logs-permission-arn $ROLE_ARN
```

Use **`ALL`** traffic type for security monitoring — you want both ACCEPT and REJECT. Records appear after a **1–5 minute** delay.

**Useful Logs Insights queries:**

```
# rejected connections (scanning)
fields @timestamp, srcAddr, dstAddr, dstPort | filter action="REJECT" | sort @timestamp desc | limit 20
# top rejected sources by port (scan map)
fields srcAddr, dstPort | filter action="REJECT" | stats count(*) as rejectedFlows by srcAddr, dstPort | sort rejectedFlows desc
```

> [!TIP]
> An idempotent script (`2>/dev/null || true` on each create-*) survives a resumed CloudShell session cleanly. Note: since 15 Jun 2026 AWS may default to **Log Analytics** — disable "Opt in to Log Analytics" in Preferences to reach classic Logs Insights.

## Build it securely

Flow Logs are a **non-negotiable baseline** on every VPC. CloudWatch Logs for operational/real-time (14–30d), S3 for archival/compliance (90d+) — you can enable both. Scope the IAM role to the specific log group, and add the confused-deputy guard:

```json
"Condition": { "StringEquals": { "aws:SourceAccount": "ACCOUNT_ID" } }
```

Best-practice settings: `--traffic-type ALL`, `--max-aggregation-interval 60` (near-real-time vs the 10-min default), KMS-encrypt the log group in production, and keep a documented library of scanning / exfiltration / C2 queries.

## Task answers

| # | Task | Question | Answer |
|---|------|----------|--------|
| 1 | Introduction | Let the invisible be seen | *(no answer)* |
| 2 | Real-World Incident | Years of undetected access | `4` |
| 2 | Real-World Incident | Detection enabled by a baseline | `Anomaly detection` |
| 3 | Identification | Three levels for Flow Logs (alphabetical) | `ENI, SUBNET, VPC` |
| 3 | Identification | What Flow Logs capture about IP traffic | `Metadata` |
| 3 | Identification | Flag on the web server page | `THM{DARK_TEMPLAR_********}` |
| 4 | Remediation | Traffic filter for security monitoring | `all` |
| 4 | Remediation | Flag from the helper | `THM{OBSERVER_********}` |
| 5 | Build It Securely | Destination with near-real-time queries/alerting | `CloudWatch Logs` |
| 5 | Build It Securely | Trust-policy key preventing the confused deputy | `aws:SourceAccount` |
| 5 | Build It Securely | Flag from the helper | `THM{VOID_**********}` |
| 6 | Conclusion | Now we can see them | *(no answer)* |

## Lessons learned

- **Logging is not retrospective** — if Flow Logs weren't running during an incident, that evidence is gone for good. Enable from day one.
- **Capture ALL traffic:** REJECT-only misses accepted exfiltration, ACCEPT-only misses scanning. You need both.
- **Scope the IAM role and add `aws:SourceAccount`** to block the confused-deputy problem.
- **Flow Logs are metadata** — pair with VPC Traffic Mirroring when you need packet-level inspection.

---

<p align="center">
  <a href="README.md">
    <img src="assets/more_writeups.svg" width="360" alt="Back to AWS Security Path">
  </a>
</p>
