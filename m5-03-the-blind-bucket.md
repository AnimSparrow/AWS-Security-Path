<p align="center">
  <img src="assets/banner-blind-bucket.svg" width="820" alt="The Blind Bucket">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/EASY-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/S3_DATA_EVENTS-1a0633?style=for-the-badge&labelColor=A479C4">
  <img src="https://img.shields.io/badge/VISIBILITY-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

## Overview

> [!NOTE]
> **TL;DR** — S3 access logging isn't on by default, and CloudTrail doesn't log S3 **data** events by default, so object access is invisible. You confirm the blind spot, enable CloudTrail S3 data events and S3 server access logging, then query the data events in CloudWatch Logs Insights to see exactly who read what. Framed by the 2018 Chegg breach.

## Real-world incident — Chegg (2018)

A former contractor kept access to **shared root credentials** after their engagement, enumerated S3, and downloaded ~40M user records (names, emails, hashed passwords, scholarship data). Because **S3 data events were not enabled**, the bulk download left no CloudTrail trace. The breach went undetected for **five months**, until a threat-intel vendor found 25M plaintext passwords on a forum. The FTC cited Chegg for failing to monitor for exfiltration.

> [!IMPORTANT]
> Two failures compounded: **root credentials** carry unrestricted permissions and can't be scoped by IAM, and **CloudTrail management events don't capture GetObject/PutObject/DeleteObject** — object access needs data events explicitly enabled. Together they created a total blind spot.

## Server access logs vs CloudTrail data events

| | S3 Server Access Logs | CloudTrail S3 Data Events |
|---|---|---|
| Records | every HTTP request | API data operations (GetObject, PutObject...) |
| Latency | minutes to hours | ~15 minutes |
| Format | space-separated | JSON |
| IAM identity | **no** (just IP/user-agent) | **yes (full caller ARN)** |
| Integration | S3 prefix search | CloudWatch, Insights, Athena, GuardDuty |

Enable **both**: server logs for per-request detail, data events for IAM context and monitoring integration.

## Identification — confirm the blind spot

```bash
aws s3api get-bucket-logging --bucket $BUCKET_NAME              # empty = no server access logging
aws cloudtrail get-event-selectors --trail-name $TRAIL_NAME
#   ReadWriteType: All, IncludeManagementEvents: true
#   DataResources: []   ← ZERO S3 data events
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventSource,AttributeValue=s3.amazonaws.com
#   only management events (GetBucketOwnershipControls...), no GetObject
```

`ReadWriteType: All` means the trail tracks read and write **management** events, but **`DataResources: []`** means object access is invisible — retrieving the flag (or any object) leaves no trace. Exactly the Chegg blind spot.

## Remediation — enable both logging layers

```bash
# 1. CloudTrail S3 data events for this bucket
aws cloudtrail put-event-selectors --trail-name $TRAIL_NAME --event-selectors "[{
  \"ReadWriteType\":\"All\",\"IncludeManagementEvents\":true,
  \"DataResources\":[{\"Type\":\"AWS::S3::Object\",\"Values\":[\"arn:aws:s3:::${BUCKET_NAME}/\"]}]}]"

# 2. access is now recorded — query it in CloudWatch Logs Insights (CWLI)
#    fields @timestamp, eventName, userIdentity.arn, requestParameters.key
#    | filter eventSource="s3.amazonaws.com" and eventName in ["GetObject","PutObject","DeleteObject"]
#    → GetObject | user/... | flag.txt   (full IAM ARN of the caller — what Chegg lacked)

# 3. S3 server access logging as a complementary record
aws s3api put-bucket-logging --bucket $BUCKET_NAME \
  --bucket-logging-status "{\"LoggingEnabled\":{\"TargetBucket\":\"$LOGGING_BUCKET\",\"TargetPrefix\":\"...\"}}"
```

> [!TIP]
> The query language short name is **CWLI** (CloudWatch Logs Insights). Data events have a **~15 minute delay** (2–3 min in the lab) before records appear — don't panic if the first query is empty.

## Build it securely — a fully monitored deployment

```bash
# secure both data + logging buckets: Block Public Access, BucketOwnerEnforced, encryption
# logging bucket policy → logging.s3.amazonaws.com with aws:SourceAccount guard
# then BOTH log sources on the data bucket:
aws s3api put-bucket-logging --bucket $DATA_BUCKET --bucket-logging-status "{...TargetBucket:$LOG_BUCKET...}"
aws cloudtrail put-event-selectors --trail-name $TRAIL_NAME --event-selectors "[{...DataResources:[{Type:AWS::S3::Object,Values:[arn:aws:s3:::$DATA_BUCKET/]}]}]"
```

Detection queries worth keeping (CWLI): **top principals by read volume**, **>100 GetObject in 5 min per principal** (bulk download like Chegg), **access to sensitive path prefixes**, and **first-time access** to a bucket (`dedup`).

> [!WARNING]
> Data events generate significant volume on high-traffic buckets. Use **specific bucket ARNs, not wildcards**, to control cost.

## Task answers

| # | Task | Question | Answer |
|---|------|----------|--------|
| 1 | Introduction | I am blind, not deaf | *(no answer)* |
| 2 | Real-World Incident | Credentials that can't be limited by IAM | `root` |
| 3 | Identification | Read/write activity type in the event selector | `all` |
| 3 | Identification | Flag stored in the S3 text file | `THM{LET_THE_HUNT_*****}` |
| 4 | Remediation | Short name for the query language | `CWLI` |
| 4 | Remediation | Flag | `THM{NOW_EVERYTHING_IS_*******}` |
| 5 | Build It Securely | Flag | `THM{YOU_ARE_NOW_********}` |
| 6 | Conclusion | Now your blind eyes see | *(no answer)* |

## Lessons learned

- **CloudTrail doesn't log S3 data events by default** — treat their absence as a misconfiguration, and enable them for every sensitive bucket.
- **Server access logs and data events complement each other** — per-request HTTP detail vs IAM identity + tool integration.
- **Logs are useless unless queried** — build detection queries for bulk reads, anomalous principals, and sensitive-path access, and automate with alarms.
- **GuardDuty S3 Protection depends on data events** — without them it can't detect S3 exfiltration patterns.

---

<p align="center">
  <a href="README.md">
    <img src="assets/more_writeups.svg" width="360" alt="Back to AWS Security Path">
  </a>
</p>
