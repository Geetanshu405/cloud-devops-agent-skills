# Security Policy

This repository contains instructions (skills) that guide AI agents to generate
Bash scripts for AWS. It does not contain a running service. The security-relevant
risks are in what the skills could lead an agent to generate, and in anything
sensitive accidentally committed here.

## What to report

- A skill instruction or reference example that could lead an agent to generate a
  script that deletes, overwrites, or exposes cloud resources without the gating
  the skills claim to provide (for example, overwriting an existing policy without
  checking first).
- A reference example that contains an incorrect or non-existent AWS CLI command,
  or a flag that changes behavior in a way the skill doesn't document.
- Real credentials, account IDs, internal hostnames, or other sensitive data
  committed to the repository.

## What is out of scope

- Damage caused by running generated scripts without review, testing, or
  appropriate credentials. The README's production-safety section covers this.
- General AWS misconfiguration questions unrelated to the skills.

## How to report

Please do not open a public issue for a problem that involves exposed sensitive
data. Use GitHub's private vulnerability reporting: on the repository's
**Security** tab, choose **Report a vulnerability**.

For other safety concerns with a skill (for example, a misleading example), a
regular issue is fine. Include the skill name, the section or example, and what
an agent could generate as a result.

## Response

This is a small open-source project maintained on a best-effort basis. Reports
are reviewed as time allows, and there is no guaranteed response time.

## If you find exposed credentials

If you believe a real credential appears anywhere in this repository or its
history, rotate it immediately with your cloud provider. Do not wait for the
report to be handled.