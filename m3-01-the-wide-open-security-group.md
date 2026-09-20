<p align="center">
  <img src="assets/banner-wide-open-security-group.svg" width="820" alt="The Wide-Open Security Group">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/EASY-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/SECURITY_GROUPS-1a0633?style=for-the-badge&labelColor=A479C4">
  <img src="https://img.shields.io/badge/NETWORKING-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

## Overview

> [!NOTE]
> **TL;DR** — An eager admin builds a web app on EC2 and opens SSH (22), HTTP (80) and MySQL (3306) to `0.0.0.0/0` in a single Security Group. You audit the rules, strip the dangerous ones, replace the DB rule with an **SG-to-SG reference**, and rebuild a multi-tier SG model from scratch. Framed by the 2017 MongoDB Apocalypse.

## Real-world incident — The MongoDB Apocalypse (2017)

Automated scripts swept cloud IP ranges for hosts with the default MongoDB port **27017** open (Shodan had already indexed thousands). Attackers connected to databases that had **no authentication**, exfiltrated (or claimed to), wiped every collection, and dropped a ransom note. By mid-January 27k+ databases were destroyed; by September 45k+, spreading to Elasticsearch, Hadoop, CouchDB, Cassandra and MySQL.

> [!IMPORTANT]
> The AWS-side root cause was **security groups** with the database port open to the internet. Add no authentication and no backups, and a single open inbound rule becomes an extinction event. The lesson: database and management ports never belong on `0.0.0.0/0`.

## Identification — audit the Security Groups

```bash
# find the instances: web app (public IP) + db server (private, no public IP)
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].{Name:Tags[?Key=='Name']|[0].Value,PublicIp:PublicIpAddress,SG:SecurityGroups[0].GroupName}" --output table

# inspect the web app SG inbound rules
WEB_SG_ID=$(aws ec2 describe-security-groups --filters 'Name=group-name,Values=thm-web-app-sg' --query 'SecurityGroups[0].GroupId' --output text)
aws ec2 describe-security-group-rules --filters "Name=group-id,Values=$WEB_SG_ID" \
  --query "SecurityGroupRules[].{Protocol:IpProtocol,FromPort:FromPort,Source:CidrIpv4,Description:Description}" --output table
```

**Findings:** `thm-web-app-sg` has **3 inbound rules**, all from `0.0.0.0/0`:
- **22 (SSH)** — a management port exposed to the whole internet. Bad.
- **80 (HTTP)** — acceptable for a public web app, but should move to **443 (HTTPS)**.
- **3306 (MySQL)** — the database port open to the world, on both the web app *and* the DB SG. This is the MongoDB Apocalypse in miniature.

> [!TIP]
> Core SG concepts to keep straight: **stateful** (return traffic auto-allowed, so inbound rules suffice), **allow-only** (implicit deny — you can't write a deny rule), and **instance-level** (attached to the ENI; up to **5** SGs by default, rules combined).

## Remediation

Strategy: SSH → use SSM instead; MySQL → the web app doesn't need it inbound; DB port → allow only the web tier via an **SG reference**, never a CIDR.

```bash
# 1. drop public SSH from the web SG (use SSM for admin)
aws ec2 revoke-security-group-ingress --group-id $WEB_SG_ID --protocol tcp --port 22 --cidr 0.0.0.0/0
# 2. drop public MySQL from the web SG
aws ec2 revoke-security-group-ingress --group-id $WEB_SG_ID --protocol tcp --port 3306 --cidr 0.0.0.0/0
# 3. on the DB SG: remove 0.0.0.0/0, add an SG-to-SG rule (only the web SG may reach 3306)
aws ec2 revoke-security-group-ingress    --group-id $DB_SG_ID --protocol tcp --port 3306 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $DB_SG_ID --protocol tcp --port 3306 --source-group $WEB_SG_ID
# 4. verify from the web instance (via SSM): the DB is reachable internally
nc -v 10.0.2.10 3306 -w 3 < /dev/null
```

The key mechanism is the **SG-to-SG reference**: instead of a source IP/CIDR you name *another security group*. The rule shows `ReferencedGroupInfo` rather than `CidrIpv4`. The database now accepts 3306 only from instances carrying `thm-web-app-sg` — dynamic, survives IP changes, and never exposed to the internet.

## Build it securely — multi-tier SGs

Five principles: **default deny** (new SGs have no inbound), **one SG per role** (web tier / db tier / management), **SG references** for internal traffic, **controlled management ports** (SSM over SSH), and **egress rules** for sensitive tiers.

```bash
# web tier: HTTP + HTTPS from the internet only
aws ec2 authorize-security-group-ingress --group-id $WEB_SG_NEW \
  "IpProtocol=tcp,FromPort=80,ToPort=80,IpRanges=[{CidrIp=0.0.0.0/0}]" \
  "IpProtocol=tcp,FromPort=443,ToPort=443,IpRanges=[{CidrIp=0.0.0.0/0}]"
# db tier: MySQL from the web SG only (SG2SG)
aws ec2 authorize-security-group-ingress --group-id $DB_SG_NEW --protocol tcp --port 3306 --source-group $WEB_SG_NEW
# db egress: replace allow-all with HTTPS to the VPC only (VPC endpoints)
aws ec2 revoke-security-group-egress    --group-id $DB_SG_NEW --ip-permissions "IpProtocol=-1,IpRanges=[{CidrIp=0.0.0.0/0}]"
aws ec2 authorize-security-group-egress --group-id $DB_SG_NEW "IpProtocol=tcp,FromPort=443,ToPort=443,IpRanges=[{CidrIp=10.0.0.0/16}]"
```

Extra patterns: **three-tier** (Web → App → DB, so the DB never talks to the web tier directly), **bastion/jump host** (dedicated tight SG if SSH is unavoidable — but prefer SSM), and **microservices** (one SG per service = an auditable communication map).

## Task answers

| # | Task | Question | Answer |
|---|------|----------|--------|
| 1 | Introduction | Let us prevent an apocalypse | *(no answer)* |
| 2 | Real-World Incident | Primary misconfigured network-layer service | `security groups` |
| 3 | Identification | Inbound rules on `thm-web-app-sg` | `3` |
| 3 | Identification | Expected port / safer port | `80/443` |
| 3 | Identification | Default SGs per ENI | `5` |
| 4 | Remediation | Flag on the web app page | `THM{NEVER_GET_OUT_OF_THE_****}` |
| 4 | Remediation | Flag from the database service | `THM{NAPALM_IN_THE_*******}` |
| 4 | Remediation | Flag from the verifier | `THM{LISTEN_TO_THE_*******}` |
| 5 | Build It Securely | Service preferred instead of SSH | `SSM` |
| 5 | Build It Securely | Flag from the checker | `THM{APOCALYPSE_*******}` |
| 6 | Conclusion | I feel more secure already | *(no answer)* |

## Lessons learned

- **Management and datastore ports never go on `0.0.0.0/0`** — SSH (22), RDP (3389), MySQL (3306), PostgreSQL (5432), MongoDB (27017), Redis (6379), Elasticsearch (9200). Scanners find them in minutes.
- **SG-to-SG references beat hardcoded IPs** for internal tiers: dynamic, scale-friendly, and self-documenting.
- **Stateful and allow-only** — you can't override a permissive rule with a deny; you remove the rule and design the flow.
- **Egress matters too.** Restricting outbound on sensitive tiers caps the blast radius if an instance is compromised.

---

<p align="center">
  <a href="README.md">
    <img src="assets/more_writeups.svg" width="360" alt="Back to AWS Security Path">
  </a>
</p>
