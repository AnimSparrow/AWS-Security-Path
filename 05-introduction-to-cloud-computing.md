<p align="center">
  <img src="assets/banner-cloud-computing.svg" width="820" alt="Introduction to Cloud Computing">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/EASY-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/EC2-1a0633?style=for-the-badge&labelColor=A479C4">
  <img src="https://img.shields.io/badge/COMPUTE-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

## Overview

> [!NOTE]
> **TL;DR** — AWS compute options (EC2 / containers / Lambda) and where each sits on the responsibility spectrum, then a deep dive on EC2 core concepts (AMIs, instance types, User Data, key pairs), connecting via SSM, the instance metadata service, and the "pets vs cattle" mindset. Two flags come from misused User Data and the web server.

## Notes / key concepts

**Compute options — the responsibility slider:**

| Service | You manage | AWS manages |
|---|---|---|
| **EC2** | OS, patches, apps, data, security config | hardware, hypervisor, physical |
| **Containers (ECS/EKS/Fargate)** | app code, deps, task config | host OS (Fargate), orchestrator |
| **Lambda** | code, IAM role, triggers | provisioning, scaling, patching |

Rule of thumb: **EC2** for full OS control / long-running processes; **containers** for consistent, fast-starting deployments; **Lambda** for event-driven, short-lived tasks. More AWS management = less to secure, but *data and access are always yours*.

**EC2 core concepts:**
- **AMI** — a template (OS + software + config). Bake a hardened, patched image once; launch consistent instances from it.
- **Instance types** — `t3.micro` = `family(t=burstable).generation(3).size(micro)`. Families: `t`=burstable, `m`=balanced, `c`=compute-optimized, `r`=memory-optimized. Watch the bill.
- **User Data** — a script that runs **once**, on first launch. Runs as **root**, stored in instance metadata (base64).
- **Key Pairs** — SSH access (public key on instance, private key with you). SSM Session Manager is the safer alternative.

> [!WARNING]
> **User Data runs as root and lives in instance metadata in plaintext (base64).** Never put secrets there — anyone who can read the instance attribute or reach IMDS can decode them. Use Secrets Manager / SSM Parameter Store instead. The room's flag is literally sitting in a User Data script to prove the point.

**Connecting — SSM > SSH:**

| | SSH | SSM Session Manager |
|---|---|---|
| Needs open port 22 | yes | **no** |
| Needs public IP | yes | no |
| Access control | key file | **IAM policy** |
| Logging | manual | built-in |

SSM Agent talks *outbound* to the SSM service, so no inbound exposure. You land as `ssm-user`.

**Instance Metadata Service (IMDS)** — reachable from inside the instance at `169.254.169.254`. Holds instance type, IAM role creds, User Data, etc. (This is the exact endpoint abused in the Capital One SSRF breach from the Shared Responsibility room.)

## Lab — enumeration highlights

**Read the User Data (where the secret shouldn't be):**

```bash
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=room15-web" "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].InstanceId" --output text)

aws ec2 describe-instance-attribute --instance-id $INSTANCE_ID \
  --attribute userData --query "UserData.Value" --output text | base64 --decode
# → #!/bin/bash ... echo "THM{...}" > /var/www/html/index.html
```

**Query IMDS from inside the instance (IMDSv2, token-based):**

```bash
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-type
```

## "Pets vs Cattle"

Moving on-prem → cloud is a mindset shift:

- **Pets** — hand-fed servers with names, patched individually; rebuild takes hours.
- **Cattle** — identical, disposable instances from pre-baked images; replace instead of repair.

**Immutable infrastructure** = you build a *new AMI* for patches/config and roll it out, rather than mutating live instances. Payoffs: no config drift, reproducible (IaC), and **reduced attack persistence** — replacing a compromised instance wipes the foothold.

## Task answers

| # | Task | Question | Answer |
|---|------|----------|--------|
| 1 | Introduction | Let us compute | *(no answer)* |
| 2 | Compute Services | Most control + most responsibility | `ec2` |
| 2 | Compute Services | Run code without provisioning infra | `lambda` |
| 2 | Compute Services | Lab instance name | `room15-web` |
| 3 | EC2 Core Concepts | Flag in the instance User Data | `THM{NO_PASSWORDS_****}` |
| 3 | EC2 Core Concepts | User that User Data scripts run as | `root` |
| 4 | Connecting to Instances | User shown by `whoami` (via SSM) | `ssm-user` |
| 4 | Connecting to Instances | IMDS IP address | `169.254.169.254` |
| 5 | Pets vs Cattle | Building new AMIs instead of patching | `immutable` |
| 6 | Conclusion | Compute is the engine | *(no answer)* |

## Lessons learned

- **User Data is not a vault.** Root execution + plaintext-in-metadata means any secret there is effectively public to anyone on the box or with `describe-instance-attribute`.
- **SSM beats SSH for security.** No open port 22, no public IP, IAM-gated, fully logged. If SSH is unavoidable, scope the SG to specific IPs and rotate keys.
- **IMDS is a credential source — treat it as sensitive.** SSRF that reaches `169.254.169.254` can lift role creds; IMDSv2's token requirement is the mitigation.
- **Cattle, not pets.** Disposable, immutable instances shrink drift and kill attacker persistence — replacement *is* remediation.

---

<p align="center">
  <a href="README.md">
    <img src="assets/more_writeups.svg" width="360" alt="Back to AWS Security Path">
  </a>
</p>
