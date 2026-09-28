---
name: validation-bash-generator
description: Generate a single production-ready AWS validation Bash script that confirms whether a remediation for a given Check_ID actually succeeded, with a real pass/fail exit code. Output must match the coding style of the DevOps team's validated verification scripts — parameter-driven, direct AWS CLI, explicit verdict logic. Use whenever a previously applied remediation needs to be confirmed as compliant for a given Check_ID.
---

# AWS Validation Script Generator

## Purpose

Generate exactly one Bash validation script per finding that checks whether a remediation actually took effect, and returns a real pass/fail exit code. Output the script only — no markdown fences, no explanation, no metadata.

**This is the one skill in the set where verdict logic is the point, not something to avoid.** Remediation and rollback act; dry-run reports raw state and leaves interpretation to the reader. Validation has to decide — `if`/`else`, boolean tracking, and an explicit "pass" or "fail" message are the actual mechanism here, not defensive bloat. Don't strip these out chasing the minimalism the other three skills enforce.

**The reference examples teach structure and style only — never treat a matching Check_ID as a cue to reuse an example's script.** Rebuild from the actual remediation and parameters given in the current request.

## Execution Environment

Same platform as the other three skills: Bash, AWS CLI, valid credentials, and parameters are already provided. Assume GNU `date` (Linux) if date arithmetic is needed — never hedge with BSD-syntax fallbacks. Never generate dependency checks or environment validation.

## Before Writing Any Validation Script

1. **What did the remediation actually change?** The exact API call(s) and the exact end state they were meant to produce — not what the Check_ID name implies.
2. **Is that end state a specific, checkable fact** (a flag value, a policy attachment, a rule that should no longer exist, an age threshold), or does confirming it "was done right" require human judgment the script can't evaluate (e.g. whether the _correct_ IAM role was chosen)?
3. **Classify into Category A, B, or C** (below) based on question 2 — not based on how the _remediation itself_ was classified in the other skills. A manual, judgment-based remediation can still have a fully deterministic Category A validation if the target end state is a hard fact.
4. **List every parameter needed.** Validation often needs more than the remediation did — it needs the _expected value_ to compare against, not just the resource identifier (e.g. which target bucket logging should point to). Check whether an expected-value parameter is missing before writing the comparison.

## Script Structure

- No `set -e` — a validation script's whole job is branching on command results; strict mode would abort exactly the flow it needs.
- Validate only the parameters used, reusing the remediation's exact names, plus any expected-value parameters identified in step 4 above.
- One or two read-only AWS CLI calls. If a check genuinely has multiple independent components (a filter and an alarm, for instance), one boolean flag per component, checked with a single `if`/`&&` at the end — mirrors the reference pattern in Example 5.
- `✓`/`✗` prefix on each per-component result line; a final "Verification successful" or "Verification failed" line before exiting.
- `exit 0` only when verification passes; `exit 1` for both a missing parameter and a failed verification. Downstream automation branches on this exit code, so it has to be accurate — never `exit 0` on a result you haven't actually confirmed.
- No jq. Prefer `--query` with `--output text` and a direct string/grep comparison over parsing raw JSON.

## Never Generate

- A verdict without a corresponding check — every `exit 0` must follow a real comparison against actual AWS state, never an assumption.
- Loops or functions — a flat sequence of `if` blocks and boolean flags covers every case in the reference set; if a check seems to need a loop, it's likely checking too many things for one script.
- Self-discovery of the resource being validated (e.g. searching for "the" relevant trail or bucket) when an identifier could instead be a parameter. Confirming the _right_ resource is compliant is the entire point — guessing which resource to check undermines that.
- Silent "existence-only" checks presented as full verification. If the script can only confirm something changed, not that it changed correctly, say so in the output (see Example 2).

## Validation Classification Framework

### Category A — Deterministic Verification

The remediation's intended end state is a specific, checkable fact — a flag, a named policy attachment, a rule absence, a date threshold. Read the current value, compare against the expected value, and assert pass/fail with no ambiguity. This is the default; use it whenever the end state is a hard fact, even if a human had to make judgment calls to get there.

### Category B — Existence-Only Verification

The remediation required human judgment (which IAM role to attach, which policy to write), so validation can confirm that _something_ now exists in the expected shape, but not that it's the _correct_ choice. State that limitation explicitly in the success message rather than implying full verification occurred.

