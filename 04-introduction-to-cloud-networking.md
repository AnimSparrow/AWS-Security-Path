<p align="center">
  <img src="assets/banner-cloud-networking.svg" width="820" alt="Introduction to Cloud Networking">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/EASY-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/VPC-1a0633?style=for-the-badge&labelColor=A479C4">
  <img src="https://img.shields.io/badge/NETWORKING-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

## Overview

> [!NOTE]
> **TL;DR** — The AWS networking building blocks: VPCs, subnets (public vs private), route tables, gateways (IGW/NAT), and the two access-control layers (Security Groups vs NACLs). Ends by tracing a packet's full journey and pulling a flag off a lab web server.

## Notes / key concepts

**VPC** — your own isolated network in the cloud. Lives in a **single region**, spans multiple Availability Zones, defined by a **CIDR block** (e.g. `10.0.0.0/16`) that you carve into subnets. Every account has a default VPC per region; for production you build your own.

**Subnets & routing:**

| Subnet | Internet route | Reachable from internet? |
|---|---|---|
| **Public** | route to **Internet Gateway** (`0.0.0.0/0 → igw-...`) | yes (with public IP) |
| **Private** | route to **NAT Gateway** (outbound only) | no |

Each subnet is tied to a **route table**. The `local` route keeps VPC-internal traffic inside; the `0.0.0.0/0` catch-all sends the rest outward (IGW for public, NAT for private). Route table IDs start with `rtb-`.

**Gateways:** IGW = bidirectional internet for public subnets; NAT = outbound-only for private subnets.

**Security Groups vs NACLs** — the two most confused controls in AWS:

| | Security Group | NACL |
|---|---|---|
| Level | Instance (ENI) | Subnet |
| Stateful? | **Yes** — reply traffic auto-allowed | **No** — needs explicit in *and* out rules |
| Rules | Allow only | Allow **and** Deny |
| Evaluation | all rules together | numbered order |
| Default inbound | deny all | allow all |

> [!IMPORTANT]
> **Stateful vs stateless is the whole distinction.** An SG remembers the connection, so an inbound allow covers the response automatically — you only need inbound rules. A NACL forgets, so you must open **both directions**. Layer them: tight SGs on instances, NACLs as a subnet-wide safety net.

## Packet flow (the money diagram)

Inbound request from the internet to the web server:

```text
Internet → IGW → VPC router → route table → NACL (inbound) → SG → ENI → instance
                                    ↑ reply path ↓
              response ← NACL (outbound, stateless) ← SG (stateful)
```

The reply is where stateful/stateless bites: the **SG lets the response back automatically**, but the **NACL still evaluates its outbound rules** independently.

## Lab — the two mini-games + live enumeration

**Mini-game (VPC Network Builder), 2 levels:**
- **L1** — place components in order: `IGW → Route Rule → NACL → SG → EC2`, per subnet.
- **L2** — configure to least privilege:
  - Public route: `0.0.0.0/0 → Internet Gateway`
  - Private route: `local only` (no default route)
  - Web SG: `HTTP :80 from 0.0.0.0/0`
  - Private SG: `SSH :22 from 172.16.1.0/24 only` (managed from the public subnet, not the internet)
  - NACLs: **both** inbound & outbound (stateless)

**Live web server enumeration:**

```bash
LAB_VPC=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=false" \
  --query "Vpcs[0].VpcId" --output text)

aws ec2 describe-instances \
  --filters "Name=vpc-id,Values=$LAB_VPC" "Name=instance-state-name,Values=running" \
  --query "Reservations[*].Instances[*].{ID:InstanceId,PublicIP:PublicIpAddress,Name:Tags[?Key=='Name']|[0].Value}" \
  --output table

curl http://<public-ip>          # web page serves the flag
```

## Task answers

| # | Task | Question | Answer |
|---|------|----------|--------|
| 1 | Introduction | Ready | *(no answer)* |
| 2 | VPC Service | VPC name | `lab-vpc` |
| 2 | VPC Service | Lab VPC CIDR block | `172.16.0.0/16` |
| 3 | Subnet and Routing | Public subnet CIDR | `172.16.1.0/24` |
| 3 | Subnet and Routing | Usual route ID prefix | `rtb` |
| 4 | Security Groups & NACLs | Which control is stateful | `Security Group` |
| 4 | Security Groups & NACLs | Inbound ports on `lab-web-sg` (ascending) | `22,80,443` |
| 5 | Network Flow | Mini-game flag | `THM{AWS_NETWORKING_***}` |
| 5 | Network Flow | Web server flag | `THM{TANGLED_****}` |
| 6 | Conclusion | Well-segmented VPCs | *(no answer)* |

## Lessons learned

- **Stateful (SG) vs stateless (NACL) drives everything.** Forget it and you'll either lock yourself out (missing NACL outbound) or leave a hole open.
- **Segmentation limits blast radius.** Public/private split + least-privilege SGs means a compromised web box can't freely reach the private tier.
- **"Managed from the public subnet only" ≠ "open to the world".** Scoping SSH to `172.16.1.0/24` instead of `0.0.0.0/0` is the difference between a bastion pattern and an exposed instance.
- **Avoid catch-all `0.0.0.0/0` and port ranges** unless the traffic genuinely needs it. Port-specific entries are the least-privilege default.

---

<p align="center">
  <a href="README.md">
    <img src="assets/more_writeups.svg" width="360" alt="Back to AWS Security Path">
  </a>
</p>
