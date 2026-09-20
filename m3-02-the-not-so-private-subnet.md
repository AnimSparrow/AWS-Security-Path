<p align="center">
  <img src="assets/banner-not-so-private-subnet.svg" width="820" alt="The Not So Private Subnet">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/EASY-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/ROUTE_TABLES-1a0633?style=for-the-badge&labelColor=A479C4">
  <img src="https://img.shields.io/badge/SEGMENTATION-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

## Overview

> [!NOTE]
> **TL;DR** — A subnet named `backend-private` turns out to be publicly routable: it shares a route table that has a `0.0.0.0/0 → IGW` route, and its instance has a public IP. A subnet is private because of its **routing**, not its name. You confirm the exposure, fix it with a dedicated private route table, disable public-IP auto-assign, and rebuild a properly segmented VPC. Framed by the 2018 Tesla cloud breach.

## Real-world incident — Tesla (2018)

Tesla ran an internal Kubernetes dashboard **reachable from the public internet with no authentication**. Inside it, attackers found a pod holding AWS credentials, used them to deploy cryptomining on Tesla's own compute (hidden behind CloudFlare, CPU kept low to dodge alerts), and reached an S3 bucket of vehicle telemetry and engineering data. RedLock found it during routine internet scanning — not Tesla's own monitoring.

> [!IMPORTANT]
> A properly isolated private subnet with **no IGW route** would have made that dashboard undiscoverable by external scanners. The failed principle is **network segmentation**: network isolation is the first line of defence, authentication the second — here both were missing.

## Identification — is "private" actually private?

```bash
# subnet settings — watch MapPublicIpOnLaunch
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" \
  --query "Subnets[*].{Name:Tags[?Key=='Name']|[0].Value,CIDR:CidrBlock,PublicIP:MapPublicIpOnLaunch,SubnetId:SubnetId}" --output table
#   backend-private → PublicIP: True   ← a private subnet must never auto-assign public IPs

# the route table bound to the backend subnet — look for an IGW route
aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=$BACKEND_SUBNET" \
  --query "RouteTables[].{Name:Tags[?Key=='Name']|[0].Value,Routes:Routes[].{Dest:DestinationCidrBlock,Target:GatewayId}}"
#   0.0.0.0/0 → igw-...   ← this makes the subnet public

# prove it: the "internal" API answers from the internet
curl -s http://$INTERNAL_API_IP:8080     # returns JSON + flag
```

**Root cause:** both subnets share `room32-shared-rt`, which carries `0.0.0.0/0 → IGW`. A dedicated private route table was never created — "private" lived only in the name.

> [!TIP]
> If `--filters "Name=association.subnet-id,..."` returns `None` for the name, list all route tables in the VPC (`describe-route-tables --filters "Name=vpc-id,..."`) and match by association — the shared RT is the one carrying the IGW route.

## Remediation

```bash
# 1. dedicated private route table — local route only, no IGW
PRIVATE_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
  --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=room32-private-rt}]" \
  --query "RouteTable.RouteTableId" --output text)
# 2. swap the backend subnet's association onto it
ASSOC_ID=$(aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=$BACKEND_SUBNET" \
  --query "RouteTables[0].Associations[?SubnetId=='$BACKEND_SUBNET'].RouteTableAssociationId" --output text)
aws ec2 replace-route-table-association --association-id $ASSOC_ID --route-table-id $PRIVATE_RT
# 3. stop auto-assigning public IPs on future launches
aws ec2 modify-subnet-attribute --subnet-id $BACKEND_SUBNET --no-map-public-ip-on-launch
# 4. verify: curl from the internet times out; curl from the web server (SSM) over the private IP still works
```

> [!IMPORTANT]
> `--no-map-public-ip-on-launch` only affects **future** launches — the existing instance keeps its public IP. But once the IGW route is gone, internet traffic can no longer reach it: **the routing path no longer exists**. Routing beats the public-IP attribute. The `local` route stays in both tables, so intra-VPC traffic (web → API) is preserved.

## Build it securely

Before building, ask: what needs internet (public subnet)? what is internal-only (private)? what outbound do private resources need (NAT, not IGW)? how many AZs (prod ≥2)?

```bash
# public subnet: auto-assign public IP ON, route table WITH 0.0.0.0/0 → IGW
aws ec2 modify-subnet-attribute --subnet-id $PUB  --map-public-ip-on-launch
aws ec2 create-route --route-table-id $PUB_RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW
# private subnet: auto-assign OFF, route table with ONLY the local route (no 0.0.0.0/0)
aws ec2 modify-subnet-attribute --subnet-id $PRIV --no-map-public-ip-on-launch
```

The difference between public and private is exactly the presence of `0.0.0.0/0 → IGW`. A segmented VPC with one public and one private tier needs a **minimum of 2 route tables** — sharing one is precisely the bug from task 3.

## Task answers

| # | Task | Question | Answer |
|---|------|----------|--------|
| 1 | Introduction | Rise and shine | *(no answer)* |
| 2 | Real-World Incident | Internal tool exposed to the internet | `kubernetes dashboard` |
| 2 | Real-World Incident | Network principle not applied | `network segmentation` |
| 3 | Identification | Route table on the backend subnet | `room32-shared-rt` |
| 3 | Identification | Destination CIDR pointing to the IGW | `0.0.0.0/0` |
| 3 | Identification | Flag from probing the backend public IP | `THM{UNFORESEEN_************}` |
| 4 | Remediation | Subcommand that replaces a route-table association | `replace-route-table-association` |
| 4 | Remediation | Flag from the verifier | `THM{BLACK_MESA_*******}` |
| 5 | Build It Securely | Minimum route tables (1 public + 1 private) | `2` |
| 5 | Build It Securely | Subnet attribute to disable on private subnets | `MapPublicIpOnLaunch` |
| 5 | Build It Securely | Flag from the secure-build verifier | `THM{RIGHT_PLACE_RIGHT_*****}` |
| 6 | Conclusion | The right route table... | *(no answer)* |

## Lessons learned

- **A subnet's exposure is decided by its route table, not its name or tags.** IGW route present → public, full stop.
- **Never share a route table between public and private subnets** — each tier needs its own with only the routes it requires.
- **Disable `MapPublicIpOnLaunch` on every private subnet** so no future instance silently gets a public IP.
- **Routing is network-level; Security Groups are host-level.** Both layers matter — don't lean on SGs alone for isolation.

---

<p align="center">
  <a href="README.md">
    <img src="assets/more_writeups.svg" width="360" alt="Back to AWS Security Path">
  </a>
</p>
