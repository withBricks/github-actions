# Bricks Shared GitHub Actions

Reusable composite actions for AWS infrastructure automation, development environment setup, and CI/CD workflows.

## Usage

Reference these actions in your workflows using:

```yaml
- uses: withBricks/github-actions/configure-aws-access@v1
  with:
    target-account-role: arn:aws:iam::ACCOUNT_ID:role/RoleName
```

## Versioning

- `@v1` - Latest v1.x (recommended for stability, auto-updates with patches)
- `@v1.0.0` - Specific version (pin to exact release)
- `@main` - Latest development (not recommended for production)

## Available Actions

### 1. configure-aws-access

Configures AWS credentials using GitHub OIDC and optionally assumes a cross-account role.

**Inputs:**
- `bricks-ou-role` (optional): Bricks OU GitHub Actions OIDC role ARN
  - Default: `arn:aws:iam::843062504052:role/BricksOUGitHubActionsRole`
- `target-account-role` (optional): Target account cross-account role ARN
- `aws-region` (optional): AWS region (default: `us-east-1`)

**Example:**
```yaml
- name: Configure AWS access
  uses: withBricks/github-actions/configure-aws-access@v1
  with:
    target-account-role: arn:aws:iam::968666601602:role/BricksOUCrossAccountAccessRole
    aws-region: us-east-1
```

### 2. get-db-credentials

Retrieves database credentials from AWS Secrets Manager with automatic password masking.

**Security Features:**
- Passwords are automatically masked before any output
- Never exposes the full secret JSON in logs
- Uses GitHub Actions `::add-mask::` for protection

**Inputs:**
- `secret-id` (required): Secrets Manager secret ID

**Outputs:**
- `username`: Database username
- `password`: Database password (masked in logs)

**Example:**
```yaml
- name: Get database credentials
  id: db_creds
  uses: withBricks/github-actions/get-db-credentials@v1
  with:
    secret-id: postgresAdminRole

- name: Use credentials
  env:
    DB_USER: ${{ steps.db_creds.outputs.username }}
    DB_PASS: ${{ steps.db_creds.outputs.password }}  # Masked in logs
  run: |
    echo "Username: $DB_USER"
    # Password is masked automatically
```

### 3. connect-bastion

Establishes SSM port forwarding to RDS through EC2 bastion instance.

**Architecture:**
```
GitHub Runner → SSM Session → EC2 Bastion → RDS (private subnet)
localhost:5432               i-xxxxx        postgres.equips.service:5432
```

**Inputs:**
- `environment` (required): Environment name (dev or prod)
- `db-host` (required): Database hostname (e.g., `postgres.equips.service`)
- `db-port` (optional): Database port (default: `5432`)
- `local-port` (optional): Local port for forwarding (default: `5432`)

**Outputs:**
- `instance-id`: EC2 bastion instance ID
- `ssm-pid`: SSM session process ID for cleanup

**Example:**
```yaml
- name: Connect to database via bastion
  id: bastion
  uses: withBricks/github-actions/connect-bastion@v1
  with:
    environment: dev
    db-host: postgres.equips.service
    db-port: 5432
    local-port: 5432

- name: Use database connection
  env:
    PGHOST: localhost
    PGPORT: 5432
  run: |
    psql -c "SELECT version();"
```

**Requirements:**
- EC2 bastion instance tagged as `{environment}-github-bastion`
- Instance must be running and have SSM agent online
- Bastion must have network access to target database

### 4. cleanup-bastion

Stops SSM port forwarding session.

**Inputs:**
- `ssm-pid` (required): Process ID of the SSM session to kill

**Example:**
```yaml
- name: Cleanup bastion connection
  if: always()  # Always run, even on failure
  uses: withBricks/github-actions/cleanup-bastion@v1
  with:
    ssm-pid: ${{ steps.bastion.outputs.ssm-pid }}
```

### 5. create-github-deployment

Creates a GitHub deployment and sets it to `in_progress` status. This enables deployment tracking in GitHub's deployment view and environments.

**Inputs:**
- `github-token` (required): GitHub token for API access (typically `${{ github.token }}`)
- `repository` (required): Repository in `owner/repo` format
- `sha` (required): Git SHA to deploy
- `environment` (required): Environment name for the deployment
- `description` (optional): Description for the deployment
- `initial-status-description` (optional): Description for the initial `in_progress` status
  - Default: `"Deployment in progress..."`

