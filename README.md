# Bifrost Community Workspace

A community-maintained [Bifrost](https://bifrost.sh) workspace for MSPs. Contains production-ready modules, AI agents, workflows, and apps that any MSP can deploy and customize.

## What's Included

### Features

**HaloPSA Report Agent** — An AI agent that generates HaloPSA SQL reports from natural language. Searches its knowledge base for schema patterns, writes and executes queries, iterates on errors, and saves what it learns for next time.

**Microsoft CSP App** — A full React application for managing Microsoft CSP tenants. Links tenants to Bifrost organizations, handles application consent, manages GDAP relationships and role assignments, and provides batch operations.

**AutoElevate Integration** — An AI agent that reviews AutoElevate privilege elevation requests against your approval policy and autonomously approves, creates rules, or escalates to a human tech.

### Modules (MSP Integration SDKs)

| Module | Description |
|--------|-------------|
| `modules/halopsa.py` | HaloPSA PSA platform |
| `modules/autoelevate.py` | AutoElevate privilege elevation (with TOTP MFA) |
| `modules/ninjaone.py` | NinjaOne RMM |
| `modules/huntress.py` | Huntress EDR |
| `modules/itglue.py` | IT Glue documentation (US/EU/AU) |
| `modules/pax8.py` | Pax8 distributor (OAuth2) |
| `modules/cove.py` | Cove Data Protection / N-able Backup |
| `modules/sendgrid.py` | SendGrid email |
| `modules/immybot.py` | ImmyBot software deployment |
| `modules/microsoft/` | Microsoft Graph, CSP, GDAP, Exchange, Auth |

### Extension Helpers

| Extension | Description |
|-----------|-------------|
| `modules/extensions/halopsa.py` | Pagination, enriched tickets, batch ops, SQL execution, ticket creation |
| `modules/extensions/ninjaone.py` | Remote PowerShell execution via fetch-and-execute pattern |
| `modules/extensions/sendgrid.py` | Higher-level email sending with integration config |
| `modules/extensions/permissions.py` | Bifrost RBAC role-checking and authorization |

### Shared Tools

- **HaloPSA tools** — Auth-checked ticket operations, notes, agreements, time entry
- **Microsoft tools** — Email via Graph API, Exchange data providers
- **Bifrost utilities** — Organization management, role management, permissions

## Prerequisites

- [Bifrost CLI](https://docs.bifrost.sh) installed and authenticated
- Accounts with the integrations you plan to use (HaloPSA, Microsoft 365, AutoElevate, etc.)

## Setup

1. Clone this repo:
   ```bash
   git clone https://github.com/jackmusick/bifrost-workspace-community.git
   cd bifrost-workspace-community
   ```

2. Initialize Bifrost:
   ```bash
   bifrost init
   ```

3. Configure your integrations in the Bifrost dashboard — each integration needs your credentials and org mappings.

4. For the **Elevation Agent**, set the `autoelevate_approval_policy` config with your organization's approval policy text.

5. For the **Microsoft CSP App**, update the `RESELLER_LINK` in `apps/microsoft-csp/components/TenantTable.tsx` with your Partner Center reseller invitation URL.

## Configuration

Key config values to set (via Bifrost dashboard or `.bifrost/configs.yaml`):

| Config | Used By | Description |
|--------|---------|-------------|
| `autoelevate_approval_policy` | Elevation Agent | Your approval policy text |
| `autoelevate_approval_email_template_id` | Elevation Agent | HaloPSA email template for approvals |
| `autoelevate_denial_email_template_id` | Elevation Agent | HaloPSA email template for denials |
| `ninja_script_id` | NinjaOne Extension | Pre-deployed script ID for remote execution |

## Contributing

Contributions are welcome! This workspace is meant to grow with the MSP community. If you've built something useful on Bifrost, consider adding it here.

1. Fork the repo
2. Create a feature branch
3. Add your module, workflow, or app
4. Submit a pull request

Please ensure any contributed code is generalized (no org-specific IDs, credentials, or customer data).

## License

MIT