### Category C — Not Applicable

The remediation made no automated state change at all — pure manual guidance. Nothing to verify. Short output, `exit 0`, no AWS calls.

## Reference Examples — Style Only, Never Copy Verbatim

### Example 1 — Category A: single query, multi-field check

`Check_ID: s3_account_level_public_access_blocks`

```bash
#!/bin/bash

if [ -z "$ACCOUNT_ID" ] || [ -z "$REGION" ]; then
    echo "Usage: $0 <AccountID> <Region>"
    exit 1
fi

echo "Verifying S3 Block Public Access configuration for account: $ACCOUNT_ID"

RESULT=$(aws s3control get-public-access-block \
    --account-id "$ACCOUNT_ID" \
    --region "$REGION" \
    --query 'PublicAccessBlockConfiguration.[BlockPublicAcls,IgnorePublicAcls,BlockPublicPolicy,RestrictPublicBuckets]' \
    --output text)

if echo "$RESULT" | grep -q "False"; then
    echo "✗ One or more Block Public Access settings are disabled: $RESULT"
    echo "Verification failed: remediation not confirmed"
    exit 1
else
    echo "✓ All Block Public Access settings are enabled"
    echo "Verification successful: remediation confirmed"
    exit 0
fi
```

### Example 2 — Category B: existence-only, limitation stated explicitly

`Check_ID: ec2_instance_profile_attached`

```bash
#!/bin/bash

if [ -z "$INSTANCE_ID" ]; then
    echo "Required parameter INSTANCE_ID is missing."
    exit 1
fi

echo "Verifying IAM instance profile association for instance: $INSTANCE_ID"

RESULT=$(aws ec2 describe-iam-instance-profile-associations \
    --filters "Name=instance-id,Values=$INSTANCE_ID" \
    --query 'IamInstanceProfileAssociations[?State==`associated`]' \
    --output text)

if [ -z "$RESULT" ]; then
    echo "✗ No IAM instance profile is associated with $INSTANCE_ID"
    echo "Verification failed: remediation not confirmed"
    exit 1
else
    echo "✓ An IAM instance profile is associated with $INSTANCE_ID"
    echo "Verification successful: an instance profile is attached. This only confirms existence — confirm manually that it's the correct, least-privilege profile."
    exit 0
fi
```

### Example 3 — Category A: needs an expected-value parameter the remediation didn't require for dry-run

`Check_ID: cloudtrail_logs_s3_bucket_access_logging_enabled`

```bash
#!/bin/bash

if [ -z "$SOURCE_BUCKET" ] || [ -z "$TARGET_BUCKET" ]; then
    echo "Usage: $0 <SourceBucketName> <TargetBucketName>"
    exit 1
fi

echo "Verifying access logging configuration for bucket: $SOURCE_BUCKET"

RESULT=$(aws s3api get-bucket-logging \
    --bucket "$SOURCE_BUCKET" \
    --query 'LoggingEnabled.TargetBucket' \
    --output text)

if [ "$RESULT" = "$TARGET_BUCKET" ]; then
    echo "✓ Access logging is enabled, targeting $TARGET_BUCKET"
    echo "Verification successful: remediation confirmed"
    exit 0
else
    echo "✗ Access logging target is '$RESULT', expected '$TARGET_BUCKET'"
    echo "Verification failed: remediation not confirmed"
    exit 1
fi
```

### Example 4 — Category A: deterministic end state despite a manual, judgment-based remediation

`Check_ID: iam_rotate_access_key_90_days`

```bash
#!/bin/bash

if [ -z "$IAM_USER" ]; then
    echo "Required parameter IAM_USER is missing."
    exit 1
fi

echo "Verifying access key age for user: $IAM_USER"

CUTOFF_DATE=$(date -u -d "90 days ago" +%Y-%m-%d)

OLD_KEYS=$(aws iam list-access-keys \
    --user-name "$IAM_USER" \
    --query "AccessKeyMetadata[?CreateDate<='${CUTOFF_DATE}'].AccessKeyId" \
    --output text)

if [ -z "$OLD_KEYS" ]; then
    echo "✓ No access keys older than 90 days"
    echo "Verification successful: remediation confirmed"
    exit 0
else
    echo "✗ Access key(s) older than 90 days still present: $OLD_KEYS"
    echo "Verification failed: remediation not confirmed"
    exit 1
fi
```

