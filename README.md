<p align="center">
  <img src="assets/banner-path.svg" width="820" alt="AWS Security — Writeups">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/AWS-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/CLOUD_SECURITY-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

Notes and solutions from the TryHackMe **Defending AWS** learning path — one `.md` per room. Each writeup carries a short TL;DR, the key concepts worth keeping, a per-lab walkthrough where relevant, and the task answers.

> [!NOTE]
> Flags are **masked** on purpose (portfolio etiquette + platform rules). The reasoning is the point, not the copy-paste.

> [!TIP]
> Read the honest **[S.P.A.R.R.O.W. review of this path](AWS_PATH_REVIEW.md)** (scored **4.54 / 10**) for whether it's worth your time.

## Module 1 · Welcome to AWS

> Cloud security starts with understanding the platform. A solid foundation across AWS core services, identity, networking, storage, and native security tooling.

| #  | Room | Status |
| --- | ---- | ------ |
| 01 | [First Steps Into AWS](m1-01-first-steps-into-aws.md) | ✅ |
| 02 | [Shared Responsibility Model](m1-02-shared-responsibility-model.md) | ✅ |
| 03 | [Introduction to IAM](m1-03-introduction-to-iam.md) | ✅ |
| 04 | [Introduction to Cloud Networking](m1-04-introduction-to-cloud-networking.md) | ✅ |
| 05 | [Introduction to Cloud Computing](m1-05-introduction-to-cloud-computing.md) | ✅ |
| 06 | [Introduction to Cloud Storage](m1-06-introduction-to-cloud-storage.md) | ✅ |
| 07 | [Introduction to AWS Security Tools](m1-07-introduction-to-aws-security-tools.md) | ✅ |

> [!NOTE]
> **Module 1 complete — 7/7 rooms.** All foundational rooms of *Welcome to AWS* are done. Topic Rewind Recap is the module's closing knowledge check.

## Module 2 · Identity and Access Management

> Identity is the new perimeter in the cloud. How misconfigured users, forgotten credentials, and overpowered roles become an attacker's easiest entry point.

| #  | Room | Status |
| --- | ---- | ------ |
| 01 | [The Over-Privileged User](m2-01-the-over-privileged-user.md) | ✅ |
| 02 | [The Forgotten Access Key](m2-02-the-forgotten-access-key.md) | ✅ |
| 03 | [The Overpowered Role](m2-03-the-overpowered-role.md) | ✅ |
| 04 | [The Silence of the IAMs](m2-04-the-silence-of-the-iams.md) | ✅ |

> [!NOTE]
> **Module 2 complete — 4/4 rooms.** Arc: over-privileged users → forgotten keys → overpowered roles → detection. Each room built on a real breach (Code Spaces, Uber, Capital One, Ubiquiti).

## Module 3 · Network and Perimeter Defense

> A single misconfigured rule can expose an entire AWS environment. Identify and close the network-level gaps that attackers use to gain a foothold.

| #  | Room | Status |
| --- | ---- | ------ |
| 01 | [The Wide-Open Security Group](m3-01-the-wide-open-security-group.md) | ✅ |
| 02 | [The Not So Private Subnet](m3-02-the-not-so-private-subnet.md) | ✅ |
| 03 | [The Forgotten NACL](m3-03-the-forgotten-nacl.md) | ✅ |
| 04 | [The Invisible Network](m3-04-the-invisible-network.md) | ✅ |

> [!NOTE]
> **Module 3 complete — 4/4 rooms.** Layered defence: Security Groups (host) → route tables (public/private routing) → NACLs (subnet perimeter) → Flow Logs (visibility). Each room built on a real breach (MongoDB Apocalypse, Tesla, SCARLETEEL, Marriott).

## Module 4 · Securing Compute

> Compute resources are a prime target in the cloud. From exposed ports to container misconfigurations, recognize and remediate the threats targeting AWS workloads.

| #  | Room | Status |
| --- | ---- | ------ |
| 01 | [The Exposed Port](m4-01-the-exposed-port.md) | ✅ |
| 02 | [The Unpatched Instance](m4-02-the-unpatched-instance.md) | ✅ |
| 03 | [The Leaky Metadata](m4-03-the-leaky-metadata.md) | ✅ |
| 04 | [The Oversharing Container](m4-04-the-oversharing-container.md) | ✅ |

> [!NOTE]
> **Module 4 complete — 4/4 rooms.** Arc: exposed port → patch debt → IMDS credential theft → container identity. Each room built on a real breach (TeamTNT, WannaCry, Shopify SSRF, Hildegard).

## Module 5 · Storage and Data Security

> Data breaches often begin with a misconfigured bucket or an exposed snapshot. Secure AWS storage and prevent sensitive data from reaching the wrong hands.

| #  | Room | Status |
| --- | ---- | ------ |
| 01 | [The Leaky Bucket](m5-01-the-leaky-bucket.md) | ✅ |
| 02 | [The Plain Bucket](m5-02-the-plain-bucket.md) | ✅ |
| 03 | [The Blind Bucket](m5-03-the-blind-bucket.md) | ✅ |
| 04 | [The Shared Snapshot](m5-04-the-shared-snapshot.md) | ✅ |

> [!NOTE]
> **Module 5 complete — 4/4 rooms.** S3 confidentiality (Leaky) → encryption boundary (Plain) → object-level visibility (Blind) → EBS snapshots (Shared). Each room built on a real breach (Verizon/Pegasus, LastPass, Chegg, Bishop Fox DEF CON).

## Running theme

Every room reinforces the same core idea: in the cloud, **identity, data, and configuration are always yours to secure**. Least privilege caps the blast radius; explicit deny wins every evaluation; and a leaked credential turns identity into the perimeter.

---

<p align="center"><sub>Styled in the synthwave-terminal identity · writeups by AnimSparrow</sub></p>
