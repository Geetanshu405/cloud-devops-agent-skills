# Contributing

Thanks for helping improve these skills. The most useful contributions are fixes
to inaccuracies, clearer rules, and gaps where a skill could lead an agent to
generate something unsafe.

## Ground rules

- **Keep the style rules.** Each skill has a "Never Generate" list and a quality
  checklist. Changes should be consistent with them, or explain clearly why a rule
  should change.
- **Reference examples teach structure and style, not answers.** Don't add
  examples that invite an agent to copy a script because a `Check_ID` matches.
- **Don't invent AWS commands.** Every CLI call in an example must be a real
  command with real flags. If you can't verify it against AWS documentation or a
  real run, don't add it.
- **No real identifiers.** No account IDs, ARNs of your own resources, hostnames,
  keys, or company details in examples, issues, or pull requests.
- **New providers get their own skills.** Don't add Azure or GCP branches to the
  AWS skills. Add clearly named, separate skills and update the README's
  provider-coverage table to say what they cover.
- **Stay consistent across the four skills.** They share parameter-name
  conventions and exit-code meanings. A change to one may require a matching
  change in the others.

## Proposing a change

1. For anything larger than a small fix, open an issue first to discuss it.
2. Fork the repository and make your change on a branch.
3. Open a pull request that explains what changed, why, and how you checked it.

## Pull request checklist

- [ ] The change follows the affected skill's "Never Generate" list and quality
      checklist, or explains why a rule should change.
- [ ] Every AWS CLI command in new or edited examples is real, and any flags used
      are documented by AWS.
- [ ] Parameter names in examples match across the remediation, dry-run,
      validation, and rollback skills for the same `Check_ID`.
- [ ] Exit codes match the conventions in the README's exit-code table.
- [ ] Examples were checked for behavior. Describe how (for example, which script
      you ran, and in what kind of account). If you did not run it, say so.
- [ ] No secrets, account IDs, internal URLs, or company-specific details.
- [ ] The README is updated if the change affects what a skill does or which
      providers are covered.

## Editing a skill

- Keep the YAML frontmatter (`name`, `description`) valid. The `name` must match
  the folder name.
- Keep the description accurate. It is what an agent uses to decide when to load
  the skill.
- If a rule and an example disagree, fix one so they agree.

## Reporting problems

For unsafe or misleading skill behavior, see [SECURITY.md](./SECURITY.md). For
everything else, open an issue.