# cloud-devops-agent-skills

Skills for AI coding agents that generate **remediation, dry-run, validation, and rollback Bash scripts** for AWS security findings, using the AWS CLI.

Each skill is a `SKILL.md` file that tells an agent (Claude or another) how to write one script for one finding, in a deliberately minimal, reviewable style. The four skills are designed to be used together, so the scripts they produce share parameter names, exit-code conventions, and safety rules.

> **Scope:** the skills in this repository are AWS-only. See [Cloud provider coverage](#cloud-provider-coverage) for what that means for Azure and GCP.

---

## Contents

- [Why this exists](#why-this-exists)
- [The skills](#the-skills)
- [How the skills fit together](#how-the-skills-fit-together)
- [Which skill do I need?](#which-skill-do-i-need)
- [Quick start](#quick-start)
- [Usage examples](#usage-examples)
- [Cloud provider coverage](#cloud-provider-coverage)
- [Production safety](#production-safety)
- [Repository structure](#repository-structure)
- [Contributing](#contributing)
- [License](#license)
- [Disclaimer](#disclaimer)

---

## Why this exists

Asking an AI agent to "fix this AWS finding" tends to produce a script that works on the happy path and hides its risks: no preview, no verification, no way back, and sometimes an API call that quietly overwrites a policy someone else depends on.

These skills encode a stricter approach, modeled on hand-written scripts from a DevOps team:

- Every finding gets **four separate scripts** (remediate, preview, verify, undo), not one script that does everything.
- The agent must be **honest about what can't be automated or reversed**. Manual-only, unsupported, and not-applicable outcomes are first-class results, not failures to paper over.
- Scripts stay **small and flat**: direct AWS CLI calls, no `jq`, no loops, no functions, no dependency checks.
- Destructive patterns are gated. For example, APIs that replace a whole policy document check for an existing policy first and fall back to manual remediation.

The skills produce scripts. They do not execute anything, and the generated scripts still need human review before they touch an AWS account.

---

## The skills

| Skill | What it generates | Use it when |
|---|---|---|
| [`remediation-bash-generator`](./remediation-bash-generator) | One Bash script that fixes a given AWS finding (`Check_ID`), or a manual-remediation notice when automation isn't safe | A finding needs an executable fix |
| [`dry-run-bash-generator`](./dry-run-bash-generator) | One read-only Bash script that shows whether the paired remediation would succeed, without applying it | You want to preview a remediation before running it |
| [`validation-bash-generator`](./validation-bash-generator) | One Bash script that checks whether a remediation actually took effect and returns a real pass/fail exit code | You need to confirm a fix, or gate downstream automation on the result |
| [`rollback-bash-generator`](./rollback-bash-generator) | One Bash script that reverses a remediation, or states plainly that it can't or needn't be reversed | A remediation has to be undone |

### `remediation-bash-generator`

Generates exactly one script per finding, derived from the finding details and parameters supplied in the request (not from memory of a similarly named check).

- **Automated:** validate parameters, print one message, run the direct `aws` call(s), print one completion message. Uses plain `set -e` only when steps genuinely depend on each other.
- **Full-replacement APIs** (`put-bucket-policy`, `put-bucket-acl`, `put-key-policy`, and similar): checks for an existing configuration first. If one exists, it does not overwrite it. It prints a manual-remediation message and exits `1`.
- **Manual-remediation pattern:** used when the fix needs human judgment, root credentials, MFA, interactive confirmation, or has no AWS API. Prints a banner, the reasons, and manual steps, then exits `1`.
- **Rules:** no resource discovery when the identifier is already a parameter, no provisioning of auxiliary resources such as SNS topics, and JSON payloads go through a heredoc to `/tmp` rather than inline string concatenation.

### `dry-run-bash-generator`

Generates a script that reports what a remediation would do without changing anything. It classifies each finding:

| Category | Meaning | Script behavior |
|---|---|---|
| **A: Native dry-run** | The mutating API accepts `--dry-run` (common for EC2 actions) | The remediation's own call with `--dry-run` appended |
| **B: Read-only equivalent** | No native flag, but a `get-*`/`describe-*`/`list-*` call shows the state that would change | Prints current state and a one-line statement of what remediation would do |
| **C: Informational only** | The paired remediation is manual-only | A status report that says explicitly it is not a functional dry run |

Every message is prefixed `DRY RUN:`. The script does not compute compliance verdicts; it reports raw state and leaves interpretation to the reader. It exits `0` even when the check finds a problem, and exits `1` only for a missing parameter.

### `validation-bash-generator`

The one skill where verdict logic is the point. It confirms a remediation took effect and returns an exit code that downstream automation can branch on.

| Category | Meaning |
|---|---|
| **A: Deterministic** | The end state is a checkable fact (a flag, a policy attachment, a rule's absence, an age threshold). Compare live state to the expected value. |
| **B: Existence-only** | Human judgment was involved (for example, which IAM role to attach). The script confirms something exists and says so explicitly, without claiming the choice was correct. |
| **C: Not applicable** | The remediation made no automated change. No AWS calls, exits `0`. |

Per-component `✓`/`✗` lines, then a final "Verification successful" or "Verification failed". No `set -e`, no `jq`, no loops. Validation may need extra parameters, such as an expected value to compare against.

### `rollback-bash-generator`

Treats "rollback" as a classification problem first and a Bash problem second. A rollback is not automatically the remediation run backwards.

| Category | When | Output |
|---|---|---|
| **A: Automated reverse** | The pre-remediation state is fully known (a named policy, a fixed rule, a specific flag) | A direct reverse script, structured like a remediation |
| **B: Not supported** | The remediation was manual, the change is irreversible, or it used a full-replacement API | Banner, reason, manual reference if a real command exists, exit `1` |
| **C: Not available / not applicable** | Prior state was overwritten without being recorded, or nothing was changed automatically | One or two lines, exit `0` |

Rollbacks never guess at a prior state. A rollback may deliberately restore a *less secure* state (reopening a security group, for instance) when that state was known.

---

## How the skills fit together

The four skills share parameter names for the same finding, so one set of injected variables (for example `SG_ID` and `REGION`) can drive all four scripts. Validation may add expected-value parameters.

```
              Finding: Check_ID + parameters
                          │
        ┌─────────────────┼──────────────────┐
        ▼                 ▼                  ▼
   dry-run script   remediation script   validation script
   (read-only;      (applies the fix,    (confirms the fix;
    preview)         or says "manual")    real exit code)
        │                 │                  │
        ▼                 ▼                  ▼
   1. review        2. apply, once       3. verify
      output           reviewed             │
                                            ├── pass ── done
                                            └── fail ── investigate
                                                   │
                                                   ▼
                                           rollback script
                                        (only if a safe reverse exists)
```

Suggested order: **dry-run → review → remediation → validation → rollback if needed.** The repository does not include an orchestrator. Sequencing, approvals, and execution are up to you or your platform.

Exit-code conventions differ by script type, and they are worth knowing before you wire scripts into automation:

| Script | `exit 0` | `exit 1` |
|---|---|---|
| Remediation | Success or already compliant | Missing parameters, or manual/unsupported remediation |
| Dry-run | Check completed (even if it found a problem) | Missing parameter only |
| Validation | Verification passed, or not applicable | Missing parameter, or verification failed |
| Rollback | Rollback completed, or not applicable | Missing parameters, or rollback not supported |

A dry-run script for a Category A finding is the exception to watch. AWS returns a deliberate `DryRunOperation` response, which is a non-zero exit even when you are authorized. The skill leaves surfacing that to your downstream handling.

---

## Which skill do I need?

| I want to… | Use |
|---|---|
| Fix an AWS finding with a script | `remediation-bash-generator` |
| See what a fix would do before running it | `dry-run-bash-generator` |
| Prove a fix worked, or gate a pipeline on it | `validation-bash-generator` |
| Undo a fix, or find out whether it can be undone | `rollback-bash-generator` |
| Handle a finding end to end | All four, in the order above |

---

## Quick start

1. **Install the skills.**
   - **Claude Code:** copy the skill folders into `~/.claude/skills/` (all projects) or `.claude/skills/` in a project.
```bash
     git clone https://github.com/Geetanshu405/cloud-devops-agent-skills.git
     cp -r cloud-devops-agent-skills/*-bash-generator ~/.claude/skills/
```
   - **Claude.ai:** upload the skill folders through the Skills settings (each is a folder with a `SKILL.md`, zipped).
   - **Other agents:** the skills are plain Markdown. Provide the relevant `SKILL.md` as instructions or context. Behavior outside Claude has not been verified.

2. **Give the agent the finding.** Include the `Check_ID`, what the finding says, and the parameters that will be available as environment variables. The skills instruct the agent to build only from what you supply.

3. **Review the output.** Each skill returns the script only, with no explanation. Read it before running it (see [Production safety](#production-safety)).

4. **Run it with the parameters set as environment variables.** The generated scripts read variables such as `$ACCOUNT_ID`, `$REGION`, or `$BUCKET_NAME` directly and contain no placeholders to edit.
```bash
   export ACCOUNT_ID=123456789012 REGION=us-east-1   # example values
   bash ./dry-run.sh
```

The scripts assume Bash, the AWS CLI, and valid credentials already exist. The skills never generate dependency checks or install steps.

---

## Usage examples

### Prompting

Example prompts (adapt the wording to your agent):

```
Using the remediation-bash-generator skill, generate a remediation script for
Check_ID s3_bucket_secure_transport_policy. Available parameter: BUCKET_NAME.
```

```
Using the dry-run-bash-generator skill, generate the dry-run script paired with
the remediation above. Use the same parameter names.
```

### One finding through all four skills

The skills' reference examples use `s3_account_level_public_access_blocks`, with parameters `ACCOUNT_ID` and `REGION`:

| Skill | Approach |
|---|---|
| Remediation | `aws s3control put-public-access-block` with all four settings set to `true` |
| Dry-run (Category B) | `aws s3control get-public-access-block`, prints current config, states what remediation would set |
| Validation (Category A) | Reads the four settings; exits `1` if any is `False`, `0` if all are enabled |
| Rollback (Category A) | `aws s3control delete-public-access-block` |

Read rollbacks critically: this one removes the configuration entirely. It does not restore a recorded previous value.

### How the skills classify the same finding differently

Classification is per skill, not inherited. A manual remediation can still have a fully deterministic validation.

| Check_ID (from the reference examples) | Remediation | Dry-run | Validation | Rollback |
|---|---|---|---|---|
| `ec2_securitygroup_allow_ingress_from_internet_to_all_ports` | Automated (`revoke-security-group-ingress`, IPv4 and IPv6) | A: native `--dry-run` | A: no `0.0.0.0/0` or `::/0` all-ports rule remains | A: re-authorizes the known rules |
| `s3_bucket_secure_transport_policy` | Full-replacement API: checks for an existing policy first, falls back to manual | n/a in the examples | n/a in the examples | B: not supported; points to the `DenyInsecureTransport` `Sid` to remove by hand |
| `ec2_instance_profile_attached` | Manual (needs judgment on which profile) | C: informational | B: confirms a profile is attached, not that it's the right one | B: not supported |
| `iam_rotate_access_key_90_days` | Manual | n/a in the examples | A: no access key older than 90 days | B: deleted keys can't be restored |
| `iam_policy_attached_only_to_group_or_roles` | Manual policy review | C: lists attached and inline policies | C: not applicable | C: not applicable |

These IDs are style references inside the skills. Each skill is meant to generate scripts for the finding you give it, not to look up a supported-check list.

---

## Cloud provider coverage

| Provider | Status in this repository |
|---|---|
| **AWS** | The focus. Every skill, rule, and example uses the AWS CLI and AWS services (S3, IAM, EC2, CloudWatch, CloudTrail). |
| **Azure** | No skills, rules, or examples. |
| **GCP** | No skills, rules, or examples. |

The overall approach is provider-neutral in principle: separate preview, apply, verify, and undo scripts, honest classification of what can't be automated or reversed, and predictable exit codes. But the rules inside each skill are AWS-specific. Examples include `--dry-run` support on EC2 APIs and the list of full-replacement `put-*` APIs. Adapting to Azure or GCP would mean writing new skills with provider-specific rules, not reusing these as they are.

---

## Production safety

AI-generated infrastructure changes deserve the same scrutiny as a change from a new engineer. The skills reduce common failure modes, but they don't remove the need for review.

**Before running anything**

- **Read every generated command.** Confirm the service, action, target resource, and region are what you intend.
- **Run in a non-production account first.** Use a sandbox that mirrors the target configuration.
- **Run the dry-run script and read its output.** Category B and C dry runs show current state or a status report. They don't prove the change will succeed, and Category C explicitly performs no validation.
- **Confirm the parameters.** The scripts trust the injected variables. A wrong `SG_ID` or `BUCKET_NAME` will be acted on as given.
- **Use least-privilege credentials** scoped to the change. Never put credentials, keys, or account-specific secrets into prompts, scripts, or commits.

**Understand what the scripts don't do**

- **Flag-setting and additive remediations are called directly.** They don't pre-check whether the resource is already compliant. The skill relies on AWS's own idempotency for repeat runs.
- **Automated rollbacks can restore an insecure state** (reopening a security group, re-attaching a broad policy). That is by design when the prior state was known. Confirm you want it.
- **Manual and "not supported" outputs exit `1`.** That is an honest signal that a human must act, not a bug to suppress.
- **Validation is only as good as its check.** Category B validation confirms existence, not correctness, and says so.
- **Temporary files:** remediation scripts that write JSON use fixed `/tmp` paths. Review this on shared hosts.

**After running**

- Run the validation script and treat a non-zero exit as a failed change until you've investigated.
- Keep rollback in mind before you apply a change, not after. If the rollback skill says the change isn't reversible, you should know that first.

To report a problem in a skill that could lead an agent to generate unsafe output, see [SECURITY.md](./SECURITY.md).

---

## Repository structure

```
cloud-devops-agent-skills/
├── README.md
├── LICENSE
├── CONTRIBUTING.md
├── SECURITY.md
├── .gitignore
├── remediation-bash-generator/
│   └── SKILL.md
├── dry-run-bash-generator/
│   └── SKILL.md
├── validation-bash-generator/
│   └── SKILL.md
└── rollback-bash-generator/
    └── SKILL.md
```

---

## Contributing

Contributions are welcome, especially fixes to inaccuracies and gaps in the existing skills. See [CONTRIBUTING.md](./CONTRIBUTING.md) for the style rules and pull-request checklist. To report a safety or security problem in a skill, see [SECURITY.md](./SECURITY.md).

---

## License

Released under the [MIT License](./LICENSE).

---

## Disclaimer

These skills assist with cloud automation. They do not replace engineering review, cloud-provider documentation, or your organization's change-management processes. Generated scripts can modify or remove cloud resources. You are responsible for reviewing, testing, and approving anything you run.