<p align="center">
  <img src="assets/banner-forgotten-nacl.svg" width="820" alt="The Forgotten NACL">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/EASY-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/NACL-1a0633?style=for-the-badge&labelColor=A479C4">
  <img src="https://img.shields.io/badge/DEFENSE_IN_DEPTH-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

## Overview

> [!NOTE]
> **TL;DR** — Security Groups are tight, but both subnets still use the default "allow all" NACL, so the subnet boundary provides zero filtering. You learn how NACLs differ from SGs (stateless, subnet-level, ordered, can deny), replace the defaults with scoped custom NACLs — remembering ephemeral ports — and design a layered model. Framed by the 2023 SCARLETEEL campaign.

## Real-world incident — SCARLETEEL (2023)

Attackers exploited a public-facing app in a Kubernetes pod and dropped an XMRig miner as a decoy. From the pod they hit **IMDSv1** (no hop-limit) and stole the node's IAM role credentials, enumerated AWS, and found **Terraform state files in S3 containing plaintext IAM credentials** for a second account — which they used to pivot, exfiltrate source code, and then turn off CloudTrail in both accounts.

> [!IMPORTANT]
> Both subnets used the **default "allow all" NACL**. A NACL blocking outbound to `169.254.169.254` or restricting cross-subnet movement would have sharply limited the reach. Combine with IMDSv2 (hop-limit 1) and no plaintext creds in state files, and the chain breaks in several places.

## NACL vs Security Group

| | Security Group | NACL |
|---|---|---|
| Level | Instance (ENI) | **Subnet** |
| Stateful? | Yes | **No** — needs inbound *and* outbound rules |
| Rules | Allow only | Allow **and** Deny |
| Evaluation | all at once | **numbered order, first match wins** |
| Default | deny inbound | **allow all** (the trap) |

## Identification — spot the default NACL

```bash
aws ec2 describe-network-acls --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "NetworkAcls[*].{ID:NetworkAclId,IsDefault:IsDefault,Subnets:length(Associations),Entries:Entries[*].{Rule:RuleNumber,Action:RuleAction,CIDR:CidrBlock}}"
#   IsDefault: true, associated with BOTH subnets
#   Rule 100   → allow 0.0.0.0/0 (in + out)
#   Rule 32767 → deny  0.0.0.0/0 (the implicit *, never reached — 100 matches first)
```

**Findings:** one default NACL bound to **2 subnets**. Rule **100** ("allow all") matches every packet, so the reserved deny rule **32767** never fires. SGs are correctly scoped, but at the subnet boundary there is no second checkpoint — lateral movement inside the VPC meets zero resistance.

> [!TIP]
> **32767** is the highest possible NACL rule number — the reserved `*` deny. Custom rules use 1–32766, evaluated ascending, first match wins. Leave gaps (100, 110, 120) so you can insert later.

## Remediation — custom NACLs with ephemeral ports

```bash
# PUBLIC NACL — inbound: 80, 443, and 1024-65535 (return traffic!); outbound: 80, 443, 8080→private, 1024-65535
aws ec2 create-network-acl-entry --network-acl-id $PUBLIC_NACL_ID --rule-number 100 --protocol tcp --port-range From=80,To=80    --cidr-block 0.0.0.0/0   --rule-action allow --ingress
aws ec2 create-network-acl-entry --network-acl-id $PUBLIC_NACL_ID --rule-number 200 --protocol tcp --port-range From=1024,To=65535 --cidr-block 0.0.0.0/0 --rule-action allow --ingress   # ephemeral
# PRIVATE NACL — inbound: 8080 from public subnet + ephemeral from VPC; outbound: ephemeral→public + 443 in VPC
aws ec2 create-network-acl-entry --network-acl-id $PRIVATE_NACL_ID --rule-number 100 --protocol tcp --port-range From=8080,To=8080 --cidr-block 10.0.1.0/24 --rule-action allow --ingress
# then swap the associations off the default NACL
aws ec2 replace-network-acl-association --association-id $ASSOC --network-acl-id $NACL
```

Because NACLs are **stateless**, every allowed flow needs *two* rules — the request direction and a return-direction rule on the **ephemeral range 1024–65535**. Forget either and connections silently fail despite "correct" service-port rules. After remediation the default NACL has **0 associations**.

## Build it securely

Ask: what needs internet? what flows exist between subnets (each = 2 rules, request + return)? what outbound do private resources need (NAT, not IGW)? what should be completely blocked? The private NACL has **no inbound `0.0.0.0/0`** — the implicit `*` (32767) denies anything outside the VPC CIDR.

> [!WARNING]
> A verifier that checks resources by **tag** can FAIL even when the NACLs are correct — if the NACLs have no `Name` tag it expects. Tag each custom NACL (`create-tags` keyed by which subnet it's associated with), and confirm you're using the right VPC ID after a fresh CloudShell session. Two classic automation traps: missing tags and stale IDs.

## Task answers

| # | Task | Question | Answer |
|---|------|----------|--------|
| 1 | Introduction | It's NACL Day. Again | *(no answer)* |
| 2 | Real-World Incident | Service used to steal IAM credentials | `IMDS` |
| 2 | Real-World Incident | S3 files holding second-account credentials | `Terraform state files` |
| 3 | Identification | Rule number that denies all traffic | `32767` |
| 3 | Identification | Subnets on the default NACL | `2` |
| 3 | Identification | Flag from the web server | `THM{WHAT_****}` |
| 4 | Remediation | Property forcing explicit ephemeral rules | `Stateless` |
| 4 | Remediation | Flag from the helper | `THM{NO_MORE_*******}` |
| 5 | Build It Securely | Ephemeral port range (Linux) | `1024-65535` |
| 5 | Build It Securely | Flag from the secure-build verifier | `THM{TODAY_IS_*********}` |
| 6 | Conclusion | Today is different | *(no answer)* |

## Lessons learned

- **Never leave the default "allow all" NACL** on a real subnet — replace it so the boundary actually filters.
- **NACLs are stateless:** every flow needs a request rule *and* an ephemeral (1024–65535) return rule, in both directions.
- **Number with gaps and remember first-match-wins** — ordering is the whole ballgame.
- **Private subnets get no inbound `0.0.0.0/0`**, and NACLs' explicit **deny** lets you block CIDRs and lateral movement that SGs can't express.

---

<p align="center">
  <a href="README.md">
    <img src="assets/more_writeups.svg" width="360" alt="Back to AWS Security Path">
  </a>
</p>
