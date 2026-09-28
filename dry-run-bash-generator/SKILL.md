---
name: dry-run-bash-generator
description: Generate a single production-ready AWS dry-run Bash script that validates whether a remediation for a given Check_ID would succeed, without applying any change. Output must match the coding style of the DevOps team's validated dry-run scripts — minimal, parameter-driven, read-only unless a native --dry-run flag exists. Use whenever a remediation needs to be previewed or validated before it runs for a given Check_ID.
---

# AWS Dry-Run Script Generator

## Purpose

Generate exactly one Bash dry-run script per finding that checks whether the paired remediation would succeed, without changing anything. Output the script only — no markdown fences, no explanation, no metadata.

**This skill inverts a rule from the remediation and rollback skills.** There, `describe-*`/`get-*`/`list-*` calls are forbidden by default because the identifier is already known and a read call would be pointless discovery. Here, that read call _is the entire point_ — a dry-run script that makes no read call has done nothing useful. Don't carry the "no discovery" instinct over from the other two skills.

**The reference examples teach structure and style only — never treat a matching Check_ID as a cue to reuse an example's script.** Rebuild from the actual finding and parameters given in the current request.

## Execution Environment

Same platform as the remediation and rollback scripts: Bash, AWS CLI, valid credentials, and parameters are already provided. Never generate dependency checks, package installs, or environment validation.

## Before Writing Any Dry-Run Script

1. **What exact AWS API call(s) would the paired remediation make?** Get this from the actual remediation logic for this Check_ID, not from the name alone.
2. **Does that specific mutating API accept a native `--dry-run` flag?** Most EC2 mutating actions do (`revoke-security-group-ingress`, `authorize-security-group-ingress`, etc.). S3, IAM, CloudWatch Logs, and SNS generally don't.
3. **If yes → Category A.** If no, is there a `get-*`/`describe-*`/`list-*` call that reports the exact state the remediation would change? **→ Category B.**
4. **If the paired remediation is itself manual-only** (no automated action, human-judgment or root-required), this is **Category C** — a status report, not a functional dry run. Say so explicitly in the output rather than implying a real validation happened.
5. **Reuse the exact parameter names the paired remediation script uses for this Check_ID.** Never invent new ones.

## Script Structure

- No `set -e` unless the script has multiple dependent steps.
- Validate only the parameters this script uses, with the same names the remediation uses.
- Every message is prefixed `DRY RUN:` so log output is unambiguous — this never silently looks like a real change was applied.
- One or, at most, two read-only/dry-run calls. If a Check_ID seems to need more than that, the classification is probably wrong, not the call count.
- A literal used once is inlined at its call site, not assigned to a variable.
- `exit 0` throughout, including when the check finds a problem — completing the check is success; only a missing parameter is `exit 1`. A dry run reporting "this isn't configured yet" is not a script failure.

## Never Generate

- Any call that mutates state, under any framing — the one exception is a mutating call carrying `--dry-run` in a genuine Category A case.
- Compliance verdicts computed with conditional logic. Print the raw result of the read call and a one-line statement of what remediation would do with it; let the caller (human or downstream automation) interpret the result. No `if`/`grep`/parsing to decide pass/fail.
- jq, loops, functions.

## Dry-Run Classification Framework

### Category A — Native Dry-Run

The mutating API itself accepts `--dry-run`. The dry-run script is the remediation's own call, parameters unchanged, with `--dry-run` appended — no new logic. AWS returns a deliberate `DryRunOperation` response if the caller is authorized, or a real authorization error if not. Note: this response is technically non-zero exit even on the "you're authorized" path — don't add `set -e` here, and don't try to branch on the exit code without confirming with DevOps how they want that surfaced downstream.

### Category B — Read-Only Equivalent

No native dry-run exists, but a `get-*`/`describe-*`/`list-*` call reports the exact current state the remediation would change. Run it, print the result, and state in one line what the remediation would do differently. Don't compute or assert compliance — just show current state next to intended state and let the reader compare.

### Category C — Informational Only

The paired remediation is manual-only. There's no automated action to validate, so this script is a status report to inform a human decision, not a real dry run. State that plainly in the completion message rather than implying more happened than did.

## Reference Examples — Style Only, Never Copy Verbatim

### Example 1 — Category B: read-only state check

`Check_ID: s3_account_level_public_access_blocks`

```bash
#!/bin/bash

if [ -z "$ACCOUNT_ID" ] || [ -z "$REGION" ]; then
    echo "Usage: $0 <AccountID> <Region>"
    exit 1
fi

echo "DRY RUN: Fetching current S3 Block Public Access configuration for account: $ACCOUNT_ID"

aws s3control get-public-access-block \
    --account-id "$ACCOUNT_ID" \
    --region "$REGION"

echo "DRY RUN complete. No changes applied. Remediation would set BlockPublicAcls, IgnorePublicAcls, BlockPublicPolicy, and RestrictPublicBuckets to true."
```

### Example 2 — Category C: informational only, manual remediation

`Check_ID: ec2_instance_profile_attached`

```bash
#!/bin/bash

if [ -z "$INSTANCE_ID" ]; then
    echo "Required parameter INSTANCE_ID is missing."
    exit 1
fi

echo "DRY RUN: Checking IAM instance profile association for instance: $INSTANCE_ID"

aws ec2 describe-iam-instance-profile-associations \
    --filters "Name=instance-id,Values=$INSTANCE_ID"

echo "DRY RUN complete. No changes applied. This finding requires manual selection of the correct IAM Instance Profile — see remediation guidance."
```

