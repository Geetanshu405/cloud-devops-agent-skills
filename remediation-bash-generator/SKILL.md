---
name: remediation-bash-generator
description: Generate a single production-ready AWS remediation Bash script for a given Check_ID. Output must match the coding style of the DevOps team's validated remediation scripts — minimal, parameter-driven, direct AWS CLI, no rollback, no framework. Use whenever an AWS cloud security finding requires an executable remediation script.
---

# AWS Remediation Script Generator

## Purpose

Generate exactly one Bash remediation script per finding, in the style of the DevOps team's hand-written, production-approved scripts. Output the script only — no rollback script, no markdown fences, no explanations, no metadata.

**The reference examples at the bottom of this file teach structure and style only — never treat a matching Check_ID as a cue to reuse an example's script.** Every script must be derived from the actual finding details and parameters supplied in the current request. If a request's Check_ID happens to match a reference example, the example's variable names, literal values, filter patterns, thresholds, and even its AWS API choice are not guaranteed to be correct for this request — re-derive them from what was actually provided. Copying an example because the name looks familiar produces scripts with the wrong parameter names or the wrong logic silently baked in.

## Execution Environment

The script runs inside the remediation platform, which already provides Bash, AWS CLI, valid credentials, and all required parameters as injected environment variables.

Never generate: dependency checks, `command -v`, package installs, or any code that assumes the environment might be incomplete. Go straight to the remediation logic.

## Before Writing Any Script

Work through these four questions using only the finding description and parameters given in the current request — not memory of a similarly-named example:

1. **What AWS API actually fixes this finding?** Confirm the service and action from the finding's description, not from recalling which API a similar-sounding Check_ID used before.
2. **What parameters were actually provided for this request?** List them. Every variable in the script must be one of these — if a name feels right because an example used it, but it wasn't provided here, that's a bug, not a shortcut. If a parameter looks like a reference to an existing resource (an ARN or ID), that resource is managed outside this script — consume it, don't recreate it.
3. **Is this safe to automate, or does it need the manual-remediation pattern?** (See "When Automation Isn't Safe" below.)
4. **Does the API replace an entire config document?** If so, apply the check-before-overwrite exception.

Only after answering these should the script get written. If the current request's Check_ID matches a reference example below, still answer all four — the example may share a name but not share the exact parameters, filter logic, or thresholds this specific request needs.

## Script Structure

```bash
#!/bin/bash

if [ -z "$PARAM" ]; then
    echo "Usage: $0 <Param>"
    exit 1
fi

echo "<one execution message>"

aws <service> <action> \
    --param "$PARAM"

echo "<one completion message>"
```

- **`set -e` only when the script has multiple sequential, dependent steps** (e.g. a pre-check, then a temp-file write, then an `aws` call that depends on both succeeding). A single-call remediation stays bare `#!/bin/bash` with no strict-mode flags, matching Examples 1–3 and 5. Never use `set -euo pipefail` — plain `set -e` is the observed pattern.
- Validate only the parameters this specific remediation uses. Never validate an unused variable.
- Use platform-injected variables directly (`$ACCOUNT_ID`, `$BUCKET_NAME`, `$SECURITY_GROUP_ID`, etc.). Never emit placeholders (`<bucket-name>`, `<region>`) and never ask the user to edit the script.
- Either validation-message style is fine and should be picked to fit the call: `"Usage: $0 <X> <Y>"` for positional-feeling multi-param scripts, or `"Required parameter X is missing."` for a single named parameter. Don't force one style everywhere — the reference scripts use both.
- One execution message before the AWS CLI call(s), one completion message after. Nothing in between except the call itself.
- `exit 0` for success or already-compliant. `exit 1` for missing parameters or unsupported/manual remediation.
- Comments: omit by default. The one acceptable case is disambiguating two adjacent, near-identical AWS calls (e.g. an IPv4 call and an IPv6 call) — a single short comment per call, as in the reference `ec2_securitygroup_allow_ingress_from_internet_to_all_ports` script.
- **Multi-line JSON payloads (bucket policies, KMS key policies, trust policies) go in a heredoc written to `/tmp`, then referenced with `--policy file:///tmp/<name>.json`.** Never build JSON via inline string concatenation or escaped quotes (`'"$VAR"'`) — that pattern is fragile and prone to silent syntax corruption. See Example 6.
- **A literal value is inlined directly at its call site by default — even if it appears more than once.** Naming a constant (an alarm name, a metric namespace, a filter pattern) adds indirection without much benefit at this script size, and can wrongly imply the value is configurable/injected when it isn't. Duplicating a short literal across two `aws` calls is the DevOps team's own observed practice — don't "improve" it into a variable. The only case that genuinely needs a variable is a value produced by a live API response earlier in the same script, which can't be known until runtime (e.g. an ARN returned by a `create-*` call). See Example 7.

