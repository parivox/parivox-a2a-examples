# Deploying an agent to AWS

`.github/workflows/deploy-research-specialist-sdk.yml` is a working example of deploying
`research-specialist-sdk` to ECS Fargate with GitHub Actions and OIDC. **It is inert in this
repository:** it only runs when started by hand, and only where `DEPLOY_ENABLED` is set. To
use it, copy the repository and configure the pieces below in your own AWS account and
GitHub repository.

The CloudFormation template (`agents/research-specialist-sdk/infra/template.yaml`) creates
everything the service needs: cluster, task definition, security group, Cloud Map private
DNS, log group and the two task roles. The workflow builds the image, pushes it to ECR,
deploys the stack, and verifies the result.

## 1. AWS: an OIDC deploy role

Add GitHub's OIDC provider to the account (`token.actions.githubusercontent.com`, audience
`sts.amazonaws.com`), then create a role the workflow can assume. Scope its trust policy to
your repository **and** the deploy environment, so no other repository, branch or fork can
assume it:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"},
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
        "token.actions.githubusercontent.com:sub": "repo:<OWNER>/<REPO>:environment:aws-production"
      }
    }
  }]
}
```

The role's permissions need to cover, scoped to resources named `parivox-a2a-*` (the
template's `ServiceName` prefix) wherever the service supports resource-level permissions:

- ECR: create and describe the repository, push images, `ecr:GetAuthorizationToken`
- CloudFormation: create, update and describe the stack and its change sets
- ECS, Cloud Map (`servicediscovery`), CloudWatch Logs, and EC2 security groups
- IAM: create the two task roles and `iam:PassRole` on them. Restrict this to
  `arn:aws:iam::<ACCOUNT_ID>:role/parivox-a2a-*`: a deploy role that can create and pass
  arbitrary roles can grant itself anything.
- `secretsmanager:DescribeSecret` on the two secrets (the task's execution role, not the
  deploy role, reads their values)

## 2. AWS: the secrets the agent reads

Create two Secrets Manager secrets:

| Secret | Value |
|---|---|
| A2A API key | JSON object keyed by header name, e.g. `{"X-API-Key": "<long random string>"}` |
| OpenAI API key (optional) | The bare key. Without it the agent serves canned content. |

## 3. GitHub: the `aws-production` environment

Create an environment named `aws-production` with **required reviewers** and a deployment
branch rule allowing only `main`. Store these in it:

| Name | Kind | Value |
|---|---|---|
| `AWS_ROLE_TO_ASSUME` | secret | ARN of the deploy role |
| `A2A_VPC_ID` | secret | VPC the calling agent runs in |
| `A2A_SUBNET_IDS` | secret | Comma-separated subnet IDs in that VPC |
| `A2A_CLIENT_SECURITY_GROUP_ID` | secret | The only security group allowed to call the agent |
| `A2A_RESEARCH_SDK_API_KEY_SECRET_ARN` | secret | ARN of the A2A API key secret |
| `A2A_RESEARCH_SDK_OPENAI_SECRET_ARN` | secret | ARN of the OpenAI secret, or leave unset |
| `AWS_REGION` | variable | e.g. `us-east-1` |

None of the IDs and ARNs is a credential, but they are stored as secrets on purpose: GitHub
prints each step's environment and command text in the job log, and only secrets are masked
there. On a public repository that log is public. The workflow also masks the account ID,
which appears in every ECR image URI.

Finally, set the repository variable `DEPLOY_ENABLED` to `true`.

## 4. Run it

Actions → `deploy-research-specialist-sdk` → **Run workflow**. After approval the job
deploys the stack and fails if the running task definition doesn't use the image it just
built, or if the service security group admits any CIDR range. The agent's RPC URL is the
stack's `RpcUrl` output, resolvable only inside the VPC.

## Deploying automatically after CI

To deploy every green `main` build, add a `workflow_run` trigger on `ci`. Gate it on all
three of these, not just the branch name:

```yaml
if: >
  github.event.workflow_run.conclusion == 'success' &&
  github.event.workflow_run.event == 'push' &&
  github.event.workflow_run.head_branch == 'main' &&
  github.event.workflow_run.head_repository.full_name == github.repository
```

For a fork pull request, `workflow_run.head_branch` is the fork's own branch name, so a
fork that names its branch `main` passes a branch-only check. The deploy then runs in your
repository with your role. Because `workflow_run` jobs run in the context of your default
branch, an OIDC trust policy that only restricts the ref does not stop this. The
environment-scoped trust policy and required reviewers above do.
