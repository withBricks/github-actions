# Bricks Shared GitHub Actions

Reusable composite actions for AWS infrastructure automation with database access via EC2 bastion.

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

## Complete Workflow Example

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
4. After merge, create a new release tag

### Creating a Release

```bash
# Patch version (bug fixes)
git tag -a v1.0.1 -m "Fix SSM timeout handling"
git push origin v1.0.1

# Update v1 major version tag
git tag -f v1
git push -f origin v1
```
