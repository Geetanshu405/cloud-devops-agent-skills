---
name: rollback-bash-generator
description: Generate a single production-ready AWS rollback Bash script that reverses a specific remediation for a given Check_ID. Output must match the coding style of the DevOps team's validated rollback scripts — minimal, parameter-driven, direct AWS CLI, and honest about what can and cannot be safely reversed. Use whenever a previously applied remediation needs to be undone for a given Check_ID.
---

# AWS Rollback Script Generator

## Purpose

Generate exactly one Bash rollback script per finding, undoing what a specific remediation did, in the style of the DevOps team's hand-written, production-approved rollback scripts. Output the script only — no re-explanation of the remediation, no markdown fences, no metadata.

**A rollback is not automatically "the remediation script backwards."** Some remediations can be reversed deterministically. Some can't be reversed at all, even manually. Some were never automated in the first place, so there's nothing to undo. Getting this classification right matters more than the Bash itself — see "Rollback Feasibility Decision Framework" below.

**The reference examples teach structure and style only — never treat a matching Check_ID as a cue to reuse an example's script.** Rebuild from the actual remediation and parameters given in the current request, even when a Check_ID matches one below exactly.

## Execution Environment

Same platform as the remediation scripts: Bash, AWS CLI, valid credentials, and parameters are already provided. Never generate dependency checks, package installs, or environment validation — go straight to the rollback logic.

## Before Writing Any Rollback Script

Work through these in order, using only the remediation and parameters given for this request:

1. **What exact AWS API call(s) did the original remediation make, with what parameters?** Not what the Check_ID name suggests — the actual call.
2. **Is the pre-remediation state fully known and deterministic?** A specific policy ARN that was detached, a specific flag that was set, a specific rule with a fixed CIDR/protocol — all reversible. An unknown/arbitrary prior configuration the remediation overwrote or deleted without recording it first — not reversible.
3. **Did the remediation make an automated change at all**, or was it itself manual-only guidance? If the latter, there's nothing to roll back.
4. **Classify into one of the three categories below**, then write only what that category calls for.

## Script Structure

Identical Bash conventions to the remediation skill:

- No `set -e`/strict-mode header on a single-call script; `set -e` only for multiple sequential dependent steps.
- Validate only the parameters this specific rollback uses, with the **same parameter names the paired remediation used** — never invent new names for the same identifiers. Drop any remediation parameter that isn't needed for the reverse (e.g. a remediation that took a source and target bucket may only need the source bucket to roll back).
- One execution message, the direct `aws` call(s), one completion message. No progress narration.
- A literal used once is inlined at its call site, not assigned to a variable, unless it's captured from a live API response.
- Multi-line JSON payloads go through a heredoc to `/tmp`, never inline string concatenation.
- `exit 0` for a completed or not-applicable rollback; `exit 1` only for Category B (not supported).

## Never Generate

- `describe-*`/`get-*`/`list-*` calls or capture variables when the identifier is already a parameter.
- jq, shell functions, loops, nested conditionals.
- A rollback that guesses at a prior state it can't actually know. If you can't name the exact value being restored, it belongs in Category B or C, not Category A.
- A fabricated AWS command that doesn't exist, in either an automated or a manual-guidance script.

## Rollback Feasibility Decision Framework

### Category A — Automated Reverse

Use only when the pre-remediation state is fully deterministic: a specific named policy, a specific flag/config value, a specific rule with a fixed CIDR or protocol. Structure exactly like a remediation script — validate, one exec message, direct reverse `aws` call(s), one completion message. Automating a reverse is allowed to restore a _less secure_ prior state (see Example 4) — that's the nature of rollback — but only when that prior state is known, not inferred.

### Category B — Rollback Not Supported

Use when the original remediation itself required manual/root/interactive action, or when the change is inherently irreversible regardless of who performs it (e.g. a deleted access key). Mirrors the remediation skill's own manual pattern: a `====` banner, `"ROLLBACK NOT SUPPORTED"`, a short explanation of why, a manual command reference if one genuinely exists, and `exit 1`.

### Category C — Rollback Not Available / Not Applicable