**Outputs:**
- `deployment-id`: The ID of the created deployment (use this to update status later)

**Example:**
```yaml
- name: Create deployment
  id: deployment
  uses: withBricks/github-actions/create-github-deployment@v1
  with:
    github-token: ${{ github.token }}
    repository: ${{ github.repository }}
    sha: ${{ github.sha }}
    environment: production
    description: "Deploy v1.2.3 to production"
    initial-status-description: "Applying Tofu configuration..."
```

### 6. update-github-deployment

Updates the status of an existing GitHub deployment. Use this to mark deployments as successful, failed, or other states.

**Inputs:**
- `github-token` (required): GitHub token for API access
- `repository` (required): Repository in `owner/repo` format
- `deployment-id` (required): The deployment ID to update (from create-github-deployment output)
- `state` (required): The state to set
  - Valid values: `success`, `failure`, `error`, `inactive`, `in_progress`, `queued`, `pending`
- `description` (optional): Description for the status update
- `environment-url` (optional): URL for accessing the deployment environment
- `log-url` (optional): URL for accessing the deployment logs

**Example:**
```yaml
- name: Update deployment status - success
  if: success()
  uses: withBricks/github-actions/update-github-deployment@v1
  with:
    github-token: ${{ github.token }}
    repository: ${{ github.repository }}
    deployment-id: ${{ steps.deployment.outputs.deployment-id }}
    state: success
    description: "Successfully deployed to production"
    environment-url: "https://app.example.com"

- name: Update deployment status - failure
  if: failure()
  uses: withBricks/github-actions/update-github-deployment@v1
  with:
    github-token: ${{ github.token }}
    repository: ${{ github.repository }}
    deployment-id: ${{ steps.deployment.outputs.deployment-id }}
    state: failure
    description: "Deployment failed - check logs for details"
```

### 7. publish-to-github-packages

Automatically version, build, and publish npm packages to GitHub Packages with release creation. Handles version incrementing, tagging, and release notes generation.

**Features:**
- Automatic semantic versioning (patch, minor, major)
- Package building before publishing
- GitHub Packages authentication configuration
- Git tag creation and pushing
- Optional GitHub release creation with installation instructions
- Support for multiple package managers (npm, bun, yarn, pnpm)

**Inputs:**
- `package-path` (required): Path to package directory (e.g., `packages/types`)
- `version-bump` (optional): Version bump type - `patch`, `minor`, or `major` (default: `patch`)
- `scope` (required): Package scope (e.g., `@bricks`, `@withBricks`)
- `skip-ci-pattern` (optional): Pattern to add to commit message to skip CI (default: `[skip ci]`)
- `create-release` (optional): Whether to create a GitHub release (default: `true`)
- `tag-prefix` (optional): Prefix for git tags (e.g., `types-v`, `api-v`) (default: `v`)
- `github-token` (required): GitHub token with `contents:write` and `packages:write` permissions
- `node-version` (optional): Node.js version to use (default: `20`)
- `package-manager` (optional): Package manager to use - `npm`, `bun`, `yarn`, or `pnpm` (default: `npm`)

**Outputs:**
- `version`: The new version number that was published
- `tag`: The git tag that was created
- `package-name`: The package name that was published

**Example:**
```yaml
- name: Publish package to GitHub Packages
  id: publish
  uses: withBricks/github-actions/publish-to-github-packages@v1
  with:
    package-path: packages/types
    version-bump: patch
    scope: '@withBricks'
    tag-prefix: types-v
    github-token: ${{ secrets.GITHUB_TOKEN }}
    
- name: Output published version
  run: |
    echo "Published ${{ steps.publish.outputs.package-name }} version ${{ steps.publish.outputs.version }}"
    echo "Tag: ${{ steps.publish.outputs.tag }}"
```

**Requirements:**
- Package must be properly configured in `package.json` with:
  ```json
  {
    "name": "@withBricks/package-name",
    "publishConfig": {
      "access": "restricted",
      "registry": "https://npm.pkg.github.com"
    }
  }
  ```
- Workflow must have required permissions:
  ```yaml
  permissions:
    contents: write
    packages: write
  ```

### 8. sync-to-gitlab

Syncs GitHub repository commits to a GitLab repository with automatic divergence detection and commit author preservation. Handles Copilot-authored commits by using the workflow actor instead.

**Features:**
- Detects if GitLab already has the commit (skips unnecessary syncs)
- Prevents sync if GitLab has diverged from GitHub
- Preserves original commit author information
- Handles Copilot merge commits by using the PR author
- Secure credential handling with automatic cleanup

