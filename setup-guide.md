# Volley Setup Guide

## Overview

Volley is a self-contained email marketing platform that you deploy into your own AWS account. It provides contact management, email template creation, campaign scheduling and delivery, analytics, and CRM integrations. A single deployment provisions all required AWS resources under a unique tenant prefix, giving you full control over your data and infrastructure.

## Prerequisites

Before deploying, ensure you have the following installed and configured:

| Requirement | Minimum Version | How to Check |
|-------------|----------------|--------------|
| AWS Account | Admin access (IAM) | `aws sts get-caller-identity` |
| AWS CLI | v2.x | `aws --version` |
| Node.js | 22+ | `node --version` |
| npm | 10+ | `npm --version` |
| jq | 1.6+ | `jq --version` |
| Bash | 5+ | `bash --version` |

Your AWS CLI must be configured with credentials that have administrator access. Run `aws configure` if you have not already set this up.

## Configuration

### Step 1: Create your environment file

Copy the example environment file and fill in your values:

```bash
cp .env.example .env
```

### Step 2: Configure environment variables

Open `.env` in your editor and set the following variables:

#### Required Variables

| Variable | Description | Format / Example |
|----------|-------------|-----------------|
| `TENANT_PREFIX` | Unique identifier for this tenant's AWS resources. Used as a prefix for all resource names (DynamoDB tables, S3 buckets, Lambda functions, etc.). | Lowercase alphanumeric, e.g., `acme` |
| `AWS_REGION` | AWS region where all resources will be provisioned. | AWS region code, e.g., `us-east-1`, `ap-southeast-2` |
| `JWT_SECRET` | Random string used to sign authentication tokens. Must be at least 32 characters. Generate with `openssl rand -base64 48`. | Random string, min 32 chars |
| `SENDER_DOMAIN` | Domain you own that will be used for sending emails. Must be verified in Amazon SES. | Domain name, e.g., `mail.example.com` |
| `ADMIN_EMAIL` | Email address for the initial admin account created during setup. | Valid email, e.g., `admin@example.com` |
| `ADMIN_PASSWORD` | Password for the initial admin account. | Strong password, min 8 chars |
| `UNSUBSCRIBE_SECRET` | Random string used to sign one-click unsubscribe tokens (RFC 8058). Must be at least 32 characters. Generate with `openssl rand -base64 48`. | Random string, min 32 chars |
| `FRONTEND_URL` | URL where the dashboard will be accessible. Set to the CloudFront URL after first deployment, or your custom domain. | URL, e.g., `https://d1234abcd.cloudfront.net` |

#### Branding Variables (Optional)

| Variable | Description | Default |
|----------|-------------|---------|
| `VITE_APP_NAME` | Application name displayed in the dashboard header and browser tab. | `Volley` |
| `VITE_LOGO_URL` | URL to your company logo image. Leave empty for text-only branding. | Empty (text only) |
| `VITE_PRIMARY_COLOR` | Primary accent color for the dashboard UI. | `#2563EB` (blue) |
| `APP_NAME` | Application name used in transactional emails (verification, password reset). | `Volley` |
| `LOGO_URL` | Logo URL used in transactional email headers. | Empty |
| `PRIMARY_COLOR` | Primary color used in transactional email styling. | `#2563EB` |

#### CRM Integration Variables (Optional)

Set these only if you plan to sync contacts with a CRM system. They can be configured later.

| Variable | Description | Where to Get It |
|----------|-------------|----------------|
| `HUBSPOT_ACCESS_TOKEN` | HubSpot Private App token for contact sync. | HubSpot Settings > Integrations > Private Apps |
| `ZOHO_CLIENT_ID` | Zoho CRM OAuth client ID. | Zoho API Console (api-console.zoho.com) > Self Client |
| `ZOHO_CLIENT_SECRET` | Zoho CRM OAuth client secret. | Same as above |
| `ZOHO_REFRESH_TOKEN` | Zoho CRM OAuth refresh token. | Generated via Zoho Self Client flow |
| `ZOHO_API_DOMAIN` | Zoho API base URL (varies by data center region). | `https://www.zohoapis.com` (US), `https://www.zohoapis.eu` (EU), etc. |
| `ZOHO_ACCOUNTS_DOMAIN` | Zoho accounts URL for token refresh (varies by region). | `https://accounts.zoho.com` (US), `https://accounts.zoho.eu` (EU), etc. |

#### AI Assistant Variables (Optional)

Set these only if you want to use AI-powered features (email content generation, subject line suggestions). AI features are optional and the platform works fully without them.

| Variable | Description | Default |
|----------|-------------|---------|
| `BEDROCK_REGION` | AWS region for Amazon Bedrock API calls. Use `us-east-1` for best model availability. | `us-east-1` |
| `AI_MODEL_ID` | Bedrock model ID for AI features. Cross-region inference IDs (`us.` prefix) are recommended for reliability. | `us.anthropic.claude-haiku-4-5-20251001-v1:0` |

**Bedrock Model Access (Required for AI features):**

Before AI features will work, you must enable model access in the AWS Console:

1. Navigate to **Amazon Bedrock** > **Model access** > **Manage model access**
2. Enable access for **Anthropic Claude 3.5 Haiku** (or the model specified in `AI_MODEL_ID`)
3. Wait for the access status to show "Access granted" (typically immediate)

If Bedrock model access is not enabled, AI features will return errors but all other platform features (contacts, templates, campaigns, analytics) will work normally.

