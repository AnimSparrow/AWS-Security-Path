<p align="center">
  <img src="banner-path.svg" width="820" alt="AWS Security — Writeups">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/AWS-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/CLOUD_SECURITY-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

Notes and solutions from the TryHackMe **AWS** security path — one `.md` per room. Each writeup carries a short TL;DR, the key concepts worth keeping, a per-lab walkthrough where relevant, and the task answers.

> [!NOTE]
> Flags are **masked** on purpose (portfolio etiquette + platform rules). The reasoning is the point, not the copy-paste.

## Module 1 · Welcome to AWS

> Cloud security starts with understanding the platform. A solid foundation across AWS core services, identity, networking, storage, and native security tooling.

| # | Room | Status |
|:--:|---|:--:|
| 01 | [First Steps Into AWS](01-first-steps-into-aws.md) | ✅ |
| 02 | [Shared Responsibility Model](02-shared-responsibility-model.md) | ✅ |
| 03 | [Introduction to IAM](03-introduction-to-iam.md) | ✅ |
| 04 | [Introduction to Cloud Networking](04-introduction-to-cloud-networking.md) | ✅ |
| 05 | [Introduction to Cloud Computing](05-introduction-to-cloud-computing.md) | ✅ |
| 06 | [Introduction to Cloud Storage](06-introduction-to-cloud-storage.md) | ✅ |
| 07 | [Introduction to AWS Security Tools](07-introduction-to-aws-security-tools.md) | ✅ |

> [!NOTE]
> **Module 1 complete — 7/7 rooms.** All foundational rooms of *Welcome to AWS* are done. Topic Rewind Recap is the module's closing knowledge check.

## Module 2 · Identity and Access Management

> Identity is the new perimeter in the cloud. How misconfigured users, forgotten credentials, and overpowered roles become an attacker's easiest entry point.

| # | Room | Status |
|:--:|---|:--:|
| 01 | [The Over-Privileged User](m2-01-the-over-privileged-user.md) | ✅ |
| 02 | [The Forgotten Access Key](m2-02-the-forgotten-access-key.md) | ✅ |
| 03 | [The Overpowered Role](m2-03-the-overpowered-role.md) | ✅ |
| 04 | [The Silence of the IAMs](m2-04-the-silence-of-the-iams.md) | ✅ |

> [!NOTE]
> **Module 2 complete — 4/4 rooms.** Arc: over-privileged users → forgotten keys → overpowered roles → detection. Each room built on a real breach (Code Spaces, Uber, Capital One, Ubiquiti).

## Running theme so far

Every room reinforces the same core idea: in the cloud, **identity, data, and configuration are always yours to secure**. Least privilege caps the blast radius; explicit deny wins every evaluation; and a leaked credential turns identity into the perimeter.

---

<p align="center"><sub>Styled in the synthwave-terminal identity · writeups by AnimSparrow</sub></p>