**Inputs:**
- `gitlab-token` (required): GitLab personal access token or OAuth token
- `gitlab-repo-url` (required): GitLab repository URL (e.g., `https://gitlab.com/owner/repo.git`)
- `branch` (optional): Branch to sync (default: `master`)
- `commit-sha` (required): Commit SHA to sync
- `author-name` (required): Original commit author name
- `author-email` (required): Original commit author email
- `actor-name` (optional): GitHub actor name (used for Copilot commits)
- `actor-email` (optional): GitHub actor email (used for Copilot commits)

**Outputs:**
- `synced`: Whether sync was performed (`true`/`false`)
- `skipped`: Whether sync was skipped because GitLab already had the commit (`true`/`false`)

**Example:**
```yaml
- name: Checkout with full history
  uses: actions/checkout@v4
  with:
    fetch-depth: 0

- name: Sync to GitLab
  uses: withBricks/github-actions/sync-to-gitlab@v1
  with:
    gitlab-token: ${{ secrets.GITLAB_SYNC_TOKEN }}
    gitlab-repo-url: https://gitlab.com/myorg/myrepo.git
    branch: master
    commit-sha: ${{ github.sha }}
    author-name: ${{ github.event.head_commit.author.name }}
    author-email: ${{ github.event.head_commit.author.email }}
    actor-name: ${{ github.actor }}
    actor-email: ${{ github.event.pusher.email || format('{0}@users.noreply.github.com', github.actor) }}
```

**Requirements:**
- Repository must be checked out with full history (`fetch-depth: 0`)
- GitLab token needs write access to the target repository
- Token should be stored as a repository secret

## Package Installation from GitHub Packages

For packages published using the `publish-to-github-packages` action, consumers need to configure authentication:

### For CI/CD (GitHub Actions)
Uses `GITHUB_TOKEN` automatically with `actions/setup-node`:

```yaml
- uses: actions/setup-node@v4
  with:
    registry-url: 'https://npm.pkg.github.com'
    scope: '@withBricks'
    
- run: npm install
  env:
    NODE_AUTH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### For Local Development

Create `~/.npmrc` with your GitHub Personal Access Token:
```
@withBricks:registry=https://npm.pkg.github.com
//npm.pkg.github.com/:_authToken=YOUR_GITHUB_PERSONAL_ACCESS_TOKEN
```

To generate a token:
1. Go to GitHub Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Generate new token with `read:packages` scope
3. Add token to `~/.npmrc`

### For Project Configuration

Create `.npmrc` in project root:
```
@withBricks:registry=https://npm.pkg.github.com
```

Then use environment variable for token:
```bash
echo "//npm.pkg.github.com/:_authToken=${GITHUB_TOKEN}" >> .npmrc
```

## Complete Workflow Examples

### Publish Package to GitHub Packages

See [examples/publish-package-workflow.yml](examples/publish-package-workflow.yml) for a complete example workflow.

### Database Deployment with Bastion

```yaml
name: Database Deployment

on:
  workflow_dispatch:
    inputs:
      environment:
        required: true
        type: choice
        options:
          - dev
          - prod

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # 1. Configure AWS Access
      - name: Configure AWS
        uses: withBricks/github-actions/configure-aws-access@v1
        with:
          target-account-role: arn:aws:iam::968666601602:role/BricksOUCrossAccountAccessRole

      # 2. Get DB Credentials (securely)
      - name: Get DB credentials
        id: db_creds
        uses: withBricks/github-actions/get-db-credentials@v1
        with:
          secret-id: postgresAdminRole

      # 3. Connect via Bastion
      - name: Connect to database
        id: bastion
        uses: withBricks/github-actions/connect-bastion@v1
        with:
          environment: ${{ inputs.environment }}
          db-host: postgres.equips.service

      # 4. Run Database Operations
      - name: Run migrations
        env:
          TF_VAR_postgres_host: localhost
          TF_VAR_postgres_port: 5432
        run: |
          tofu apply -auto-approve

      # 5. Cleanup (always runs)
      - name: Cleanup
        if: always()
        uses: withBricks/github-actions/cleanup-bastion@v1
        with:
          ssm-pid: ${{ steps.bastion.outputs.ssm-pid }}
```

### Infrastructure Deployment with GitHub Deployment Tracking

```yaml
name: Infrastructure Deployment