#### Other Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `VITE_API_URL` | API Gateway URL. Set automatically after API Gateway is provisioned. For local development, use `http://localhost:3001/api`. | `http://localhost:3001/api` |
| `USERS_TABLE_NAME` | DynamoDB table name for auth data. Auto-derived from TENANT_PREFIX. | `edm-users` |

## Deployment

### Step 1: Install dependencies

```bash
npm install
```

This installs all packages for both the frontend and backend workspaces.

### Step 2: Run the deploy script

```bash
bash scripts/deploy.sh
```

The deploy script will:

1. Validate your `.env` configuration
2. Provision DynamoDB tables (`{TENANT_PREFIX}-users`, `{TENANT_PREFIX}-data`)
3. Create S3 buckets for templates and frontend assets
4. Configure SES domain identity and email sender
5. Create SQS queues for email delivery
6. Deploy Lambda functions (auth, contacts, campaigns, templates, admin, domain, integrations, sender, webhook, authorizer, ai)
7. Set up API Gateway with Lambda proxy integrations
8. Configure EventBridge Scheduler for campaign scheduling
9. Set up SES event tracking (bounces, complaints, deliveries)
10. Build and deploy the frontend to S3 + CloudFront
11. Seed the admin account and starter templates

Deployment takes approximately 5-10 minutes.

### Step 3: Configure DNS for SES domain verification

After deployment, you must add DNS records to verify your sender domain with Amazon SES. The deploy script will output the required DKIM and SPF records.

**DKIM records** (3 CNAME records):

The script outputs three CNAME records. Add each to your domain's DNS:

```
Type: CNAME
Name: {token1}._domainkey.yourdomain.com
Value: {token1}.dkim.amazonses.com

Type: CNAME
Name: {token2}._domainkey.yourdomain.com
Value: {token2}.dkim.amazonses.com

Type: CNAME
Name: {token3}._domainkey.yourdomain.com
Value: {token3}.dkim.amazonses.com
```

**SPF record** (TXT record):

```
Type: TXT
Name: yourdomain.com
Value: "v=spf1 include:amazonses.com ~all"
```

DNS propagation can take up to 72 hours, but typically completes within a few hours. Check verification status in the Settings > Domains page of the dashboard.

### Step 4: Access the dashboard

The deploy script prints the CloudFront URL at the end of execution. Open this URL in your browser and log in with the admin credentials you configured in `.env`.

```
Dashboard URL: https://d1234abcd.cloudfront.net
```

## Updating

### Frontend-only changes

If you have made changes only to the frontend code (UI, styling, branding):

```bash
bash scripts/deploy-frontend.sh
```

This rebuilds the Vite frontend, uploads assets to S3, and invalidates the CloudFront cache. Changes are live within a few minutes.

### Full redeploy

For backend changes or full redeployment:

```bash
bash scripts/deploy.sh
```

The deploy script is idempotent. It checks whether each resource already exists before creating it, and updates Lambda function code for existing functions. You can run it safely at any time.

## Teardown

To remove all AWS resources for this tenant:

```bash
bash scripts/teardown.sh
```

The teardown script will prompt for confirmation before proceeding. It deletes resources in reverse dependency order:

1. CloudFront distribution (disable, wait for propagation, then delete)
2. S3 bucket contents and buckets
3. API Gateway
4. Lambda functions and IAM roles
5. SQS queues
6. EventBridge scheduler group
7. SNS topics and SES configuration
8. DynamoDB tables

**Warning:** This permanently deletes all data including contacts, templates, campaigns, and analytics. This action cannot be undone.

## Troubleshooting

### SES Sandbox Restrictions

**Problem:** Emails are not being delivered to recipients.

**Cause:** New AWS accounts start in the SES sandbox, which only allows sending to verified email addresses.

**Solution:** Request production access through the AWS console:
1. Go to Amazon SES > Account dashboard
2. Click "Request production access"
3. Fill in the form describing your use case
4. AWS typically approves requests within 24 hours

While in sandbox mode, you can still test by adding recipient email addresses as verified identities in SES.

### IAM Permission Errors

**Problem:** Deploy script fails with "Access Denied" or "UnauthorizedAccess" errors.

**Cause:** Your AWS CLI credentials do not have sufficient permissions.

**Solution:** Ensure your IAM user or role has administrator access, or at minimum has permissions for: DynamoDB, S3, Lambda, API Gateway, SES, SQS, SNS, EventBridge, CloudFront, IAM, and CloudWatch Logs.

### Domain Verification Pending

**Problem:** Domain status shows "Pending" in the dashboard after adding DNS records.

**Cause:** DNS propagation has not completed yet.

**Solution:** Wait up to 72 hours for DNS propagation. You can check the current status with:

```bash
aws sesv2 get-email-identity --email-identity yourdomain.com --region your-region
```

Verify that the DKIM CNAME records are correctly configured using a DNS lookup tool.

### Frontend Shows Localhost API URL

**Problem:** The deployed dashboard makes requests to `http://localhost:3001` instead of the API Gateway URL.

**Cause:** `VITE_API_URL` was not set to the API Gateway URL before building the frontend. Vite inlines environment variables at build time.

**Solution:** Set `VITE_API_URL` to your API Gateway invoke URL in `.env`, then rebuild and redeploy the frontend:

```bash
bash scripts/deploy-frontend.sh
```

### CloudFront Returns 403 on Page Refresh

**Problem:** Navigating directly to a URL like `/contacts` returns a 403 error.

**Cause:** CloudFront custom error responses are not configured for SPA routing.

**Solution:** This is handled automatically by the deploy script. If you encounter this issue, verify that the CloudFront distribution has custom error responses mapping 403 and 404 to `/index.html` with a 200 status code.
