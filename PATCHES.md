# Narmi fork patches

This is a fork of
[`github-aws-runners/terraform-aws-github-runner`](https://github.com/github-aws-runners/terraform-aws-github-runner).

The `narmi` branch = an upstream release tag + the minimal set of local patches
below. We tag our own releases as `<upstream-tag>-narmi.N`
(e.g. `v6.7.3-narmi.1`).

## Workflow

- `origin`   -> `narmi/terraform-aws-github-runner` (this fork)
- `upstream` -> `github-aws-runners/terraform-aws-github-runner`

To adopt a new upstream release, rebase the patches onto the new tag:

```bash
git fetch upstream --tags
git rebase --onto <new-tag> <old-base-tag> narmi
# resolve conflicts per-patch
git tag <new-tag>-narmi.1
```

Deployment repos must pin an immutable `?ref=<tag>-narmi.N`, never a branch.

## Carried patches

### 1. Add `sts:TagSession` to the runner instance trust policy

- **File:** `modules/runners/policies/instance-role-trust-policy.json`
- **Change:** `"Action": "sts:AssumeRole"` -> `["sts:AssumeRole", "sts:TagSession"]`
- **Why:** Runner workloads assume downstream roles (e.g.
  `GitHubActionsBankingDeployer`) using session tags; the runner instance role's
  trust policy must allow `sts:TagSession` in addition to `sts:AssumeRole`.
- **Externalizable?** No. Upstream exposes no variable to inject additional
  trust-policy statements, so this must be a source patch.
- **Removal criteria:** Remove if upstream adds a variable to append trust-policy
  statements (e.g. `runner_role_additional_trust_policy_statements`), then set it
  from the deployment repo instead.
- **Upstream PR:** none filed yet.

## Explicitly NOT carried (dropped during migration)

These existed in the previous hard-fork and were intentionally dropped; the
functionality is either handled upstream, externalized in the deployment repo,
or removed as dead/buggy code.

- **ECR / S3-cache / assume-role+Bedrock permissions** — externalized in the
  deployment repo as customer-managed IAM policies attached via
  `runner_iam_role_managed_policy_arns`.
- **InstanceId self-tagging + SSM-condition removal** — dropped. Upstream already
  scopes each runner to its own SSM token parameter (the scale-up lambda tags the
  parameter with `InstanceId` and the policy condition matches it). The previous
  edit removed that condition and weakened isolation; clean upstream restores it.
- **`lambda-publish-sqs-policy.json` quoting** — dropped. It was a latent bug
  (invalid JSON when `job_retry` is enabled); upstream is correct.
- **`modules/ecr`** — dropped. Dead code; never referenced or applied.
- **GuardDuty / Inspector exclusion tags** — not a module change; lives in the
  deployment repo via `runner_ec2_tags`.