on:
  workflow_dispatch:
    inputs:
      environment:
        required: true
        type: choice
        options:
          - dev
          - prod
      project:
        required: true
        type: string

permissions:
  id-token: write
  contents: read
  deployments: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - uses: actions/checkout@v4

      # 1. Create deployment tracking
      - name: Create deployment
        id: deployment
        uses: withBricks/github-actions/create-github-deployment@v1
        with:
          github-token: ${{ github.token }}
          repository: ${{ github.repository }}
          sha: ${{ github.sha }}
          environment: ${{ inputs.environment }}
          description: "Deploy ${{ inputs.project }} to ${{ inputs.environment }}"
          initial-status-description: "Deploying infrastructure..."

      # 2. Configure AWS
      - name: Configure AWS
        uses: withBricks/github-actions/configure-aws-access@v1
        with:
          target-account-role: ${{ secrets.TARGET_ROLE_ARN }}

      # 3. Run deployment
      - name: Deploy infrastructure
        run: |
          cd ${{ inputs.project }}
          tofu init
          tofu apply -auto-approve

      # 4. Update deployment status - success
      - name: Update deployment status - success
        if: success()
        uses: withBricks/github-actions/update-github-deployment@v1
        with:
          github-token: ${{ github.token }}
          repository: ${{ github.repository }}
          deployment-id: ${{ steps.deployment.outputs.deployment-id }}
          state: success
          description: "Successfully deployed ${{ inputs.project }} to ${{ inputs.environment }}"
          environment-url: "https://${{ inputs.environment }}.example.com"

      # 5. Update deployment status - failure
      - name: Update deployment status - failure
        if: failure()
        uses: withBricks/github-actions/update-github-deployment@v1
        with:
          github-token: ${{ github.token }}
          repository: ${{ github.repository }}
          deployment-id: ${{ steps.deployment.outputs.deployment-id }}
          state: failure
          description: "Failed to deploy ${{ inputs.project }} to ${{ inputs.environment }}"
```

### GitLab Repository Sync

```yaml
name: Sync to GitLab

on:
  push:
    branches:
      - master
  workflow_dispatch:

permissions:
  contents: read

jobs:
  sync:
    runs-on: ubuntu-latest
    name: Sync repository to GitLab
    steps:
      - name: Checkout with full history
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Sync to GitLab
        uses: withBricks/github-actions/sync-to-gitlab@v1
        with:
          gitlab-token: ${{ secrets.GITLAB_SYNC_TOKEN }}
          gitlab-repo-url: https://gitlab.com/myorg/myrepo.git
          branch: master
          commit-sha: ${{ github.sha }}
          author-name: ${{ github.event.head_commit.author.name }}
          author-email: ${{ github.event.head_commit.author.email }}
          actor-name: ${{ github.actor }}
          actor-email: ${{ github.event.pusher.email || format('{0}@users.noreply.github.com', github.actor) }}
```

### 9. setup-bun-with-dependencies

Sets up Bun runtime, caches dependencies, and installs packages with frozen lockfile. Optimized for CI/CD pipelines with intelligent caching.

**Features:**
- Installs specified Bun version (or latest)
- Caches Bun install cache and node_modules for faster builds
- Installs dependencies with `--frozen-lockfile` for reproducible builds
- Supports custom working directories for monorepo setups

**Inputs:**
- `bun-version` (optional): Bun version to install (default: `1.3.0`)
- `working-directory` (optional): Working directory where package.json is located (default: `.`)

**Example:**
```yaml
- name: Setup Bun and install dependencies
  uses: withBricks/github-actions/setup-bun-with-dependencies@v1
  with:
    bun-version: 1.3.0
    working-directory: .

- name: Build project
  run: bun run build
```

**Monorepo Example:**
```yaml
- name: Setup Bun for backend
  uses: withBricks/github-actions/setup-bun-with-dependencies@v1
  with:
    bun-version: 1.3.0
    working-directory: apps/backend

- name: Build backend
  working-directory: apps/backend
  run: bun run build
```

**Requirements:**
- Project must have a `bun.lock` or `bun.lockb` lockfile
- `package.json` must exist in the working directory

### 10. configure-npm-github-packages

Configures npm to use GitHub Packages registry with authentication for scoped packages.

**Features:**
- Configures npm registry for specified scope to GitHub Packages
- Sets up authentication token for package installation/publishing
- Creates or appends to `.npmrc` file in the current directory

**Inputs:**
- `scope` (optional): Package scope (default: `@withbricks`)
- `github-token` (required): GitHub token for authentication (typically `secrets.GITHUB_TOKEN`)

**Example:**
```yaml
- name: Configure npm for GitHub Packages
  uses: withBricks/github-actions/configure-npm-github-packages@v1
  with:
    github-token: ${{ secrets.GITHUB_TOKEN }}