Use when either: the remediation took an automated action but consumed or overwrote an _unknown_ prior state it never recorded (so there's nothing concrete to restore), or the remediation made no automated change at all (pure guidance). Both land on the same short output — one or two `echo` lines, `exit 0`, no banner. This is not a failure state; it's an honest acknowledgment that nothing can or needs to be undone.

## Exception: Full-Replacement APIs Default to Category B

Any remediation that used a full-replacement API under the check-before-overwrite exception (`put-bucket-policy`, `put-bucket-acl`, `put-key-policy`, and similar) gets a **Category B, Not Supported** rollback by default — not an automated delete. Deleting or reverting the policy risks removing statements someone else added since the remediation ran, and there's no reliable way to confirm the current policy is still exactly what the remediation created. State this in the manual message and point the operator to the specific statement `Sid` the remediation added, so they can verify and remove it by hand if it's still there. See Example 8.

## Reference Examples — Style Only, Never Copy Verbatim

### Example 1 — Category A: simple direct reverse

`Check_ID: s3_account_level_public_access_blocks`

```bash
#!/bin/bash

if [ -z "$ACCOUNT_ID" ] || [ -z "$REGION" ]; then
    echo "Usage: $0 <AccountID> <Region>"
    exit 1
fi

echo "Removing S3 Block Public Access configuration..."

aws s3control delete-public-access-block \
    --account-id "$ACCOUNT_ID" \
    --region "$REGION"

echo "Rollback completed."
```

### Example 2 — Category A: reverse of attach/detach

`Check_ID: iam_policy_cloudshell_admin_not_attached`

```bash
#!/bin/bash
set -e

if [ -z "$IAM_USER_NAME" ]; then
    echo "Required parameter IAM_USER_NAME is missing."
    exit 1
fi

POLICY_ARN="arn:aws:iam::aws:policy/AWSCloudShellFullAccess"

echo "Re-attaching AWSCloudShellFullAccess policy..."

aws iam attach-user-policy \
    --user-name "$IAM_USER_NAME" \
    --policy-arn "$POLICY_ARN"

echo "Rollback completed successfully."
```

### Example 3 — Category A: multi-step teardown, `set -e` for dependent calls

`Check_ID: cloudwatch_log_metric_filter_disable_or_scheduled_deletion_of_kms_cmk`

```bash
#!/bin/bash
set -e

if [ -z "$LOG_GROUP_NAME" ] || [ -z "$REGION" ]; then
    echo "Usage: $0 <LogGroupName> <Region>"
    exit 1
fi

echo "Removing CloudWatch alarm and metric filter..."

aws cloudwatch delete-alarms \
    --region "$REGION" \
    --alarm-names "KMSDisableOrDeleteCMKAlarm"

aws logs delete-metric-filter \
    --region "$REGION" \
    --log-group-name "$LOG_GROUP_NAME" \
    --filter-name "KMSDisableOrDeleteCMK"

echo "Rollback completed successfully."
```

Note: `LOG_GROUP_NAME`/`REGION` validation is added here beyond what the raw guide script had — the guide's version used both without validating either, which is a gap worth closing rather than reproducing.

### Example 4 — Category A: restoring a known but less-secure prior state

`Check_ID: ec2_securitygroup_allow_ingress_from_internet_to_all_ports`

```bash
#!/bin/bash

if [ -z "$SG_ID" ] || [ -z "$REGION" ]; then
    echo "Required parameters not provided."
    echo "SG_ID=$SG_ID"
    echo "REGION=$REGION"
    exit 1
fi

echo "Restoring unrestricted inbound access to Security Group: $SG_ID"

# Restore IPv4 rule
aws ec2 authorize-security-group-ingress \
    --group-id "$SG_ID" \
    --region "$REGION" \
    --ip-permissions '[{"IpProtocol":"-1","IpRanges":[{"CidrIp":"0.0.0.0/0"}]}]'

# Restore IPv6 rule
aws ec2 authorize-security-group-ingress \
    --group-id "$SG_ID" \
    --region "$REGION" \
    --ip-permissions '[{"IpProtocol":"-1","Ipv6Ranges":[{"CidrIpv6":"::/0"}]}]'

echo "Rollback completed."
```

This one deliberately reopens the security group to the internet — that's correct rollback behavior here because the original open rule was a specific, fully known CIDR pair, not a guess.

### Example 5 — Category B: manual judgment required

`Check_ID: ec2_instance_profile_attached`

```bash
#!/bin/bash

echo "========================================="
echo "ROLLBACK NOT SUPPORTED"
echo "========================================="

echo "Detaching an IAM Instance Profile automatically is not recommended."
echo ""
echo "If required, manually remove the instance profile using:"
echo ""
echo "aws ec2 disassociate-iam-instance-profile \\"
echo "    --association-id $ASSOCIATION_ID"

exit 1
```

### Example 6 — Category B: inherently irreversible

`Check_ID: iam_rotate_access_key_90_days`

```bash
#!/bin/bash

echo "========================================="
echo "ROLLBACK NOT SUPPORTED"
echo "========================================="

echo "Deleted IAM access keys cannot be restored."
echo "If the old key has been deleted, create a new access key and update all applications accordingly."

exit 1
```

Same banner pattern as Example 5, but the underlying reason is different: this isn't "we chose not to automate," it's "the data needed to reverse this is gone." Both still resolve to Category B — the operator needs to know either way.

### Example 7 — Category C: state lost vs. never changed

`Check_ID: ec2_securitygroup_default_restrict_traffic`

```bash
#!/bin/bash

echo "Rollback is not available."
```

`Check_ID: iam_policy_attached_only_to_group_or_roles`

```bash
#!/bin/bash

echo "Rollback Not Applicable"
exit 0
```

Both are one line, both resolve gracefully — no banner, no parameters, no drama. The first is here because the remediation removed an unknown set of existing inbound rules without recording them, so there's nothing concrete to restore. The second is here because the remediation itself never made an automated change — it was pure manual guidance — so there's nothing to undo.

### Example 8 — Full-replacement API defaults to Not Supported

`Check_ID: s3_bucket_secure_transport_policy`

```bash
#!/bin/bash

echo "========================================="
echo "ROLLBACK NOT SUPPORTED"
echo "========================================="

echo "This remediation added a DenyInsecureTransport statement to the bucket policy."
echo "Automatically removing it risks deleting policy statements added by someone else since."
echo ""
echo "To roll back manually:"
echo "1. Run: aws s3api get-bucket-policy --bucket $BUCKET_NAME"
echo "2. Confirm the statement with Sid \"DenyInsecureTransport\" is still present and unchanged."
echo "3. Remove only that statement and re-apply the policy with put-bucket-policy."

exit 1
```

## Quality Checklist

- Correctly classified into Category A, B, or C — not defaulted to "automate it" because that's the more impressive-looking output
- Category A: pre-remediation state is genuinely deterministic, not inferred
- Category B: `====` banner, clear reason, `exit 1`
- Category C: one or two lines, no banner, `exit 0`
- Same parameter names as the paired remediation, trimmed to only what the reverse needs
- No single-use literal pulled into a variable unless it's a live API response
- Full-replacement-API remediations default to Category B, referencing the specific `Sid` added
- Script length and tone match the reference examples
