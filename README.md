# photo-uploader-infra

CloudFormation infrastructure for the Photo Uploader lab: a highly available, secure, containerized fullstack photo-gallery app on Amazon ECS Fargate, in a single AWS Region.

Application code lives in a separate repo: [photo-uploader-app](https://github.com/Eric-Obeng/photo-uploader-app).

## Architecture

- Multi-AZ VPC — public subnets for the ALB, private subnets for ECS and RDS
- Public Application Load Balancer routing to ECS Fargate tasks in private subnets
- Amazon S3 for image storage (private bucket, no public access)
- Amazon CloudFront (Price Class 200) serving images via OAC, restricted by S3 bucket policy
- Amazon RDS PostgreSQL (db.t3 family) for photo description metadata
- Amazon ECR for container images
- Amazon ECS Auto Scaling: min 1 / desired 1 / max 4, scaling on CPU utilization
- Amazon EventBridge rule on ECR image push → triggers CodePipeline
- AWS CodeDeploy blue/green deployment to ECS
- VPC Endpoints for CloudWatch Logs, S3, and ECR
- CI/CD to AWS via GitHub OIDC (no long-lived credentials)
- All resources provisioned via CloudFormation with GitSync

## Repo layout

```text
templates/
  bootstrap.yaml            # Standalone: deployment-artifacts + access-logs S3 buckets
  github-oidc.yaml          # Standalone: GitHub OIDC deploy roles (app + infra repos)
  root.yaml                 # Root stack — declares and wires all nested stacks below
  root-deployment.yaml      # GitSync stack deployment file for root.yaml
  nested/
    network.yaml             # VPC, subnets, route tables, VPC endpoints
    security-groups.yaml     # Least-privilege security groups
    storage.yaml             # S3 buckets (images, pipeline artifacts)
    cdn.yaml                 # CloudFront distribution + OAC
    database.yaml            # RDS PostgreSQL
    ecr.yaml                 # ECR repository
    alb.yaml                 # Application Load Balancer
    ecs.yaml                 # ECS cluster, service, task definition, Auto Scaling
    codedeploy.yaml          # CodeDeploy application/deployment group (blue/green)
    pipeline.yaml             # CodePipeline
    eventbridge.yaml          # EventBridge rule: ECR push (latest tag) -> CodePipeline
.github/workflows/
  deploy-bootstrap.yml       # Deploys bootstrap.yaml + syncs nested/ to S3 on every push
diagrams/
  architecture.drawio        # Network/architecture diagram source
```

## Deployment

Nested stacks create a bootstrapping order problem: `root.yaml`'s child stacks are
fetched from an S3 bucket, but that bucket is itself one of the things this repo
creates. The steps below resolve that with two one-time, unavoidable actions (an
OIDC trust can't authorize the workflow that would otherwise create it; Git sync's
repository link is a console/PR-based action by design) — everything else after
that runs automatically on every push to `main`.

### 1. One-time: deploy the GitHub OIDC roles

Run this once, using your own AWS credentials (CloudShell or a local terminal —
never long-lived keys stored anywhere):

```bash
aws cloudformation deploy \
  --stack-name photo-uploader-github-oidc \
  --template-file templates/github-oidc.yaml \
  --capabilities CAPABILITY_NAMED_IAM
```

Then read the two role ARNs it created:

```bash
aws cloudformation describe-stacks \
  --stack-name photo-uploader-github-oidc \
  --query "Stacks[0].Outputs"
```

### 2. One-time: set repository secrets

In **photo-uploader-infra** (Settings → Secrets and variables → Actions → Secrets):

| Secret | Value |
| --- | --- |
| `INFRA_DEPLOY_ROLE_ARN` | `InfraDeployRoleArn` output from step 1 |
| `AWS_REGION` | the region you're deploying to |

In **photo-uploader-app**, set `APP_DEPLOY_ROLE_ARN` (the `AppDeployRoleArn` output),
`AWS_REGION`, and `ROOT_STACK_NAME` (the stack name you'll give `root.yaml` in step 4).

### 3. Push to `main`

Pushing any change under `templates/bootstrap.yaml` or `templates/nested/**` triggers
`.github/workflows/deploy-bootstrap.yml`, which automatically:

1. Deploys/updates `bootstrap.yaml` (idempotent).
2. Syncs `templates/nested/*` to the deployment-artifacts bucket under a commit-SHA prefix.
3. Rewrites `templates/root-deployment.yaml` with the real bucket names and this commit's
   SHA, and commits it back to `main`.

### 4. One-time: link the repo to CloudFormation Git sync

In the CloudFormation console: **Create stack → Sync from Git**, then:

- Link this repository via a CodeConnections connection (GitHub OAuth, one-time).
- Branch: `main`.
- Template file path: `templates/root.yaml`.
- Stack deployment file: choose **I am providing my own file**, path `templates/root-deployment.yaml`.

Submitting opens a pull request — merging it creates the root stack. From then on,
any push that changes `root.yaml` or `root-deployment.yaml` (including the automatic
commits from step 3) updates the stack automatically — no further manual deploys.

## Status

All templates written; infra automation (steps above) not yet exercised end-to-end.