## Never Generate

- `describe-*` / `get-*` / `list-*` calls, or any variable whose only job is to capture that output, when the identifier is already an injected parameter. Resource discovery is only acceptable when a required identifier truly cannot be supplied as a parameter.
- jq or other JSON parsing, shell functions, loops, nested conditionals, multi-step validation.
- A separate "check if already compliant" API call before remediating an **additive or flag-setting** operation (e.g. `put-public-access-block`, `detach-user-policy`, `revoke-security-group-ingress` for one specific rule). These only ever affect the exact thing being remediated, so call them directly and let AWS's own idempotency handle repeat runs.
- Progress/debug narration ("Checking...", "Parsing...", "Comparing...").
- **Provisioning auxiliary infrastructure the finding doesn't itself require** — an SNS topic, a topic policy, an IAM role created just to send a notification. If a parameter name implies a reference to something that already exists (`SNS_TOPIC_ARN`, `KMS_KEY_ARN`, `ROLE_ARN`), consume that reference directly. Never invent a resource name and create a parallel copy of something the platform is meant to supply. If the finding truly needs a brand-new resource with no existing parameter for it, that's a signal to check with the parameter list again before assuming — not a default to fall back on.

If two implementations both safely fix the finding, generate the one with fewer commands and less code.

## Exception: Full-Replacement APIs Must Check First

Some AWS APIs **replace an entire configuration document** rather than toggle a flag or add/remove one specific rule — `put-bucket-policy`, `put-bucket-acl`, `put-key-policy`, `put-role-policy` (when replacing rather than adding a named inline policy), `put-queue-attributes` (Policy attribute). Calling these directly risks silently deleting unrelated statements a customer already depends on.

For these APIs only:

1. Check whether a policy/config already exists (e.g. `aws s3api get-bucket-policy --bucket "$BUCKET_NAME" >/dev/null 2>&1`).
2. If one exists, do **not** overwrite it. Print a manual-remediation message explaining that an existing policy was found and must be merged by hand, then `exit 1`.
3. If none exists, create it directly with the required statement.

This is the only case where a pre-check belongs in an automated remediation script. It is not resource discovery (no identifier is being resolved) — it is a safety gate against a destructive overwrite. See Example 6.

## When Automation Isn't Safe

Generate manual-remediation guidance instead of an automated script when: picking the right resource requires human/business judgment, AWS requires root credentials, MFA, or interactive confirmation, or AWS has no API for the action at all. Never invent a command AWS doesn't actually support. If only part of a finding can be safely automated, automate that part and leave the rest manual.

Manual-remediation scripts follow their own established pattern — see Example 4. This is the one place decorative `====` banners and a numbered manual-steps list are correct; that pattern does not apply to automated scripts.

## Quality Checklist

Before returning the script, confirm:

- `#!/bin/bash`, with `set -e` only if the script has multiple dependent steps
- Only the parameters actually used are validated; no placeholders
- Every identifier comes from an injected variable, never discovered via CLI
- **Every variable name and literal value traces back to this request's actual finding details/parameters — not to a reference example's values, even if the Check_ID matches one**
- No jq, temp variables, functions, or nested logic
- **If a provided parameter is an ARN/ID, the script consumes it — it doesn't provision a new resource under an invented name for something that parameter already points to**
- No single-use _or_ repeated-use literal has been pulled into a variable unless it's an API response captured earlier in the same script
- **If the remediation calls a full-replacement API (put-bucket-policy, put-bucket-acl, put-key-policy, etc.), it checks for an existing config first and falls back to manual remediation rather than overwriting it**
- Multi-line JSON payloads use a heredoc to a temp file, never inline string concatenation
- One execution message + one completion message (or the manual-remediation banner pattern, if applicable)
- Correct exit code: 0 for success/compliant, 1 for missing params or manual-only
- Script length and tone match the reference examples below

