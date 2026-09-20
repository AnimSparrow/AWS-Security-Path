<p align="center">
  <img src="assets/banner-unpatched-instance.svg" width="820" alt="The Unpatched Instance">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/EASY-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/PATCH_MANAGER-1a0633?style=for-the-badge&labelColor=A479C4">
  <img src="https://img.shields.io/badge/GOLDEN_AMI-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

## Overview

> [!NOTE]
> **TL;DR** — An instance launched from a two-year-old AMI carries silent patch debt: no maintenance window, no schedule. You quantify the debt with Patch Manager, install fixes via `AWS-RunPatchBaseline`, then codify a repeatable process with a custom patch baseline and a patched golden AMI. Framed by WannaCry / the 2017 NHS outbreak.

## Real-world incident — WannaCry / NHS (2017)

Microsoft published **MS17-010** (a critical SMBv1 RCE) in March; the patch was free and documented. Two months later WannaCry — carrying the **EternalBlue** exploit (NSA tool leaked by the Shadow Brokers) — swept **TCP/445 with SMBv1** enabled, needing no user interaction. ~80 of 236 NHS England trusts were hit, ~19,000 appointments cancelled, staff reverting to pen and paper. 200,000+ systems in 150 countries within hours.

> [!IMPORTANT]
> The fix existed for two months. The failures were **no repeatable patch process**, a **legacy protocol left enabled** (SMBv1), **no segmentation** (flat network → worm spread), and an **unmanaged host inventory** (patch state unknown until the attack revealed it). In AWS, none of this is a tooling gap — Patch Manager, Run Command and golden AMIs exist precisely for this.

## Identification — quantify the patch debt

```bash
# 1. instance must be an SSM-managed node
aws ssm describe-instance-information --filters "Key=InstanceIds,Values=$INSTANCE_ID" \
  --query "InstanceInformationList[0].{Ping:PingStatus,Platform:PlatformName,Agent:AgentVersion}" --output table   # Online

# 2. scan (Patch Manager must build a record first)
SCAN_COMMAND_ID=$(aws ssm send-command --instance-ids "$INSTANCE_ID" \
  --document-name AWS-RunPatchBaseline --parameters Operation=Scan --query "Command.CommandId" --output text)
aws ssm get-command-invocation --command-id "$SCAN_COMMAND_ID" --instance-id "$INSTANCE_ID" \
  --query "{Status:Status,StatusDetails:StatusDetails}"   # poll until Success

# 3. read the patch state
aws ssm describe-instance-patch-states --instance-ids "$INSTANCE_ID" \
  --query "InstancePatchStates[0].{Missing:MissingCount,Installed:InstalledCount,Failed:FailedCount}"
```

> [!TIP]
> `describe-instance-patch-states` returns empty while the scan is still `InProgress` — **poll `get-command-invocation` until `Success` before reading the state.** (The task flag lives in an EC2 tag, so it's retrievable regardless of scan timing.)

## Remediation — install via Run Command

```bash
# same document, Operation=Install instead of Scan
PATCH_COMMAND_ID=$(aws ssm send-command --instance-ids "$INSTANCE_ID" \
  --document-name AWS-RunPatchBaseline --parameters Operation=Install --query "Command.CommandId" --output text)
# poll to Success (2-5 min), then re-check: Missing should drop to 0
aws ssm get-command-invocation --command-id "$PATCH_COMMAND_ID" --instance-id "$INSTANCE_ID" --query "{Status:Status}"
aws ssm describe-instance-patch-states --instance-ids "$INSTANCE_ID" \
  --query "InstancePatchStates[0].{Missing:MissingCount,Installed:InstalledCount,Failed:FailedCount}"   # Missing:0
```

The only difference from the scan is one parameter — **`Operation=Scan`** (report) vs **`Operation=Install`** (apply). Always poll to `Success` before re-checking.

## Build it securely — baseline + golden AMI

Patching a live instance is half the fix; the next launch from the same stale AMI starts unpatched. Two pillars close the loop. Ask first: which OS / how many distros? which classifications + severities? how fast to approve after release? golden AMI or in-place only?

```bash
# 1. custom patch baseline = policy as code
aws ssm create-patch-baseline --name room42-linux-baseline --operating-system AMAZON_LINUX_2023 \
  --approval-rules 'PatchRules=[{PatchFilterGroup={PatchFilters=[
    {Key=CLASSIFICATION,Values=[Security,Bugfix]},
    {Key=SEVERITY,Values=[Critical,Important,Medium]}]},ApproveAfterDays=0,ComplianceLevel=CRITICAL}]'
# 2. golden AMI = patched state baked into every future launch
aws ec2 create-image --instance-id "$INSTANCE_ID" --name patched-baseline-ami --no-reboot
```

This is the immutable-infrastructure pattern: patches go into a **new template**, not into long-lived servers.

## Task answers

| # | Task | Question | Answer |
|---|------|----------|--------|
| 1 | Introduction | The known vulnerability has a published fix | *(no answer)* |
| 2 | Real-World Incident | Protocol at the core of the attack | `SMB` |
| 2 | Real-World Incident | Exploit used to propagate | `EternalBlue` |
| 3 | Identification | API showing the missing patch count | `describe-instance-patch-states` |
| 3 | Identification | Flag | `THM{BACKDOOR_**********}` |
| 4 | Remediation | Flag from the verifier | `THM{NO_MORE_*****}` |
| 5 | Build It Securely | Flag from the secure build | `THM{CLEAN_LAUNCH_********}` |
| 6 | Conclusion | The only winning move is to patch | *(no answer)* |

## Lessons learned

- **Patching is a process, not a one-off** — a defined schedule plus a verification step.
- **`AWS-RunPatchBaseline` + Run Command** gives auditable, on-demand patching of SSM-managed nodes.
- **Always verify with `get-command-invocation`** — submitting a command is not the same as confirming success.
- **Custom baselines codify the policy; golden AMIs stop new instances inheriting debt** from a stale image.

---

<p align="center">
  <a href="README.md">
    <img src="assets/more_writeups.svg" width="360" alt="Back to AWS Security Path">
  </a>
</p>
