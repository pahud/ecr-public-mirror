# Docker Hub to ECR Public Mirror

This repository contains a GitHub Actions workflow designed to mirror specific Docker images and tags from Docker Hub to an AWS ECR Public registry.

## Workflow Overview

The core logic resides in `.github/workflows/mirror.yml`. This workflow performs the following actions:

1.  **Triggers:**
    *   Runs automatically on a schedule (daily at midnight UTC).
    *   Can be manually triggered via the GitHub Actions UI (`workflow_dispatch`).
2.  **Authentication:**
    *   Uses GitHub OIDC to securely authenticate with AWS without needing long-lived access keys. Assumes an IAM role specified in the `AWS_ROLE_TO_ASSUME` secret.
    *   Logs into AWS ECR Public.
    *   Logs into Docker Hub using `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` secrets.
3.  **Mirroring (Matrix Strategy):**
    *   Utilizes a matrix strategy to handle multiple Docker images efficiently.
    *   For each image defined in the matrix:
        *   Iterates through a specified list of tags.
        *   Pulls the image:tag from Docker Hub.
        *   Re-tags the image for the target ECR Public repository.
        *   Pushes the re-tagged image to ECR Public.

## Currently Mirrored Images

The workflow is configured via the `matrix` in the job definition:

| Docker Hub Image          | Tags Mirrored                     | Target ECR Public Repository                     |
| :------------------------ | :-------------------------------- | :----------------------------------------------- |
| `n8nio/n8n`               | `stable`, `beta`, `next`, `latest` | `public.ecr.aws/pahudnet/n8nio/n8n`              |
| `cloudflare/cloudflared`  | `latest`, `latest-arm64`, `latest-amd64` | `public.ecr.aws/pahudnet/cloudflare/cloudflared` |

*(Note: The target ECR Public registry alias `pahudnet` is configured via the `ECR_PUBLIC_REGISTRY` environment variable in the workflow.)*

## Prerequisites

For the workflow to run successfully, the following prerequisites must be met:

1.  **AWS IAM Role:** An IAM role must exist in your AWS account that the GitHub Actions workflow can assume via OIDC. This role needs permissions to log in to ECR Public (`ecr-public:GetAuthorizationToken`, `sts:GetServiceBearerToken`) and push images (`ecr-public:PutRepositoryPolicy`, `ecr-public:DescribeRepositories`, `ecr-public:CreateRepository`, `ecr-public:DescribeImages`, `ecr-public:InitiateLayerUpload`, `ecr-public:UploadLayerPart`, `ecr-public:CompleteLayerUpload`, `ecr-public:BatchCheckLayerAvailability`, `ecr-public:PutImage`).
2.  **GitHub Secrets:** The following secrets must be configured in the GitHub repository settings:
    *   `AWS_ROLE_TO_ASSUME`: The ARN of the IAM role mentioned above.
    *   `DOCKERHUB_USERNAME`: Your Docker Hub username.
    *   `DOCKERHUB_TOKEN`: A Docker Hub Personal Access Token (PAT) with read permissions.
3.  **ECR Public Repositories:** The target repositories **must exist** in your ECR Public gallery *before* the workflow runs. The workflow **does not** create them automatically. Based on the current configuration, you need:
    *   `public.ecr.aws/pahudnet/n8nio/n8n`
    *   `public.ecr.aws/pahudnet/cloudflare/cloudflared`

## Adding More Images

To mirror additional images or change the tags for existing ones:

1.  Edit the `.github/workflows/mirror.yml` file.
2.  Locate the `strategy.matrix.include` section within the `mirror-images` job.
3.  Add a new entry or modify an existing one, specifying the `image_name` (Docker Hub image) and `tags_to_mirror` (space-separated list of tags).
4.  Ensure the corresponding target repository exists in your ECR Public gallery (e.g., `public.ecr.aws/pahudnet/<image_name>`).
5.  Commit and push the changes.
