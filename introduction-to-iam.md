<p align="center">
  <img src="assets/banner-introduction-to-iam.svg" width="820" alt="Introduction to IAM">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/EASY-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/IAM-1a0633?style=for-the-badge&labelColor=A479C4">
  <img src="https://img.shields.io/badge/POLICIES-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

## Overview

> [!NOTE]
> **TL;DR** — The IAM foundations: identities (root / user / group / role), the five policy types, how a request is evaluated (PARC + explicit-deny-wins), policy JSON structure, credential types, and the Principle of Least Privilege — with two hands-on labs (a leaky logs bucket and a PoLP puzzle).

Security in AWS starts and ends with IAM. This room is the backbone the rest of the path leans on.

## Notes / key concepts

**Identities:**

| Identity | What it is |
|---|---|
| **Root user** | created with the account, full access — lock it down with MFA + strong password and store it away |
| **IAM user** | a person/app with *permanent* credentials (password, access keys) |
| **IAM group** | a bucket of users, purely to manage permissions at scale |
| **IAM role** | *no permanent* credentials — temporary, assumable access (best for services like EC2/Lambda) |

**Policy types (5):** identity-based (most common), resource-based (e.g. bucket policy), permissions boundary (max grantable permissions), service control policy (org-wide guardrails via AWS Organizations), session policy (limits a programmatic assume-role session).

**Evaluation logic** — AWS first assembles a request context, remembered as **PARC** (*Principal, Action, Resource, Condition*), then:

```text
1. Explicit Deny        → stop, DENY  (deny always wins)
2. Permission Boundary  → constraint check
3. IAM Policies         → evaluate attached policies
4. Allow                → explicit allow + not blocked above
5. Default Deny         → nothing allowed it → DENY
```

**Policy JSON skeleton** — `Version` + one or more `Statement` elements, each with `Sid` (optional), `Effect` (`Allow`/`Deny`, case-sensitive), `Principal`, `Action`, `Resource`, `Condition`.

> [!TIP]
> **The Deny/Allow trap.** A statement that denies `ec2:TerminateInstances` for *everyone except* `try-terminate-me` (`ArnNotLike`) does **not** implicitly allow that user — without a second explicit `Allow` statement, they still hit **default deny**. Explicit deny narrows; it never grants.

**Credential types:** console passwords, access keys (long-term programmatic), temporary credentials (STS, short-lived), role credentials (for compute, never stored on disk).

## Lab 1 — leaky logs bucket

Created a new user `trylogme`, dropped it into the `LogsReaders` group, logged in via an incognito window, and explored S3:

```text
S3 → try-log-me-<accountid> → logs/you-found-me.txt → download → flag
```

The lesson baked in: **a read-only group + a discoverable bucket = data exposure.** The user was only supposed to read logs, but "read logs" was enough to walk out with the stash.

| | |
|---|---|
| Bucket prefix | `try-log-me` |
| Flag | `THM{MY_***********}` |

## Lab 2 — Principle of Least Privilege (3 levels)

PoLP = minimum permissions, minimum resources, minimum time. The puzzle drills exactly the mistakes people make:

- **L1 — action too broad.** `s3:*` → tighten to `s3:GetObject` (get is all "download objects" needs; `*` silently includes deletes, ACL changes, policy changes).
- **L2 — action *and* resource.** `s3:GetObject` scoped to `arn:aws:s3:::try-polp-me/public/*` — minimum action + minimum resource.
- **L3 — three statements, one field each:**
  - `s3:ListBucket` must target the **bucket ARN** (`try-polp-me`, *no* `/*`) — listing is a bucket-level action.
  - `s3:GetObject` must target only the needed object prefix (`.../logs/*`).
  - EC2 restricted to `ec2:DescribeInstances` (read-only).

> [!IMPORTANT]
> `ListBucket` acts on the **bucket**; `GetObject` acts on the **objects**. Getting the ARN granularity wrong here is the single most common IAM policy bug.

| | |
|---|---|
| Flag | `THM{POLP_*******}` |

## Task answers

| # | Task | Question | Answer |
|---|------|----------|--------|
| 1 | Introduction | Ready to start | *(no answer)* |
| 2 | IAM Identities | Groups your user is in (alphabetical) | `AWS103, LabReadOnly` |
| 3 | Understanding Policies | Most common policy type | `identity-based` |
| 3 | Understanding Policies | Context gathered before evaluation (mnemonic) | `principal,action,resource,condition` |
| 3 | Understanding Policies | A best practice for a regular user | `MFA` |
| 4 | Policy Structure | Action that retrieves objects from `thm-bucket` | `s3:GetObject` |
| 4 | Policy Structure | Result when `try-terminate-me` requests termination | `allow` |
| 5 | Credentials | Prefix of the bucket you found | `try-log-me` |
| 5 | Credentials | Flag inside the bucket | `THM{MY_***********}` |
| 6 | Least Privilege | Flag after all three exercises | `THM{POLP_*******}` |
| 7 | Conclusion | It's hammer time | *(no answer)* |

## Lessons learned

- **Deny always wins, and default is deny.** A missing explicit `Allow` is as blocking as an explicit `Deny` — a huge share of "why can't I do X" comes down to this.
- **`ListBucket` ≠ `GetObject` in ARN scope.** Bucket-level vs object-level actions target different ARNs; mixing them up either breaks access or over-grants it.
- **Wildcards hide blast radius.** `s3:*` reads as "download" to a hurried author but includes delete and policy changes. Name the exact action.
- **Least privilege isn't a phase, it's a default.** Start narrow (one action/resource pair), add only what's proven necessary, and gate sensitive actions behind MFA conditions.

---

<p align="center">
  <a href="../README.md">
    <img src="../assets/more_writeups.svg" width="360" alt="Back to AWS Security Path">
  </a>
</p>
