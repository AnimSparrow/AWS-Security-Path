<p align="center">
  <img src="assets/banner-oversharing-container.svg" width="820" alt="The Oversharing Container">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/TRYHACKME-1a0633?style=for-the-badge&labelColor=FF2A6D">
  <img src="https://img.shields.io/badge/EASY-1a0633?style=for-the-badge&labelColor=00F0FF">
  <img src="https://img.shields.io/badge/ECS-1a0633?style=for-the-badge&labelColor=A479C4">
  <img src="https://img.shields.io/badge/WORKLOAD_IDENTITY-1a0633?style=for-the-badge&labelColor=A479C4">
</p>

---

## Overview

> [!NOTE]
> **TL;DR** — An ECS task definition with no `taskRoleArn` falls back to the EC2 instance role — platform credentials far broader than any container should hold. You audit the service, spot the `null` task role, create a scoped task role and register a corrected revision, then build a clean identity split (separate execution and task roles) from the start. Framed by TeamTNT's 2021 Hildegard campaign.

## The three ECS identity roles

| Role | Who it's for | What it does |
|---|---|---|
| **Container instance role** | the EC2 host | lets the host register with ECS |
| **Task execution role** | ECS infrastructure | pull image from ECR, ship logs to CloudWatch |
| **Task role** | the application workload | the app's own scoped AWS permissions |

Omit the **task role** and the workload borrows the **instance role** instead.

## Real-world incident — Hildegard / TeamTNT (2021)

TeamTNT's first at-scale Kubernetes campaign scanned for unauthenticated API servers and kubelets, deployed malicious containers via the cluster's own scheduler, then from inside a pod queried `169.254.169.254` to steal the **EC2 node's** IAM credentials — because pods had no per-pod identity (IRSA, the Kubernetes equivalent of `taskRoleArn`), every pod on a node shared the node role. Those creds enumerated AWS, and Hildegard dropped XMRig plus a reverse shell, disguised as a known Linux process.

> [!IMPORTANT]
> The core architectural failure is identical to an ECS task without `taskRoleArn`: **no workload identity isolation**, so application containers inherit platform credentials through the metadata endpoint. An over-privileged node role maximises the blast radius of any single container compromise.

## Identification — inspect the task definition

```bash
TASK_DEF_ARN=$(aws ecs describe-services --cluster "$CLUSTER_NAME" --services "$SERVICE_NAME" \
  --query "services[0].taskDefinition" --output text)   # → room44-app:1 (original revision)

aws ecs describe-task-definition --task-definition "$TASK_DEF_ARN" \
  --query "taskDefinition.{Family:family,TaskRole:taskRoleArn,ExecutionRole:executionRoleArn}"
#   TaskRole: null                          ← falls back to the instance role (the host)
#   ExecutionRole: room44-lab-execution-role ← execution role exists, task role does not

# sensitive data hiding in the container env vars (readable with ecs:DescribeTaskDefinition)
aws ecs describe-task-definition --task-definition "$TASK_DEF_ARN" \
  --query "taskDefinition.containerDefinitions[0].environment[?name=='FLAG'].value" --output text
```

**Findings:** `taskRoleArn: null` means the container inherits the **host** (EC2 instance) role, exactly the Hildegard path. And the flag sits in a plaintext container **environment variable**, visible to anyone with `ecs:DescribeTaskDefinition` — a reminder that secrets belong in Secrets Manager, not task-definition env vars.

## Remediation — give the container its own identity

```bash
# 1. scoped task role: trust ecs-tasks.amazonaws.com + a boundary, minimal S3 policy
aws iam create-role --role-name room44-app-task-role \
  --assume-role-policy-document file://trust.json \
  --permissions-boundary arn:aws:iam::$ACCOUNT_ID:policy/Room44-LearnerBoundary
aws iam put-role-policy --role-name room44-app-task-role --policy-name ... --policy-document file://policy.json
TASK_ROLE_ARN=$(aws iam get-role --role-name room44-app-task-role --query 'Role.Arn' --output text)

# 2. register a new revision carrying taskRoleArn (jq strips the read-only fields first)
aws ecs describe-task-definition --task-definition "$TASK_DEF_ARN" --query taskDefinition > td.json
jq --arg r "$TASK_ROLE_ARN" \
  'del(.taskDefinitionArn,.revision,.status,.requiresAttributes,.compatibilities,.registeredAt,.registeredBy)
   | .taskRoleArn=$r' td.json > td-new.json
NEW=$(aws ecs register-task-definition --cli-input-json file://td-new.json --query 'taskDefinition.taskDefinitionArn' --output text)  # → room44-app:2

# 3. point the service at the new revision
aws ecs update-service --cluster "$CLUSTER_NAME" --service "$SERVICE_NAME" --task-definition "$NEW"
```

> [!TIP]
> Task roles trust **`ecs-tasks.amazonaws.com`** (ECS assumes the role on the task's behalf), not `ec2`. And task definitions are **immutable** — you don't edit `:1`, you register `:2`; `jq` must drop the read-only fields (`taskDefinitionArn`, `revision`, `status`, `requiresAttributes`, `compatibilities`, `registeredAt`, `registeredBy`) or `register-task-definition` rejects the input.

## Build it securely — split execution and task roles

```bash
# execution role: ECS infra only (ECR pull, CloudWatch logs) — AWS-managed policy
aws iam create-role --role-name room44-secure-execution-role --assume-role-policy-document file://trust.json --permissions-boundary $BOUNDARY
aws iam attach-role-policy --role-name room44-secure-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
# task role: only what the app calls (scoped S3 read)
aws iam create-role --role-name room44-secure-task-role --assume-role-policy-document file://trust.json --permissions-boundary $BOUNDARY
aws iam put-role-policy --role-name room44-secure-task-role ...
# task definition with BOTH roles set explicitly
aws ecs register-task-definition --cli-input-json file://secure-taskdef.json
```

Four questions first: which service assumes it (`ecs-tasks.amazonaws.com` for both), which API actions the app actually needs, which specific resources, and what conditions further limit it.

## Task answers

| # | Task | Question | Answer |
|---|------|----------|--------|
| 1 | Introduction | (intro) | *(no answer)* |
| 2 | Real-World Incident | Field whose absence causes instance-role fallback | `taskRoleArn` |
| 3 | Identification | Whose role the container inherits when `null` | `host` |
| 3 | Identification | Flag from the container env var | `THM{BORROWED_********}` |
| 4 | Remediation | Flag | `THM{CAUGHT_UP_WITH_*****}` |
| 5 | Build It Securely | Flag | `THM{CLEAN_AND_*****}` |
| 6 | Conclusion | No more borrowed identities | *(no answer)* |

## Lessons learned

- **Never omit `taskRoleArn` on tasks that call AWS APIs** — without it the workload silently inherits the EC2 instance role and its full permissions.
- **Keep execution and task roles separate** — infra actions vs application actions are different problems.
- **Scope the task role to exact actions/resources**, not a broad managed policy, and cap it with a permission boundary.
- **Secrets don't belong in task-definition env vars** — anyone with `ecs:DescribeTaskDefinition` can read them.

---

<p align="center">
  <a href="README.md">
    <img src="assets/more_writeups.svg" width="360" alt="Back to AWS Security Path">
  </a>
</p>