---

## Reference Examples — Style Only, Never Copy Verbatim

These show the canonical structure, tone, and length. They are not a lookup table. Even when a request's Check_ID exactly matches one below, rebuild the script from that request's actual parameters, filter patterns, thresholds, and resource names — do not paste the example and swap nothing, and do not assume its variable names are what this request provides.

### Example 1 — Simple AWS CLI remediation

`Check_ID: s3_account_level_public_access_blocks`

```bash
#!/bin/bash

if [ -z "$ACCOUNT_ID" ] || [ -z "$REGION" ]; then
    echo "Usage: $0 <AccountID> <Region>"
    exit 1
fi

echo "Enabling S3 Block Public Access for account: $ACCOUNT_ID"

aws s3control put-public-access-block \
    --account-id "$ACCOUNT_ID" \
    --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true \
    --region "$REGION"

echo "S3 Block Public Access enabled successfully."
```

### Example 2 — IAM remediation, single parameter

`Check_ID: iam_policy_cloudshell_admin_not_attached`

```bash
#!/bin/bash

if [ -z "$IAM_USER_NAME" ]; then
    echo "Required parameter IAM_USER_NAME is missing."
    exit 1
fi

POLICY_ARN="arn:aws:iam::aws:policy/AWSCloudShellFullAccess"

echo "Detaching AWSCloudShellFullAccess policy..."

aws iam detach-user-policy \
    --user-name "$IAM_USER_NAME" \
    --policy-arn "$POLICY_ARN"

echo "Policy detached successfully."
```

### Example 3 — Multi-line CLI payload

`Check_ID: cloudtrail_logs_s3_bucket_access_logging_enabled`

```bash
#!/bin/bash

if [ -z "$SOURCE_BUCKET" ] || [ -z "$TARGET_BUCKET" ]; then
    echo "Usage: $0 <SourceBucketName> <TargetBucketName>"
    exit 1
fi

echo "Enabling S3 access logging..."

aws s3api put-bucket-logging \
--bucket "$SOURCE_BUCKET" \
--bucket-logging-status "{
    \"LoggingEnabled\": {
        \"TargetBucket\": \"$TARGET_BUCKET\",
        \"TargetPrefix\": \"access-logs/\"
    }
}"

echo "S3 access logging enabled for $SOURCE_BUCKET"
```

### Example 4 — Manual remediation (unsupported automation)

`Check_ID: ec2_instance_profile_attached`

```bash
#!/bin/bash

echo "========================================="
echo "AUTO REMEDIATION NOT SUPPORTED"
echo "========================================="

echo "Finding: ec2_instance_profile_attached"
echo ""

echo "AWS cannot automatically determine which IAM Instance Profile"
echo "should be attached to the EC2 instance."

echo ""
echo "Attaching an incorrect IAM role may:"
echo "- Break application functionality"
echo "- Grant excessive permissions"
echo "- Violate the principle of least privilege"

echo ""
echo "Manual Remediation Steps:"
echo "1. Create or identify the correct IAM Role."
echo "2. Create an Instance Profile (if not already created)."
echo "3. Add the IAM Role to the Instance Profile."
echo "4. Associate the Instance Profile with the EC2 instance."

echo ""
echo "Example AWS CLI Command:"
echo "aws ec2 associate-iam-instance-profile \\"
echo "    --instance-id $INSTANCE_ID \\"
echo "    --iam-instance-profile Name=$INSTANCE_PROFILE_NAME"

exit 1
```

### Example 5 — Two near-identical calls with disambiguating comments

`Check_ID: ec2_securitygroup_allow_ingress_from_internet_to_all_ports`

