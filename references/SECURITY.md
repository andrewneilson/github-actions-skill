# Security Reference

Secrets management, OIDC authentication, and security best practices.

## Secrets

### Creating Secrets

**Repository secrets**: Settings > Secrets and variables > Actions > New repository secret

**Environment secrets**: Settings > Environments > [env] > Add secret

**Organization secrets**: Organization Settings > Secrets and variables > Actions

### Using Secrets

```yaml
steps:
  # As environment variable
  - run: echo "Using secret"
    env:
      API_KEY: ${{ secrets.API_KEY }}

  # As action input
  - uses: some-action@v1
    with:
      token: ${{ secrets.TOKEN }}
```

### Secrets Behavior

- Secrets are redacted from logs automatically
- Cannot be used directly in `if:` conditions
- Empty string if secret doesn't exist
- Not passed to workflows from forked repositories
- Not passed to reusable workflows (must pass explicitly)
- Max size: 48 KB per secret

### Check if Secret Exists

```yaml
jobs:
  build:
    env:
      HAS_SECRET: ${{ secrets.MY_SECRET != '' }}
    steps:
      - if: env.HAS_SECRET == 'true'
        run: echo "Secret is set"
```

### Pass Secrets to Reusable Workflows

```yaml
jobs:
  call:
    uses: ./.github/workflows/reusable.yml
    secrets:
      token: ${{ secrets.TOKEN }}

  # Or inherit all secrets
  call-inherit:
    uses: ./.github/workflows/reusable.yml
    secrets: inherit
```

## GITHUB_TOKEN

Automatically created token for each workflow run.

### Default Permissions

Depends on repository settings. Can be:
- **Permissive**: Read and write for most scopes
- **Restricted**: Read-only for contents and metadata

### Setting Permissions

```yaml
# Workflow level
permissions:
  contents: read
  pull-requests: write
  issues: write

# Job level
jobs:
  build:
    permissions:
      contents: read
```

### Permission Scopes

| Permission | Description |
|------------|-------------|
| `actions` | Manage Actions |
| `attestations` | Artifact attestations |
| `checks` | Check runs/suites |
| `contents` | Repository contents, releases |
| `deployments` | Deployments |
| `discussions` | Discussions |
| `id-token` | OIDC tokens |
| `issues` | Issues |
| `packages` | GitHub Packages |
| `pages` | GitHub Pages |
| `pull-requests` | Pull requests |
| `security-events` | Code scanning |
| `statuses` | Commit statuses |

### Using GITHUB_TOKEN

```yaml
steps:
  - uses: actions/checkout@v5

  - run: |
      gh pr comment ${{ github.event.pull_request.number }} \
        --body "Build complete!"
    env:
      GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  - uses: actions/github-script@v7
    with:
      github-token: ${{ secrets.GITHUB_TOKEN }}
      script: |
        github.rest.issues.createComment({
          owner: context.repo.owner,
          repo: context.repo.repo,
          issue_number: context.issue.number,
          body: 'Hello!'
        })
```

### Token Limitations

- Cannot trigger new workflow runs
- Cannot access other repositories
- Cannot push to protected branches without bypass

## OIDC (OpenID Connect)

Request short-lived tokens from cloud providers without storing credentials.

### How It Works

1. Workflow requests OIDC token from GitHub
2. GitHub issues JWT with workflow claims
3. Cloud provider validates JWT
4. Cloud provider issues short-lived access token

### Required Permission

```yaml
permissions:
  id-token: write
  contents: read
```

### AWS OIDC

**1. Configure AWS Identity Provider**

- Provider URL: `https://token.actions.githubusercontent.com`
- Audience: `sts.amazonaws.com`

**2. Create IAM Role with Trust Policy**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::ACCOUNT_ID:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:OWNER/REPO:*"
        }
      }
    }
  ]
}
```

**3. Use in Workflow**

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/my-role
          aws-region: us-east-1

      - run: aws s3 ls
```

### OIDC Subject Claims

Format: `repo:OWNER/REPO:CONTEXT`

Examples:
- `repo:octo-org/octo-repo:ref:refs/heads/main` - Main branch
- `repo:octo-org/octo-repo:environment:production` - Production environment
- `repo:octo-org/octo-repo:pull_request` - Any PR

### Trust Policy Conditions

```json
{
  "Condition": {
    "StringEquals": {
      "token.actions.githubusercontent.com:sub": "repo:owner/repo:environment:production"
    }
  }
}
```

Or with wildcards:

```json
{
  "Condition": {
    "StringLike": {
      "token.actions.githubusercontent.com:sub": "repo:owner/repo:*"
    }
  }
}
```

