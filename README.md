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

### Prerequisites

**Required in your workflow:**
1. Checkout your repository
2. Configure AWS credentials

### Basic Deploy from Branch

```yaml
name: Deploy to S3
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-24.04
    permissions:
      id-token: write
      contents: write
      deployments: write
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
      
      - uses: aws-actions/configure-aws-credentials@61815dcd50bd041e203e49132bacad1fd04d2708
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1
      
      - name: Deploy to S3
        uses: LogixDevCo/s3-deploy-action@v1.0.0
        with:
          deploy-type: 'from-branch'
          branch: 'main'
          environment: 'staging'
          node-version: '20'
          target-url: 'https://staging.example.com'
          deployment-prefix: 'frontend'
```

### Deploy from Pull Request

```yaml
name: Deploy PR to S3
on:
  workflow_dispatch:
    inputs:
      pull-request-nb:
        description: 'PR number'
        required: true

jobs:
  deploy:
    runs-on: ubuntu-24.04
    permissions:
      id-token: write
      contents: read
      pull-requests: read
      deployments: write
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
        with:
          ref: main
          fetch-depth: 0
      
      - uses: aws-actions/configure-aws-credentials@61815dcd50bd041e203e49132bacad1fd04d2708
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1
      
      - name: Deploy to S3
        uses: LogixDevCo/s3-deploy-action@v1.0.0
        with:
          deploy-type: 'from-pr'
          pull-request-nb: ${{ github.event.inputs.pull-request-nb }}
          environment: 'staging'
          node-version: '20'
          target-url: 'https://staging.example.com'
          deployment-prefix: 'frontend'
```

### Production with Approval and Cloudflare

```yaml
name: Deploy to Production
on:
  push:
    tags:
      - 'v*'

jobs:
  deploy:
    runs-on: ubuntu-24.04
    permissions:
      id-token: write
      contents: write
      deployments: write
      issues: write
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683
      
      - uses: aws-actions/configure-aws-credentials@61815dcd50bd041e203e49132bacad1fd04d2708
        with:
          role-to-assume: ${{ secrets.AWS_PROD_ROLE_ARN }}
          aws-region: us-east-1
      
      - name: Deploy to S3
        uses: LogixDevCo/s3-deploy-action@v1.0.0
        with:
          deploy-type: 'from-tag'
          commit-tag: ${{ github.ref_name }}
          environment: 'production'
          node-version: '20'
          target-url: 'https://example.com'
          deployment-prefix: 'frontend'
          approvers: 'john-doe,jane-smith'
          cloudflare-zone-id: ${{ secrets.CLOUDFLARE_ZONE_ID }}
          cloudflare-token: ${{ secrets.CLOUDFLARE_PURGE_API_TOKEN }}
          sentry-project: 'my-app'
          slack-webhook: ${{ secrets.SLACK_WEBHOOK_URL }}
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `deploy-type` | Deployment type (from-branch, from-pr, from-tag) | Yes | - |
| `environment` | Target environment | Yes | - |
| `node-version` | Node.js version | Yes | - |
| `target-url` | Target URL | Yes | - |
| `deployment-prefix` | Deployment prefix identifier | Yes | - |
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

And environment config files:

```
deploy.dev.config
deploy.staging.config
deploy.production.config
```

Each config file should contain:
```bash
full_domain_name=staging.example.com
bucket=staging-example-com
```

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
