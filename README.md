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

```
templates/
  network.yaml            # VPC, subnets, route tables, VPC endpoints
  security-groups.yaml    # Least-privilege security groups
  storage.yaml             # S3 buckets (images + deployment artifacts)
  cdn.yaml                 # CloudFront distribution + OAC
  database.yaml            # RDS PostgreSQL
  ecr.yaml                 # ECR repository
  ecs.yaml                 # ECS cluster, service, task definition, Auto Scaling
  alb.yaml                 # Application Load Balancer
  codedeploy.yaml           # CodeDeploy application/deployment group (blue/green)
  pipeline.yaml             # CodePipeline
  eventbridge.yaml          # EventBridge rule: ECR push -> CodePipeline
  github-oidc.yaml          # GitHub OIDC provider + deploy role
diagrams/
  architecture.drawio       # Network/architecture diagram source
```

## Status

Scaffolding only — templates are being built out incrementally.