### Azure OIDC

```yaml
- uses: azure/login@v1
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

### GCP OIDC

```yaml
- uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: 'projects/PROJECT_NUMBER/locations/global/workloadIdentityPools/POOL_ID/providers/PROVIDER_ID'
    service_account: 'sa@project.iam.gserviceaccount.com'
```

## Script Injection Prevention

### Dangerous Pattern

```yaml
# VULNERABLE - user input directly in script
- run: echo "${{ github.event.issue.title }}"
```

If title is `"; rm -rf / #`, this becomes:
```bash
echo ""; rm -rf / #"
```

### Safe Pattern

```yaml
# SAFE - use environment variable
- run: echo "$TITLE"
  env:
    TITLE: ${{ github.event.issue.title }}
```

### Untrusted Inputs

These contexts can contain attacker-controlled data:

- `github.event.issue.title`
- `github.event.issue.body`
- `github.event.pull_request.title`
- `github.event.pull_request.body`
- `github.event.comment.body`
- `github.event.review.body`
- `github.event.head_commit.message`
- `github.event.commits[*].message`
- `github.head_ref`

### Safe Contexts

These are generally safe:

- `github.repository`
- `github.repository_owner`
- `github.actor`
- `github.sha`
- `github.ref`
- `github.event_name`

## Pull Request Security

### pull_request vs pull_request_target

| Event | Runs On | Secrets | Safe for Forks |
|-------|---------|---------|----------------|
| `pull_request` | Merge commit | No (from forks) | Yes |
| `pull_request_target` | Base branch | Yes | No (be careful) |

### Safe PR Workflow Pattern

```yaml
on: pull_request

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
        # Checks out PR merge commit, safe

      - run: npm test
        # Tests PR code, safe
```

### Dangerous Pattern (Avoid)

```yaml
on: pull_request_target

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      # DANGEROUS - runs PR code with secrets access
      - uses: actions/checkout@v5
        with:
          ref: ${{ github.event.pull_request.head.sha }}

      - run: npm test
```

### Safe pull_request_target Pattern

```yaml
on: pull_request_target

jobs:
  # Job 1: Build/test without secrets (safe)
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
        with:
          ref: ${{ github.event.pull_request.head.sha }}
      - run: npm test

  # Job 2: Deploy with secrets (only trusted code)
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
        # Checks out base branch, safe

      - run: ./deploy.sh
        env:
          TOKEN: ${{ secrets.DEPLOY_TOKEN }}
```

## Permissions Best Practices

### Principle of Least Privilege

```yaml
permissions:
  contents: read    # Only what's needed

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
```

### Disable All Permissions

```yaml
permissions: {}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
```

## Environment Protection

### Configure Environments

1. Settings > Environments > New environment
2. Add protection rules:
   - Required reviewers
   - Wait timer
   - Deployment branches

### Use in Workflow

```yaml
jobs:
  deploy:
    environment:
      name: production
      url: https://example.com
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

### Environment Secrets

Secrets scoped to an environment:
- Only available to jobs targeting that environment
- Require environment approval if configured

## Dependabot Security

### Dependabot Secrets

Workflows triggered by Dependabot:
- Use read-only GITHUB_TOKEN
- Don't have access to regular secrets
- Can use Dependabot secrets

### Handle Dependabot PRs

```yaml
on: pull_request

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - run: npm test

  deploy:
    if: github.actor != 'dependabot[bot]'
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
        env:
          TOKEN: ${{ secrets.DEPLOY_TOKEN }}
```

## Artifact Attestation

Create SLSA provenance for build artifacts.

```yaml
permissions:
  id-token: write
  contents: read
  attestations: write

steps:
  - uses: actions/checkout@v5
  - run: npm run build
  - uses: actions/attest-build-provenance@v1
    with:
      subject-path: 'dist/*.js'
```

## Security Checklist

1. **Secrets**
   - [ ] Use secrets, not hardcoded values
   - [ ] Use OIDC instead of long-lived credentials
   - [ ] Rotate secrets regularly

2. **Permissions**
   - [ ] Set minimal permissions
   - [ ] Use `permissions: {}` and grant explicitly

3. **Dependencies**
   - [ ] Pin action versions to SHA
   - [ ] Enable Dependabot for actions

4. **Code**
   - [ ] Never use untrusted input in `run:` directly
   - [ ] Use environment variables for user input

5. **Pull Requests**
   - [ ] Use `pull_request` not `pull_request_target` when possible
   - [ ] Require approval for fork PRs
   - [ ] Don't run untrusted code with secrets
