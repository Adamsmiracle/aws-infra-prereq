# todo-app prerequisites (deploy FIRST)

This repository holds everything that must exist **before** the main
infrastructure repo (`cloudformation_infra`) can deploy — it breaks the
chicken-and-egg race between the pipelines and the infrastructure they need:

| Resource | Why it must exist first |
|---|---|
| S3 templates bucket | The infra repo's GitHub Actions uploads nested-stack child templates here; the parent stack reads them from here. |
| ECR repository | The app repo's CI pushes images here; the ECS service and pipeline reference it by name. |
| Infra GitHub Actions role (OIDC) | The infra repo's upload workflow assumes it. Scoped to branch `todo` only. |
| App GitHub Actions role (OIDC) | The app repo's build workflow assumes it (ECR push + deploy bundle). Scoped to branch `todo` only. |

The template is **flat** (no nested stacks), so CloudFormation GitSync can
deploy it directly from this repo with no template bucket required.

## Deploy order

1. **This repo** — connect it to CloudFormation GitSync (stack name
   `todo-app-prereqs`, deployment file `deployment.yaml`). One manual
   prerequisite remains account-level: the GitHub OIDC provider
   (`token.actions.githubusercontent.com`) must exist in IAM.
2. Copy the stack outputs into repo secrets:
   - `InfraGitHubActionsRoleArn` → infra repo secret `TODO_INFRA_GHA_ROLE_ARN`
   - `AppGitHubActionsRoleArn` → app repo secret `AWS_ROLE_ARN_TODO`
   - `TemplatesBucketName` → infra repo secret `TODO_TEMPLATES_BUCKET`
3. **Infra repo** — push to `todo`; its workflow uploads child templates to the
   bucket, then GitSync deploys the parent stack (network, security groups,
   IAM, database, cache, ALB/ECS, pipeline).
4. **App repo** — push to `todo`; its workflow builds the image, pushes to the
   ECR repo created here, and the pipeline deploys it blue/green.

## Branch scoping

Both OIDC trust policies use
`token.actions.githubusercontent.com:sub = repo:<org>/<repo>:ref:refs/heads/todo`
— a workflow run on any other branch (or a PR ref) cannot assume the roles.
Change `InfraBranch` / `AppBranch` in `deployment.yaml` if the lab branch moves.