```

**Custom Scope Example:**
```yaml
- name: Configure npm for custom scope
  uses: withBricks/github-actions/configure-npm-github-packages@v1
  with:
    scope: '@myorg'
    github-token: ${{ secrets.GITHUB_TOKEN }}
```

**Full Workflow Example:**
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - name: Configure npm for GitHub Packages
        uses: withBricks/github-actions/configure-npm-github-packages@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Install dependencies
        run: npm ci
      
      - name: Build
        run: npm run build
```

## Security Considerations

### Password Security
- ✅ Passwords masked before output using GitHub Actions `::add-mask::`
- ✅ Never retrieves full secret JSON (only extracts needed fields)
- ✅ Credentials not logged or exposed in workflow output
- ⚠️ Do NOT echo credential outputs or include in command strings

### Network Security
- All database connections go through private subnets via bastion
- Bastion has no inbound internet access (egress only)
- SSM sessions use AWS-managed encryption
- No public IP required on bastion instance

### Access Control
- OIDC authentication (no long-lived credentials)
- Cross-account role assumption with minimal permissions
- SSM access controlled by IAM policies
- Bastion access controlled by security groups

## Troubleshooting

### Bastion Not Found
```
Error: Bastion instance not found or not running
```
**Solution:** Verify EC2 instance exists with tag `Name={environment}-github-bastion` and is in running state.

### SSM Agent Offline
```
SSM status: ConnectionLost, waiting...
```
**Solution:** 
1. Check instance has `AmazonSSMManagedInstanceCore` IAM policy
2. Verify SSM agent is running: `systemctl status amazon-ssm-agent`
3. Check VPC endpoints for SSM are accessible

### Port Forwarding Timeout
```
Warning: Port forwarding may not be fully established
```
**Solution:**
1. Verify bastion can reach database host (security groups, NACLs)
2. Check database is accessible on specified port
3. Increase timeout in connect-bastion action if needed

### Password Exposure
If you suspect credentials were logged:
1. Rotate the secret immediately in Secrets Manager
2. Review workflow logs for the masked value `***`
3. Check `::add-mask::` is called before any output

## Contributing

Changes to these actions should be tested before merging to main:
1. Create a feature branch
2. Test changes in a workflow using `@your-branch-name`
3. Open PR for review
4. After merge, create a new release using the automated release workflow

## Releasing New Versions

This repository uses an automated release workflow to manage versions.

### Creating a Release

1. Go to **Actions** → **Release Version** workflow
2. Click **Run workflow**
3. Select the version bump type:
   - **patch**: Bug fixes and minor updates (e.g., `v1.0.0` → `v1.0.1`)
   - **minor**: New features, backwards compatible (e.g., `v1.0.1` → `v1.1.0`)
   - **major**: Breaking changes (e.g., `v1.1.0` → `v2.0.0`)
4. Choose whether to create a GitHub release (recommended)
5. Click **Run workflow**

The workflow will:
- ✅ Calculate the next version number
- ✅ Create a new version tag (e.g., `v1.2.3`)
- ✅ Update the major version tag (e.g., `v1`) to point to the new version
- ✅ Generate a changelog from commits
- ✅ Create a GitHub release (if selected)

### Version Tag Strategy

- **Specific versions** (`v1.2.3`): Never change, guaranteed stability
- **Major version tags** (`v1`, `v2`): Auto-update to latest minor/patch, recommended for most users
- **Branch** (`@main`): Latest development, not recommended for production

**Example usage in workflows:**
```bash
# Recommended: Auto-updates with patches and features
uses: withBricks/github-actions/configure-aws-access@v1

# Pin to exact version: Never changes
uses: withBricks/github-actions/configure-aws-access@v1.2.3

# Latest development: Use for testing only
uses: withBricks/github-actions/configure-aws-access@main
```

### Manual Release (Advanced)

If you need to create a release manually:

```bash
# Create a new version tag
git tag -a v1.0.1 -m "Release v1.0.1"
git push origin v1.0.1

# Update the major version tag to point to the new version
git tag -f v1
git push -f origin v1
```
