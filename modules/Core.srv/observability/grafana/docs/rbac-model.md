# Grafana RBAC Configuration

## Current Setup
- **Admin User**: Your account (configured in .env)
- **Organization**: Main Org (default)

## Role Definitions

### Admin
- **Who**: System owner
- **Permissions**: Full control
- **Use cases**: 
  - Configure datasources
  - Manage users
  - System configuration
  - Install plugins

### Editor
- **Who**: DevOps/Operations team (future)
- **Permissions**: 
  - Create/edit dashboards
  - Create/edit alerts
  - View datasources (read-only)
- **Use cases**:
  - Build new dashboards
  - Adjust alert thresholds
  - Day-to-day monitoring

### Viewer
- **Who**: Stakeholders, audit (future)
- **Permissions**:
  - View dashboards only
  - Cannot modify anything
- **Use cases**:
  - Check system status
  - Review alerts
  - Compliance auditing

## Adding New Users (Future)

To add users when needed:
1. Go to Administration → Users → Invite
2. Email them an invite link
3. Assign appropriate role (Editor or Viewer)
4. Admin role should be restricted to system owner only

## Service Account Tokens (Future - Phase 5)

For n8n alert webhook integration:
1. Administration → Service accounts → Create service account
2. Name: "n8n-alerting"
3. Role: Viewer (only needs to receive webhooks)
4. Generate token for n8n to authenticate