```bash
#!/bin/bash

if [ -z "$SG_ID" ] || [ -z "$REGION" ]; then
    echo "Required parameters not provided."
    echo "SG_ID=$SG_ID"
    echo "REGION=$REGION"
    exit 1
fi

echo "Removing unrestricted inbound access from Security Group: $SG_ID"

# Remove IPv4 rule
aws ec2 revoke-security-group-ingress \
    --group-id "$SG_ID" \
    --region "$REGION" \
    --ip-permissions '[{"IpProtocol":"-1","IpRanges":[{"CidrIp":"0.0.0.0/0"}]}]'

# Remove IPv6 rule
aws ec2 revoke-security-group-ingress \
    --group-id "$SG_ID" \
    --region "$REGION" \
    --ip-permissions '[{"IpProtocol":"-1","Ipv6Ranges":[{"CidrIpv6":"::/0"}]}]'

echo "Remediation completed."
```

### Example 6 — Full-replacement API: check before overwrite, heredoc JSON, `set -e`

`Check_ID: s3_bucket_secure_transport_policy`

```bash
#!/bin/bash
set -e

if [ -z "$BUCKET_NAME" ]; then
    echo "Required parameter BUCKET_NAME is missing."
    exit 1
fi

echo "Checking if bucket policy already exists for $BUCKET_NAME..."

if aws s3api get-bucket-policy --bucket "$BUCKET_NAME" >/dev/null 2>&1; then
    echo "Manual remediation required."
    echo "An existing bucket policy is already attached to bucket '$BUCKET_NAME'."
    echo "Automatic remediation has been skipped to avoid overwriting the existing policy."
    echo "Please manually add a Deny statement for insecure transport (aws:SecureTransport=false) to the existing bucket policy."
    exit 1
fi

cat > /tmp/secure-transport-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::$BUCKET_NAME",
        "arn:aws:s3:::$BUCKET_NAME/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
EOF

aws s3api put-bucket-policy \
    --bucket "$BUCKET_NAME" \
    --policy file:///tmp/secure-transport-policy.json

echo "Secure Transport policy applied successfully."
```

**Why this differs from Examples 1–3 and 5:** `put-bucket-policy` replaces the whole document, so this is the full-replacement exception above — check first, bail to manual remediation if something's already there, only create when the slate is clean. The pre-check plus heredoc plus final call are three dependent steps, which is why `set -e` is warranted here but not in the simpler examples.

### Example 7 — Consume a pre-provisioned resource, inline reused literals

`Check_ID: cloudwatch_log_metric_filter_root_usage`

```bash
#!/bin/bash
set -e

if [ -z "$LOG_GROUP_NAME" ] || [ -z "$SNS_TOPIC_ARN" ] || [ -z "$REGION" ]; then
    echo "Usage: $0 <LogGroupName> <SnsTopicArn> <Region>"
    exit 1
fi

echo "Creating metric filter for root account usage in log group '$LOG_GROUP_NAME'..."

aws logs put-metric-filter \
    --region "$REGION" \
    --log-group-name "$LOG_GROUP_NAME" \
    --filter-name "RootAccountUsage" \
    --filter-pattern '{$.userIdentity.type="Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType!="AwsServiceEvent"}' \
    --metric-transformations \
        metricName=RootAccountUsageEventCount,metricNamespace=SecurityMetrics,metricValue=1

echo "Creating alarm for root account usage..."

aws cloudwatch put-metric-alarm \
    --region "$REGION" \
    --alarm-name "RootAccountUsageAlarm" \
    --metric-name RootAccountUsageEventCount \
    --namespace SecurityMetrics \
    --statistic Sum \
    --period 300 \
    --threshold 1 \
    --comparison-operator GreaterThanOrEqualToThreshold \
    --evaluation-periods 1 \
    --alarm-actions "$SNS_TOPIC_ARN"

echo "Remediation completed."
```

**Why there's no topic-creation step:** the SNS topic is managed outside this remediation — `SNS_TOPIC_ARN` arrives as a parameter pointing at one that already exists. Creating a topic here would provision a second, script-owned resource alongside whatever the platform already manages, which is exactly the auxiliary-infrastructure trap called out above.

**Why `RootAccountUsageEventCount` and `SecurityMetrics` aren't variables** even though each appears in both `aws` calls: at this scale, writing the literal twice is simpler to read than routing it through a variable, and it's the pattern actually used in the validated script. Don't add indirection the reference doesn't have.
