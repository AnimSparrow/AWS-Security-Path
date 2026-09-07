<p align="center">
  <img src="banner-cloud-storage.svg" width="820" alt="Introduction to Cloud Storage">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/EASY-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/S3_EBS_EFS-1a0633?style=for-the-badge&labelColor=A479C4">
  <img src="https://img.shields.io/badge/STORAGE-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

## Overview

> [!NOTE]
> **TL;DR** — The three AWS storage models — **S3** (object), **EBS** (block), **EFS** (file) — what each is for and how to secure it. Hands-on: enumerate an S3 bucket via CLI, read an encrypted EBS volume over SSM, and mount a shared EFS. Four flags total.

## Notes / key concepts

**The three models:**

| Feature | S3 (Object) | EBS (Block) | EFS (File) |
|---|---|---|---|
| Access | HTTP API (CLI/SDK/console) | attached to **one** EC2 | NFS mount (**many** EC2) |
| Structure | flat (key prefixes fake folders) | filesystem (ext4/xfs) | filesystem (NFS v4) |
| AZ scope | regional | **single AZ** | regional |
| Persistence | independent of compute | survives stop/start | independent of compute |
| Security | bucket policies, ACLs, Block Public Access | encryption, snapshot perms | encryption, SG, mount targets |
| Use cases | backups, logs, static sites | OS drives, DBs, app data | shared files, CMS, ML data |

> [!IMPORTANT]
> **Data at rest is protected by encryption** across all three. For EFS there's a catch: **encryption at rest can only be enabled at creation** — you can't turn it on later.

**S3 specifics:** bucket names are **globally unique**, objects live in the bucket's region, up to 5 TB each. Access controls: **bucket policies** (preferred), **Block Public Access** (overrides any policy/ACL that would make it public), and legacy **ACLs** (avoid). The "leaky bucket" is just a bucket where Block Public Access is off and a policy/ACL drifts public.

**EBS specifics:** **AZ-locked** (volume attaches only to instances in its own AZ). Snapshots are point-in-time, incremental, stored in S3, **regional** (restore into any AZ in the region) — and **shareable, even publicly (!)**, which is a real exposure risk. On Nitro instances EBS shows up as `nvme*` devices, not `xvd*`.

**EFS specifics:** fully-managed NFS, auto-scaling. Access needs a **mount target** = an ENI in a subnet **with a Security Group** — that SG is your primary "who can mount this" control. Storage classes: Standard / IA / Archive.

## Lab walkthroughs

**S3 — enumerate & verify (the "leaky bucket" drill):**

```bash
aws s3 ls                                        # find the bucket
aws s3 ls s3://storage-lab-<id>                   # PRE data/  PRE logs/
aws s3 ls s3://storage-lab-<id>/logs/             # access-log.txt
aws s3 cp s3://storage-lab-<id>/logs/access-log.txt -   # print → flag
aws s3api get-public-access-block --bucket storage-lab-<id>
# all four BlockPublic* = true  → bucket is NOT public. Good.
```

**EBS — find the encrypted volume, read it over SSM:**

```bash
aws ec2 describe-volumes --query "Volumes[*].{ID:VolumeId,Size:Size,Encrypted:Encrypted,AZ:AvailabilityZone}" --output table
aws ec2 describe-snapshots --owner-ids self --query "Snapshots[*].{ID:SnapshotId,Encrypted:Encrypted}" --output table

aws ssm start-session --target <instance-id>
lsblk                       # nvme1n1 (or xvdf) → /mnt/data
cat /mnt/data/flag.txt
```

**EFS — locate the file system, read the shared mount:**

```bash
aws efs describe-file-systems --query "FileSystems[*].{ID:FileSystemId,Name:Name,Encrypted:Encrypted}" --output table
aws efs describe-mount-targets --file-system-id <fs-id> --query "MountTargets[*].{ID:MountTargetId,SubnetId:SubnetId,IP:IpAddress}" --output table

# inside the instance:
df -h                       # 127.0.0.1:/  → /mnt/shared  (NFS)
cat /mnt/shared/flag.txt
```

## Task answers

| # | Task | Question | Answer |
|---|------|----------|--------|
| 1 | Introduction | Ready | *(no answer)* |
| 2 | Storage Models | How is data at rest protected | `encryption` |
| 2 | Storage Models | Mini-game flag | `THM{ONE_BITE_AT_A_****}` |
| 3 | Simple Storage Service | Use instead of ACLs | `bucket policy` |
| 3 | Simple Storage Service | Flag in `access-log.txt` | `THM{NO_PUBLIC_**}` |
| 4 | Elastic Block Store | A limitation of EBS | `az-locked` |
| 4 | Elastic Block Store | Flag on the mounted volume | `THM{EBS_SPEAKS_******}` |
| 5 | Elastic File System | Control on a mount point to restrict access | `Security Groups` |
| 5 | Elastic File System | Flag on the EFS mount | `THM{SHARING_IS_******}` |
| 6 | Conclusion | Buckets remain private | *(no answer)* |

## Lessons learned

- **Match the model to the access pattern:** object (S3) for write-once/read-many, block (EBS) for one low-latency disk, file (EFS) for many instances sharing files.
- **"Leaky bucket" is the #1 storage misconfig.** Block Public Access at account + bucket level is the safety net; bucket policies over ACLs.
- **Snapshots can be made public.** An encrypted volume means nothing if its snapshot is shared to the world — review snapshot permissions regularly.
- **EFS encryption is a launch-time decision.** Miss it at creation and the only fix is recreate-and-migrate. The SG on the mount target is what gates access.

---

<p align="center">
  <a href="README.md">
    <img src="more_writeups.svg" width="360" alt="Back to AWS Security Path">
  </a>
</p>
