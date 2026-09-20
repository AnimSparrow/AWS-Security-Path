<p align="center">
  <img src="assets/banner-exposed-port.svg" width="820" alt="The Exposed Port">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/EASY-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/EC2-1a0633?style=for-the-badge&labelColor=A479C4">
  <img src="https://img.shields.io/badge/SSM-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

## Overview

> [!NOTE]
> **TL;DR** — A "temporary" port 22 opened during a late-night deploy is never closed; three days later GuardDuty fires `SSHBruteForce`. You confirm SSM is already available, remove the public SSH rule without losing access, and launch a new instance with **no inbound management port at all**. Framed by the 2020 TeamTNT campaign.

## Real-world incident — TeamTNT (2020)

An automated campaign continuously swept cloud IP ranges for **TCP/22 and TCP/3389**, brute-forced weak/default credentials, and on success harvested AWS access keys from env vars and `~/.aws/credentials`, pivoted across the account (S3, other EC2), then deployed a cryptominer and a **backdoor** for persistence. Targets weren't chosen for their data — they were ordinary workloads that shared one trait: **port 22 open to the whole internet**.

> [!IMPORTANT]
> The core failure is an **exposed management port** plus **no enforced access path**: with SSM never configured, SSH was the only option and the open port had a permanent excuse. Modern AWS needs *no* inbound management ports.

## Identification

```bash
# find the instance's SG and inspect inbound
SG_ID=$(aws ec2 describe-instances --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].SecurityGroups[0].GroupId" --output text)
aws ec2 describe-security-groups --group-ids "$SG_ID" --query "SecurityGroups[0].IpPermissions"
#   → TCP/22 from 0.0.0.0/0

# CRITICAL pre-check: is SSM already online? (so closing 22 won't lock you out)
aws ssm describe-instance-information --filters "Key=InstanceIds,Values=$INSTANCE_ID" \
  --query "InstanceInformationList[0].{Ping:PingStatus,Platform:PlatformName,Agent:AgentVersion}" --output table
#   → Ping: Online   = SSM works, the public SSH rule is redundant
```

**Findings:** SSH open to `0.0.0.0/0`, and SSM already `Online`. The public rule is pure attack surface — its blast radius is every credential that could be brute-forced or found on the box after entry.

## Remediation

```bash
# revoke the public SSH rule
aws ec2 revoke-security-group-ingress --group-id "$SG_ID" \
  --ip-permissions '[{"IpProtocol":"tcp","FromPort":22,"ToPort":22,"IpRanges":[{"CidrIp":"0.0.0.0/0"}]}]'
# confirm inbound is now empty
aws ec2 describe-security-groups --group-ids "$SG_ID" --query "SecurityGroups[0].IpPermissions"   # → [ ]
# prove access is retained — SSM still works
aws ssm start-session --target "$INSTANCE_ID"   # whoami → ssm-user
```

Closing port 22 doesn't cut access because SSM runs over the **AWS control plane**, not an inbound port — that's the whole point.

> [!WARNING]
> If the verifier FAILs with "public SSH rule still present," you're likely acting on a **stale `SG_ID`** from a previous CloudShell session. Always re-fetch the instance and SG IDs in a new session before revoking — env vars don't persist.

## Build it securely — no inbound from day one

Ask: admin path? (SSM — no inbound) · inbound rules? (none for management) · IAM role? (`AmazonSSMManagedInstanceCore`) · SSH key? (omit entirely).

```bash
# SG with no inbound rules
aws ec2 create-security-group --group-name "room41-no-inbound-sg-$(date +%s)" --vpc-id "$VPC_ID"
# launch: NO --key-name, NO public IP, WITH the SSM instance profile
aws ec2 run-instances --image-id "$AMI_ID" --instance-type t3.micro \
  --subnet-id "$SUBNET_ID" --security-group-ids "$SECURE_SG" \
  --iam-instance-profile Name="Room41ManagedInstanceProfile" --no-associate-public-ip-address
```

Omitting `--key-name` means the instance has **no SSH key pair** — unreachable by SSH from the start. Zero inbound + no public IP + SSM profile = manageable on day one with no brute-force surface.

> [!TIP]
> `--group-name "...-$(date +%s)"` gives a unique name that avoids an `already exists` clash on a resumed session.

## Task answers

| # | Task | Question | Answer |
|---|------|----------|--------|
| 1 | Introduction | Unscheduled activation | *(no answer)* |
| 2 | Real-World Incident | Port targeted for initial access | `22` |
| 2 | Real-World Incident | Program installed for persistence | `backdoor` |
| 3 | Identification | CIDR the TCP/22 rule is open to | `0.0.0.0/0` |
| 3 | Identification | Flag | `THM{NO_IDC_********}` |
| 4 | Remediation | Flag from the verifier | `THM{IRIS_******}` |
| 5 | Build It Securely | Flag from the secure build | `THM{BUGS_ON_A_**********}` |
| 6 | Conclusion | Base is secured | *(no answer)* |

## Lessons learned

- **No open management port is ever "temporary"** — scanners find it in minutes and probe until it's closed.
- **SSM Session Manager is the default access path:** no inbound port, no key pair, every session logged to CloudTrail.
- **Zero inbound management rules is the baseline** — the right number of public SSH/RDP rules is usually zero.
- **Detective controls back up preventive ones**, but GuardDuty's `SSHBruteForce` should be firing on a port that's already shut.

---

<p align="center">
  <a href="README.md">
    <img src="assets/more_writeups.svg" width="360" alt="Back to AWS Security Path">
  </a>
</p>
