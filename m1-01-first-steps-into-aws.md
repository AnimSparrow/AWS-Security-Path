<p align="center">
  <img src="assets/banner-first-steps.svg" width="820" alt="First Steps Into AWS">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/EASY-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/FOUNDATIONS-1a0633?style=for-the-badge&labelColor=A479C4">
  <img src="https://img.shields.io/badge/CLI-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

## Overview

> [!NOTE]
> **TL;DR** — Getting the lab environment running, learning the AWS Console layout, and getting comfortable with CloudShell and the AWS CLI. The room ends on running your first read-only enumeration commands (`sts get-caller-identity`, `iam list-groups`, `s3 ls`).

This is the entry point of the whole path. Nothing offensive yet — the goal is to make the environment second nature so later rooms are about *security*, not about fighting the console.

## Notes / key concepts

**Lab lifecycle** — every AWS room provisions a throwaway account via the **Cloud Details** button:

| State | Meaning |
|---|---|
| `No allocation` | nothing provisioned yet |
| `Provisioning` | lab being allocated (generate/reset) |
| `Ready` | credentials available in the Credentials tab |
| `Terminating` | being deallocated + cleaned |

Controls: **Generate** (allocate), **Reset** (fresh deploy, progress lost), **Terminate** (deallocate, progress lost), **Extend** (+1h once, on top of the default ~2h).

> [!WARNING]
> Switching rooms and generating a new environment **deallocates the previous one** — all prior progress is lost. Only one lab lives at a time.

**Console anatomy** — top nav (global search, CloudShell, region, account menu), Services menu, Console Home. Labs default to **`us-east-1` (N. Virginia)** unless a room says otherwise. When something looks missing, *check the region first* — some services are regional, some global.

**Identity check** — the cloud equivalent of `whoami`:

```bash
aws sts get-caller-identity
```

Returns three keys — `Account`, `Arn`, `UserId`. This is your first move whenever you assume a role or spin up a session, to confirm *who you actually are* right now.

**CLI command shape** — everything follows the same pattern:

```text
aws <service> <operation> [--parameters] [--region <region>] [--output <format>]
```

`aws help`, `aws <service> help`, and `aws <service> <operation> help` are your map. Operations list alphabetically — handy for the task below.

## Task answers

> [!TIP]
> Values like account name and bucket suffix are **per-lab** — yours will differ. The answers below are for reference; re-read them from your own instance.

| # | Task | Question | Answer |
|---|------|----------|--------|
| 1 | The TryHackMe Environment | I'm ready | *(no answer)* |
| 2 | AWS Environment Overview | Name of the account | `LabRoom` |
| 3 | AWS Web Interface Overview | Default deploy region | `us-east-1` |
| 4 | CloudShell Overview | Keys from the `whoami` equivalent (alphabetical) | `account,arn,userid` |
| 5 | AWS CLI Setup | CLI configured | *(no answer)* |
| 6 | CLI Commands Structure | First available EC2 command | `accept-address-transfer` |
| 6 | CLI Commands Structure | S3 bucket prefix containing the account ID | `firststeps` |
| 7 | Conclusion | Environment terminated | *(no answer)* |

**How the tricky ones fall out:**
- **Q4** — the three redacted keys in `get-caller-identity` are `UserId`, `Account`, `Arn`; sorted alphabetically → `account, arn, userid`.
- **Q6 (EC2)** — `aws ec2 help` lists operations alphabetically, so `accept-address-transfer` sits at the top.
- **Q6 (S3)** — `aws s3 ls` shows a bucket like `firststeps-<accountid>`; the prefix is `firststeps`.

## Lessons learned

- **Region + identity are the first two things to check** when CLI output disagrees with the console. Most "it's broken" moments are actually "wrong region" or "wrong identity".
- **`aws sts get-caller-identity` is the cheapest sanity check in AWS** — no permissions needed, instantly tells you which principal you're acting as.
- **The `help` tree is the real documentation.** Drilling `aws <service> <operation> help` beats guessing parameters, and the alphabetical ordering is occasionally the answer itself.
- **Terminate when done.** Labs are billed/cleaned resources — leaving them running is the cloud-cost version of leaving the lights on.

---

<p align="center">
  <a href="README.md">
    <img src="assets/more_writeups.svg" width="360" alt="Back to AWS Security Path">
  </a>
</p>
