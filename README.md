<p align="center">
  <img src="assets/banner-path.svg" width="820" alt="AWS Security — Writeups">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/AWS-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/CLOUD_SECURITY-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

Notes and solutions from the TryHackMe **AWS** security path — one `.md` per room, grouped by module. Each writeup carries a short TL;DR, the key concepts worth keeping, a per-lab walkthrough where relevant, and the task answers.

> [!NOTE]
> Flags are **masked** on purpose (portfolio etiquette + platform rules). The reasoning is the point, not the copy-paste.

## Progress

| Module | Room | Status |
|---|---|:--:|
| **01 · Welcome to AWS** | [First Steps Into AWS](01-welcome-to-aws/first-steps-into-aws.md) | ✅ |
| **01 · Welcome to AWS** | [Shared Responsibility Model](01-welcome-to-aws/shared-responsibility-model.md) | ✅ |
| **02 · Identity and Access Management** | [Introduction to IAM](02-identity-and-access-management/introduction-to-iam.md) | ✅ |
| 03 · Network and Perimeter Defense | Introduction to Cloud Networking | ⬜ |
| 04 · Securing Compute | Introduction to Cloud Computing | ⬜ |
| 05 · Storage and Data Security | Introduction to Cloud Storage | ⬜ |
| — · AWS Security Tools | Introduction to AWS Security Tools | ⬜ |

## Structure

```text
AWS-Security-Path/
├── README.md
├── assets/                          # shared graphics (path banner, button)
├── 01-welcome-to-aws/
│   ├── assets/                      # per-room banners
│   ├── first-steps-into-aws.md
│   └── shared-responsibility-model.md
└── 02-identity-and-access-management/
    ├── assets/
    └── introduction-to-iam.md
```

## Running theme so far

Every room reinforces the same core idea: in the cloud, **identity, data, and configuration are always yours to secure**. Least privilege caps the blast radius; explicit deny wins every evaluation; and a leaked credential turns identity into the perimeter.

---

<p align="center"><sub>Styled in the synthwave-terminal identity · writeups by AnimSparrow</sub></p>
