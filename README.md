# S3 Static Site Deploy Action

A GitHub Action that deploys static websites to AWS S3 with CloudFront invalidation, build automation, and deployment tracking.

## Features

- 🚀 Deploy from PRs, branches, or tags
- 📦 Automatic npm build with environment-specific configs
- ☁️ CloudFront cache invalidation
- 🌐 Cloudflare cache purging (optional)
- 🏷️ GitHub Deployments tracking
- ✅ Manual approval for production
- 🔔 Sentry release integration
- 📊 Slack notifications
- 🎯 GitHub release creation for tags

## Usage

### Basic Deploy from Branch

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
      deployments: write
    steps:
      - uses: actions/checkout@v4
      
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1
      
      - uses: LogixDevCo/s3-deploy@main
        with:
          deploy-type: 'from-branch'
          branch: 'main'
          environment: 'staging'
          node-version: '20'
          target-url: 'https://staging.example.com'
          bucket: 'staging-bucket'
          deployment-prefix: 'app'
```

### Deploy from Pull Request

```yaml
name: Deploy PR
on:
  workflow_dispatch:
    inputs:
      pull-request-nb:
        description: 'PR number'
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
      pull-requests: write
      deployments: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1
      
      - uses: LogixDevCo/s3-deploy@main
        with:
          deploy-type: 'from-pr'
          pull-request-nb: ${{ github.event.inputs.pull-request-nb }}
          environment: 'staging'
          node-version: '20'
          target-url: 'https://staging.example.com'
          bucket: 'staging-bucket'
          deployment-prefix: 'app'
```

### Multi-Environment with Variable Mapping

```yaml
name: Deploy to S3
on:
  workflow_dispatch:
    inputs:
      deploy-type:
        description: "Deployment Type"
        required: true
        type: choice
        options: [from-pr, from-branch, from-tag]
      branch:
        description: "Branch to deploy"
        required: false
        type: choice
        options: [main]
      environment:
        description: "Target Environment"
        required: true
        type: choice
        options: [staging, production]
      pull-request-nb:
        description: "PR number"
        required: false
        type: string
      commit-tag:
        description: "Tag to deploy"
        required: false
        type: string

env:
  AWS_REGION: us-east-1
  AWS_ROLE_ARN: ${{ secrets.AWS_ROLE_ARN }}

jobs:
  get-config:
    runs-on: ubuntu-latest
    permissions:
      contents: read
    outputs:
      target_url: ${{ steps.config.outputs.TARGET_URL }}
      bucket: ${{ steps.config.outputs.BUCKET }}
    steps:
      - id: config
        uses: kanga333/variable-mapper@master
        with:
          key: "${{ inputs.environment }}"
          map: |
            {
              "staging": {
                "TARGET_URL": "https://staging.example.com",
                "BUCKET": "staging-bucket"
              },
              "production": {
                "TARGET_URL": "https://example.com",
                "BUCKET": "production-bucket"
              }
            }
          export_to: output

  deploy:
    needs: get-config
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    permissions:
      contents: read
      pull-requests: write
      deployments: write
      id-token: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ env.AWS_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - uses: LogixDevCo/s3-deploy@main
        with:
          deploy-type: ${{ inputs.deploy-type }}
          environment: ${{ inputs.environment }}
          node-version: '20'
          target-url: ${{ needs.get-config.outputs.target_url }}
          bucket: ${{ needs.get-config.outputs.bucket }}
          deployment-prefix: "app"
          branch: ${{ inputs.branch }}
          pull-request-nb: ${{ inputs.pull-request-nb }}
          commit-tag: ${{ inputs.commit-tag }}
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `deploy-type` | Deployment type (from-branch, from-pr, from-tag) | Yes | - |
| `environment` | Target environment | Yes | - |
| `node-version` | Node.js version | Yes | - |
| `target-url` | Target URL | Yes | - |
| `deployment-prefix` | Deployment prefix identifier | Yes | - |
| `bucket` | S3 bucket name | Yes | - |
| `branch` | Branch to deploy (for from-branch) | No | - |
| `pull-request-nb` | PR number (for from-pr) | No | - |
| `commit-tag` | Tag to deploy (for from-tag) | No | - |
| `custom-build-folder` | Build output folder | No | `out` |
| `run-ci` | Run npm ci vs npm install | No | `true` |
| `approvers` | Comma-separated approvers | No | `''` |
| `cloudflare-zone-id` | Cloudflare zone ID | No | `''` |
| `cloudflare-token` | Cloudflare API token | No | `''` |
| `sentry-project` | Sentry project | No | `''` |
| `sentry-org` | Sentry organization | No | `''` |
| `sentry-token` | Sentry auth token | No | `''` |
| `slack-webhook` | Slack webhook URL | No | `''` |

## Environment-Specific Build

The action expects npm scripts for each environment:

```json
{
  "scripts": {
    "build:dev": "next build && next export",
    "build:staging": "next build && next export",
    "build:production": "next build && next export"
  }
}
```

Pass the `full-domain-name` and `bucket` as inputs to the action (typically from environment-specific configuration or variable mapping).

## Workflow Steps

1. **Set Deployment Ref**
2. **Merge PR** (if from-pr)
3. **Manual Approval** (if production)
4. **Setup Node.js**
5. **Install Dependencies**
6. **Start Deployment Tracking**
7. **Build Application** (environment-specific)
8. **Sync to S3**
9. **Invalidate CloudFront Cache**
10. **Purge Cloudflare Cache** (if configured)
11. **Update Deployment Status**
12. **Create Sentry Release** (if configured)
13. **Create GitHub Release** (if from tag)
14. **Send Notifications**

## Required Permissions

```yaml
permissions:
  id-token: write      # For AWS OIDC
  contents: write      # For creating releases
  pull-requests: read  # For PR details
  deployments: write   # For deployment tracking
  issues: write        # For approval issues
```

## Required Secrets

- AWS credentials (via OIDC or access keys)
- `CLOUDFLARE_PURGE_API_TOKEN` (if using Cloudflare)
- `SENTRY_AUTH_TOKEN` (if using Sentry)
- `SLACK_WEBHOOK_URL` (if using Slack)

## S3 Bucket Configuration

Bucket should be configured for static website hosting:

```bash
aws s3 website s3://my-bucket/ \
  --index-document index.html \
  --error-document error.html
```

Bucket policy for public access:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicReadGetObject",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*"
  }]
}
```

## CloudFront Distribution

The action automatically finds and invalidates the CloudFront distribution associated with your S3 bucket.

## Requirements

- Node.js project with build scripts
- S3 bucket configured for static hosting
- CloudFront distribution (optional)
- Environment config files
- `package-lock.json` for caching

## License

MIT

## Author

TahaDekmak