### Example 3 — Category B: targeted query against a broader read call

`Check_ID: iam_policy_cloudshell_admin_not_attached`

```bash
#!/bin/bash

if [ -z "$IAM_USER_NAME" ]; then
    echo "Required parameter IAM_USER_NAME is missing."
    exit 1
fi

echo "DRY RUN: Checking whether AWSCloudShellFullAccess is attached to user: $IAM_USER_NAME"

aws iam list-attached-user-policies \
    --user-name "$IAM_USER_NAME" \
    --query "AttachedPolicies[?PolicyArn=='arn:aws:iam::aws:policy/AWSCloudShellFullAccess']"

echo "DRY RUN complete. No changes applied. Remediation would detach this policy if it appears above."
```

### Example 4 — Category B: two read calls mirroring a two-call remediation

`Check_ID: cloudwatch_log_metric_filter_disable_or_scheduled_deletion_of_kms_cmk`

```bash
#!/bin/bash

if [ -z "$LOG_GROUP_NAME" ] || [ -z "$REGION" ]; then
    echo "Usage: $0 <LogGroupName> <Region>"
    exit 1
fi

echo "DRY RUN: Checking for an existing KMS disable/delete metric filter and alarm in log group: $LOG_GROUP_NAME"

aws logs describe-metric-filters \
    --region "$REGION" \
    --log-group-name "$LOG_GROUP_NAME" \
    --filter-name-prefix "KMSDisableOrDeleteCMK"

aws cloudwatch describe-alarms \
    --region "$REGION" \
    --alarm-names "KMSDisableOrDeleteCMKAlarm"

echo "DRY RUN complete. No changes applied. Remediation would create the metric filter and alarm above if not already present."
```

Two read calls are fine here because the remediation itself makes two mutating calls — the dry run mirrors that shape exactly. Don't add calls beyond what the remediation's own structure justifies.

### Example 5 — Category B despite being an EC2 call: discovery is inherent to the remediation, not a dry-run add-on

`Check_ID: ec2_securitygroup_default_restrict_traffic`

```bash
#!/bin/bash

if [ -z "$SECURITY_GROUP_ID" ] || [ -z "$REGION" ]; then
    echo "Usage: $0 <SecurityGroupID> <Region>"
    exit 1
fi

echo "DRY RUN: Fetching current inbound rules for Security Group: $SECURITY_GROUP_ID"

aws ec2 describe-security-groups \
    --group-ids "$SECURITY_GROUP_ID" \
    --region "$REGION" \
    --query "SecurityGroups[0].IpPermissions"

echo "DRY RUN complete. No changes applied. Remediation would remove all inbound rules shown above."
```

Not Category A even though it's an EC2 service: the remediation itself has no fixed rule to target — it has to `describe-security-groups` first to find out what exists before it can revoke it. The dry run is that same describe call, not a `--dry-run` flag on a revoke.

### Example 6 — Category A: the clean case, remediation's own call plus one flag

`Check_ID: ec2_securitygroup_allow_ingress_from_internet_to_all_ports`

```bash
#!/bin/bash

if [ -z "$SG_ID" ] || [ -z "$REGION" ]; then
    echo "Required parameters not provided."
    echo "SG_ID=$SG_ID"
    echo "REGION=$REGION"
    exit 1
fi

echo "DRY RUN: Validating permission to remove unrestricted inbound access from Security Group: $SG_ID"

aws ec2 revoke-security-group-ingress \
    --group-id "$SG_ID" \
    --region "$REGION" \
    --ip-permissions '[{"IpProtocol":"-1","IpRanges":[{"CidrIp":"0.0.0.0/0"}]}]' \
    --dry-run

aws ec2 revoke-security-group-ingress \
    --group-id "$SG_ID" \
    --region "$REGION" \
    --ip-permissions '[{"IpProtocol":"-1","Ipv6Ranges":[{"CidrIpv6":"::/0"}]}]' \
    --dry-run

echo "DRY RUN complete. No changes applied."
```

This is Category A because the remediation's target permissions are fixed and known (0.0.0.0/0 and ::/0) — no discovery needed either way, so the dry run is literally the remediation's two calls with `--dry-run` appended to each.

### Example 7 — Category C: multiple read calls acceptable when the review genuinely spans more than one policy type

`Check_ID: iam_policy_attached_only_to_group_or_roles`

```bash
#!/bin/bash

if [ -z "$IAM_USER_NAME" ]; then
    echo "Required parameter IAM_USER_NAME is missing."
    exit 1
fi

echo "DRY RUN: Listing policies directly attached to user: $IAM_USER_NAME"

aws iam list-attached-user-policies \
    --user-name "$IAM_USER_NAME"

aws iam list-user-policies \
    --user-name "$IAM_USER_NAME"

echo "DRY RUN complete. No changes applied. This finding requires manual policy review — see remediation guidance."
```

## Quality Checklist

- Correctly classified into Category A, B, or C — not defaulted to A because a `--dry-run` flag exists on some related API when the paired remediation doesn't actually use it
- Every `echo` is prefixed `DRY RUN:` or states "DRY RUN complete" — never ambiguous with a real remediation's output
- No conditional logic computing a compliance verdict — the script reports; it doesn't decide
- Parameter names match the paired remediation exactly
- `exit 0` in all cases except a missing parameter
- Category C explicitly states no automated action would run, rather than implying a full validation happened
- Script length and tone match the reference examples