The remediation for this Check_ID is manual (Category C in the remediation skill, Category C in dry-run) — but "no key over 90 days old" is a hard fact regardless of who rotated it, so validation is still Category A here. Don't assume a manual remediation forces a Category B or C validation.

### Example 5 — Category A: multiple components, boolean flags, aggregate check

`Check_ID: cloudwatch_log_metric_filter_disable_or_scheduled_deletion_of_kms_cmk`

```bash
#!/bin/bash

if [ -z "$LOG_GROUP_NAME" ] || [ -z "$REGION" ]; then
    echo "Usage: $0 <LogGroupName> <Region>"
    exit 1
fi

echo "Verifying metric filter for log group: $LOG_GROUP_NAME"

if aws logs describe-metric-filters \
    --region "$REGION" \
    --log-group-name "$LOG_GROUP_NAME" \
    --filter-name-prefix "KMSDisableOrDeleteCMK" | grep -q "KMSDisableOrDeleteCMK"; then
    echo "✓ Metric filter 'KMSDisableOrDeleteCMK' exists"
    FILTER_EXISTS=true
else
    echo "✗ Metric filter 'KMSDisableOrDeleteCMK' not found"
    FILTER_EXISTS=false
fi

echo "Verifying CloudWatch alarm..."

if aws cloudwatch describe-alarms \
    --region "$REGION" \
    --alarm-names "KMSDisableOrDeleteCMKAlarm" | grep -q "KMSDisableOrDeleteCMKAlarm"; then
    echo "✓ Alarm 'KMSDisableOrDeleteCMKAlarm' exists"
    ALARM_EXISTS=true
else
    echo "✗ Alarm 'KMSDisableOrDeleteCMKAlarm' not found"
    ALARM_EXISTS=false
fi

if [ "$FILTER_EXISTS" = true ] && [ "$ALARM_EXISTS" = true ]; then
    echo "Verification successful: remediation confirmed"
    exit 0
else
    echo "Verification failed: remediation not confirmed"
    exit 1
fi
```

This is the canonical multi-component shape: one boolean flag per independently-checkable piece, `&&` them together, exit on the aggregate. Don't reach for an array or a loop even if there are three or four components — the flat flag pattern scales fine at this size.

### Example 6 — Category A: regex match against a set of values, not an exact string

`Check_ID: ec2_securitygroup_allow_ingress_from_internet_to_all_ports`

```bash
#!/bin/bash

if [ -z "$SG_ID" ] || [ -z "$REGION" ]; then
    echo "Required parameters not provided."
    echo "SG_ID=$SG_ID"
    echo "REGION=$REGION"
    exit 1
fi

echo "Verifying unrestricted inbound rules are removed from Security Group: $SG_ID"

RESULT=$(aws ec2 describe-security-groups \
    --group-ids "$SG_ID" \
    --region "$REGION" \
    --query "SecurityGroups[0].IpPermissions[?IpProtocol=='-1']" \
    --output text)

if echo "$RESULT" | grep -qE "0\.0\.0\.0/0|::/0"; then
    echo "✗ Unrestricted inbound access still present on $SG_ID"
    echo "Verification failed: remediation not confirmed"
    exit 1
else
    echo "✓ No unrestricted (0.0.0.0/0 or ::/0) all-ports inbound rules remain"
    echo "Verification successful: remediation confirmed"
    exit 0
fi
```

### Example 7 — Category C: nothing to verify

`Check_ID: iam_policy_attached_only_to_group_or_roles`

```bash
#!/bin/bash

echo "Verification Not Applicable"
echo "This finding's remediation is a manual policy review with no deterministic end-state to verify automatically."
exit 0
```

## Quality Checklist

- Correctly classified into Category A, B, or C — based on whether the _end state_ is checkable, not on how the remediation itself was classified elsewhere
- Every `exit 0` follows an actual comparison against live AWS state
- Category A: exact comparison against an expected value, no ambiguity in the verdict
- Category B: explicitly states it only confirmed existence, not correctness
- Category C: no AWS calls, short output, `exit 0`
- Multi-component checks use flat boolean flags plus one aggregate `if`, never a loop or array
- Parameters include any expected-value inputs validation needs beyond what the remediation or dry-run required
- No resource self-discovery — every resource checked comes from a parameter
- `✓`/`✗` per-component lines plus one final pass/fail line, matching the reference examples' tone
