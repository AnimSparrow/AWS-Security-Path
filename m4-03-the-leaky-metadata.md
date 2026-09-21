<p align="center">
  <img src="assets/banner-leaky-metadata.svg" width="820" alt="The Leaky Metadata">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/EASY-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/IMDS-1a0633?style=for-the-badge&labelColor=A479C4">
  <img src="https://img.shields.io/badge/SSRF-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

## Overview

> [!NOTE]
> **TL;DR** — With `HttpTokens=optional`, any process that can send a plain HTTP `GET` to `169.254.169.254` can pull temporary IAM credentials — which is how an SSRF becomes credential theft. You confirm the open IMDSv1 path, retrieve creds through it, enforce IMDSv2 so the plain `GET` returns `401`, and launch a new instance that requires IMDSv2 from the start. Framed by the 2019 Shopify SSRF report.

## Real-world incident — Shopify SSRF (2019)

A HackerOne researcher found a Shopify feature that fetched a user-supplied URL server-side from an EC2 instance with an attached IAM role. Supplying `http://169.254.169.254/latest/meta-data/iam/security-credentials/` pointed that request at the metadata endpoint. Because the instance ran **`HttpTokens=optional`**, a plain `GET` returned the role name, and a follow-up returned a full set of temporary credentials (`AccessKeyId`, `SecretAccessKey`, `SessionToken`). Reported before any data was touched; Shopify fixed the SSRF and moved to IMDSv2.

> [!IMPORTANT]
> Without IMDSv2, a **single SSRF bug becomes a direct path to IAM credential theft** — no privilege escalation, no lateral movement. The other failures (no SSRF input validation blocking `169.254.0.0/16`, no network control stopping the app tier from reaching IMDS) stack on top, but IMDSv2 alone breaks the chain.

## Identification — retrieve creds through IMDSv1

```bash
# check the metadata options
aws ec2 describe-instances --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].MetadataOptions"
#   "HttpTokens": "optional"   ← IMDSv1 allowed (RED FLAG)

# from inside the instance (SSM): two plain GETs = full credentials
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/            # → role name
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/$ROLE       # → AccessKeyId + SecretAccessKey + Token
```

`HttpTokens: optional` means **two HTTP requests and you have temporary credentials** — exactly the Shopify path. No token, no auth.

## Remediation — enforce IMDSv2

```bash
# one switch, no reboot, transparent to the running app
aws ec2 modify-instance-metadata-options --instance-id "$INSTANCE_ID" \
  --http-tokens required --http-endpoint enabled
#   "HttpTokens": "required"

# the old plain GET is now rejected
curl -I http://169.254.169.254/latest/meta-data/iam/security-credentials/$ROLE
#   HTTP/1.1 401 Unauthorized     ← IMDSv1 blocked

# the token-based IMDSv2 flow still works
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/$ROLE   # → credentials
```

A **`401`** on the plain `GET` is the win: a GET-only SSRF forwarder cannot perform the required `PUT` token exchange, so it can no longer retrieve credentials.

> [!TIP]
> IMDSv2 applies immediately and needs no reboot, but credentials already issued via IMDSv1 stay valid for up to 6 hours, so enforcement isn't instantly retroactive.

## Build it securely — IMDSv2 from launch

```bash
aws ec2 run-instances --image-id "$AMI_ID" --instance-type t3.micro \
  --subnet-id "$SUBNET_ID" --security-group-ids "$SECURE_SG" \
  --iam-instance-profile Name=Room43ManagedInstanceProfile \
  --metadata-options "HttpTokens=required,HttpEndpoint=enabled" \
  --no-associate-public-ip-address
```

Enforcing on a running instance is one step; the stronger control is making sure **no future instance can launch with `optional`** — set `HttpTokens=required` at launch and enforce it through launch templates and SCPs. Pair with least-privilege instance roles so even leaked creds have a small blast radius.

## Task answers

| # | Task | Question | Answer |
|---|------|----------|--------|
| 1 | Introduction | You only need to redirect one request | *(no answer)* |
| 2 | Real-World Incident | HTTP method IMDSv2 requires for the token | `put` |
| 2 | Real-World Incident | Link-local IP the IMDS listens on | `169.254.169.254` |
| 3 | Identification | `HttpTokens` value that allows IMDSv1 | `optional` |
| 3 | Identification | Flag | `THM{OPEN_CHANNEL_*****}` |
| 4 | Remediation | HTTP code now returned by the plain `curl` | `401` |
| 4 | Remediation | Flag | `THM{CHANNEL_*******}` |
| 5 | Build It Securely | Flag | `THM{NO_REDIRECT_********}` |
| 6 | Conclusion | This time the plan includes a token exchange | *(no answer)* |

## Lessons learned

- **IMDSv2 is not just hardening — it's the specific control that breaks the SSRF-to-credential-theft chain.** Treat `HttpTokens=optional` as a finding.
- **Enforce it at launch, not just after the fact:** launch templates + SCPs stop new instances shipping with IMDSv1.
- **A GET-only SSRF can't do the PUT token exchange** — that's the whole design win of v2.
- **Layer with least-privilege roles**, so credentials that do leak reach as little as possible.

---

<p align="center">
  <a href="README.md">
    <img src="assets/more_writeups.svg" width="360" alt="Back to AWS Security Path">
  </a>
</p>
