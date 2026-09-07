<p align="center">
  <img src="assets/banner-shared-responsibility.svg" width="820" alt="Shared Responsibility Model">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/INFO-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/FOUNDATIONS-1a0633?style=for-the-badge&labelColor=A479C4">
  <img src="https://img.shields.io/badge/THEORY-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

## Overview

> [!NOTE]
> **TL;DR** — Where the line of responsibility sits between AWS and you. AWS secures the cloud (facilities, hardware, hypervisor); you secure what runs *in* it (identity, data, configuration, exposure). Three real breaches show what happens when that line is misread.

The one-line mantra to carry through the whole path:

> **AWS is responsible for the security *of* the cloud. I am responsible for the security *in* the cloud.**

## Notes / key concepts

**The sliding scale.** Responsibility doesn't vanish — it shifts. The more AWS manages a service, the less the customer has to secure. But some things never move to AWS's side:

> [!IMPORTANT]
> No matter the service, **IAM, data, and service configuration are always the customer's job** — and that's exactly where most incidents happen.

**Where the line lands per service:**

| Service | AWS handles | You handle |
|---|---|---|
| **EC2** | facilities, hardware, hypervisor | OS patching, hardening, Security Groups, IAM roles, app security |
| **RDS** | DB engine + infra patching (mostly) | DB access, encryption, parameter groups, public exposure |
| **Lambda** | runtime + infrastructure | code, permissions, secrets, logging |
| **S3** | underlying infrastructure | bucket policies, access controls, encryption, public access, lifecycle |

The mental model: **if it's identity, data, configuration, or exposure — it's on your side.**

## The three breaches

**Capital One — SSRF → metadata → data.** An SSRF flaw in a public-facing component (misconfigured WAF/app layer) was used to hit the instance metadata service, which handed back **temporary role credentials**. Those creds then read S3 the role could reach. AWS ran the platform correctly; *customer-side IAM permissions and config* built the path from "internet-facing app" to "data access."
→ **Fix: least privilege**, harden credential/metadata access, detective controls, guardrails (SCPs / permission boundaries / config rules).

**Uber (2016) — exposed credentials.** Attackers got into an engineer's GitHub (password reuse from a prior leak), found **AWS access keys hardcoded in the repo**, and used them to pull data from Uber's datastore. Once valid creds are out, **identity becomes the perimeter**.
→ **Fix: never store secrets in code** (use Secrets Manager / Parameter Store), rotate keys, prefer short-lived STS creds + roles, repo secret scanning, MFA on sensitive actions.

**"It was just a bucket" — leaky S3.** A team treats object storage as a file dump, relaxes access "just for testing," and never re-tightens it. The bucket policy / ACL drifts to public and the whole dataset is one endpoint away from the world (Pegasus Airlines, others). The real failure isn't one checkbox — it's the **missing safety net**: no Block Public Access, no preventive guardrails, no "bucket went public" alert.
→ **Fix: S3 Block Public Access** (account + bucket), policy-as-guardrails, config rules for public buckets, alert on policy changes.

## Task answers

| # | Task | Question | Answer |
|---|------|----------|--------|
| 1 | Introduction | Show me the way | *(no answer)* |
| 2 | Model Overview | One area you secure for a Lambda | `code` |
| 3 | Capital One | IAM best practice to always apply | `least privilege` |
| 4 | Uber | Use this instead of secrets in code | `secrets manager` |
| 5 | "It was just a bucket" | Common "leaky bucket" misconfiguration | `s3 public access` |
| 6 | Recap mini-game | Flag | `THM{SHARED_********************}` |
| 7 | Conclusion | With great power... | *(no answer)* |

> [!TIP]
> The recap is a "catch your responsibilities" mini-game — items rain down and you basket only the **customer** responsibilities (IAM policies, VPC route tables, app code, S3 bucket policy, CloudTrail, MFA, Security Groups, secrets, **OS patching on EC2**). Catching an AWS-owned item, or *missing* a customer one, both count as mistakes. 8/10 clears it.

## Lessons learned

- **Most high-impact cloud incidents are customer-side config, not AWS failures** — over-permissive policies, public exposure, leaked secrets, no logging/alerting.
- **Least privilege is the single highest-leverage control.** It caps the blast radius of every other mistake (see: Capital One's stolen role could only reach what it was allowed to).
- **Identity is the perimeter.** Once valid credentials leak, network controls barely matter — which is why secret hygiene and MFA earn their keep.
- **Prevention needs a safety net, not a checkbox.** Block Public Access + guardrails + detection is what stops "temporary" misconfigs from becoming permanent leaks.

---

<p align="center">
  <a href="README.md">
    <img src="assets/more_writeups.svg" width="360" alt="Back to AWS Security Path">
  </a>
</p>
