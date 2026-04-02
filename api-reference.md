# Volley API Reference

## Base URL

All API endpoints are served from your API Gateway deployment:

```
https://{api-id}.execute-api.{region}.amazonaws.com/prod
```

The base URL is printed at the end of deployment. It is also stored as the `VITE_API_URL` environment variable.

## Authentication

Most endpoints require authentication via a Bearer token in the `Authorization` header:

```
Authorization: Bearer {access_token}
```

Obtain an access token by calling `POST /auth/login`. Tokens expire after a configured period. Use `POST /auth/refresh` with a valid refresh token cookie to obtain a new access token.

### Roles

The platform has three user roles with ascending permissions:

| Role | Permissions |
|------|------------|
| `viewer` | Read-only access to contacts, templates, campaigns, and analytics |
| `editor` | Everything viewer can do, plus create/edit/delete contacts, templates, and campaigns |
| `admin` | Everything editor can do, plus user management, domain settings, and integrations |

## Endpoint Groups

---

### Auth (`/auth/*`)

Authentication endpoints. No Bearer token required.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| POST | `/auth/login` | No | Authenticate and receive access token |
| POST | `/auth/register` | No | Create a new user account |
| POST | `/auth/refresh` | No | Refresh access token using refresh token cookie |
| POST | `/auth/logout` | No | Invalidate refresh token |
| POST | `/auth/verify-email` | No | Verify email address with token |
| POST | `/auth/forgot-password` | No | Request a password reset email |
| POST | `/auth/reset-password` | No | Reset password with token |
| POST | `/auth/accept-invite` | No | Accept an admin invitation and set password |

**POST /auth/login**

```json
// Request
{ "email": "user@example.com", "password": "yourpassword" }

// Response 200
{ "user": { "id": "...", "email": "...", "role": "admin" }, "accessToken": "eyJ..." }
// Sets httpOnly refresh token cookie
```

**POST /auth/register**

```json
// Request
{ "email": "user@example.com", "password": "yourpassword", "name": "John Doe" }

// Response 201
{ "user": { "id": "...", "email": "...", "role": "viewer" }, "accessToken": "eyJ..." }
```

**POST /auth/refresh**

```json
// Request: No body. Refresh token sent via httpOnly cookie.

// Response 200
{ "accessToken": "eyJ..." }
```

**POST /auth/logout**

```json
// Request: No body. Refresh token sent via cookie.

// Response 200
{ "message": "Logged out" }
```

**POST /auth/verify-email**

```json
// Request
{ "token": "verification-token-from-email" }

// Response 200
{ "message": "Email verified" }
```

**POST /auth/forgot-password**

```json
// Request
{ "email": "user@example.com" }

// Response 200
{ "message": "If the email exists, a reset link has been sent" }
```

**POST /auth/reset-password**

```json
// Request
{ "token": "reset-token-from-email", "password": "newpassword" }

// Response 200
{ "message": "Password reset successful" }
```

**POST /auth/accept-invite**

```json
// Request
{ "token": "invite-token", "password": "yourpassword", "name": "John Doe" }

// Response 200
{ "user": { "id": "...", "email": "...", "role": "..." }, "accessToken": "eyJ..." }
```

---

### Admin (`/admin/*`)

User management endpoints. All routes require the `admin` role.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/admin/users` | Admin | List all users |
| POST | `/admin/users` | Admin | Create a new user directly |
| POST | `/admin/users/invite` | Admin | Send an invitation email to a new user |
| PUT | `/admin/users/{userId}` | Admin | Update a user's name or role |
| POST | `/admin/users/{userId}/disable` | Admin | Disable or enable a user account |

**GET /admin/users**

```json
// Response 200
{ "users": [{ "id": "...", "email": "...", "name": "...", "role": "editor", "status": "active" }] }
```

**POST /admin/users**

```json
// Request
{ "email": "new@example.com", "password": "temppassword", "name": "New User", "role": "editor" }

// Response 201
{ "user": { "id": "...", "email": "...", "name": "...", "role": "editor" } }
```

**POST /admin/users/invite**

```json
// Request
{ "email": "invited@example.com", "role": "editor" }

// Response 200
{ "message": "Invitation sent" }
```

**PUT /admin/users/{userId}**

```json
// Request
{ "name": "Updated Name", "role": "admin" }

// Response 200
{ "user": { "id": "...", "email": "...", "name": "Updated Name", "role": "admin" } }
```

**POST /admin/users/{userId}/disable**

```json
// Request
{ "disabled": true }

// Response 200
{ "user": { "id": "...", "status": "disabled" } }
```

---

### Contacts (`/contacts/*`)

Contact management and segmentation. Read operations require any authenticated role. Write operations (POST, PUT, DELETE) require `admin` or `editor` role.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/contacts` | Yes | List contacts with pagination |
| POST | `/contacts` | Editor+ | Create a new contact |
| GET | `/contacts/{contactId}` | Yes | Get a single contact |
| PUT | `/contacts/{contactId}` | Editor+ | Update a contact |
| DELETE | `/contacts/{contactId}` | Editor+ | Delete a contact |
| POST | `/contacts/import` | Editor+ | Import contacts from CSV data |
| GET | `/contacts/export` | Yes | Export all contacts as CSV |
| GET | `/contacts/segments` | Yes | List all segments |
| POST | `/contacts/segments` | Editor+ | Create a new segment |
| GET | `/contacts/segments/{segmentId}` | Yes | Get segment details with member count |
| PUT | `/contacts/segments/{segmentId}` | Editor+ | Update a segment |
| DELETE | `/contacts/segments/{segmentId}` | Editor+ | Delete a segment |

**GET /contacts**

```json
// Query params: ?limit=50&lastKey=...&search=john
// Response 200
{ "contacts": [{ "id": "...", "email": "...", "firstName": "...", "lastName": "...", "tags": [] }], "lastKey": "..." }
```

**POST /contacts**

```json
// Request
{ "email": "contact@example.com", "firstName": "Jane", "lastName": "Doe", "tags": ["newsletter"], "customFields": { "company": "Acme" } }

// Response 201
{ "contact": { "id": "...", "email": "...", "firstName": "Jane", "lastName": "Doe" } }
```

**POST /contacts/import**

```json
// Request
{ "contacts": [{ "email": "a@b.com", "firstName": "A" }, ...], "mapping": { "email": "email", "firstName": "first_name" } }

// Response 200
{ "imported": 95, "skipped": 5, "errors": [] }
```

**GET /contacts/export**

```
// Response 200
// Content-Type: text/csv
email,firstName,lastName,tags,createdAt
contact@example.com,Jane,Doe,"newsletter",2026-03-15T10:00:00Z
```

**POST /contacts/segments**

```json
// Request
{ "name": "Active Subscribers", "rules": [{ "field": "tags", "operator": "contains", "value": "active" }] }

// Response 201
{ "segment": { "id": "...", "name": "Active Subscribers", "rules": [...], "memberCount": 150 } }
```

---

### Templates (`/templates/*`)

Email template management. Read operations require any authenticated role. Write operations require `admin` or `editor` role.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/templates` | Yes | List all templates |
| POST | `/templates` | Editor+ | Create a new template |
| GET | `/templates/{templateId}` | Yes | Get template details and content |
| PUT | `/templates/{templateId}` | Editor+ | Update template content |
| DELETE | `/templates/{templateId}` | Editor+ | Delete a template |
| POST | `/templates/{templateId}/render` | Editor+ | Render template with sample data |
| POST | `/templates/{templateId}/duplicate` | Editor+ | Duplicate a template |
| GET | `/templates/{templateId}/versions` | Yes | List version history |
| POST | `/templates/{templateId}/revert` | Editor+ | Revert to a previous version |

**POST /templates**

```json
// Request
{ "name": "Welcome Email", "subject": "Welcome, {{firstName}}!", "mjml": "<mjml>...</mjml>" }

// Response 201
{ "template": { "id": "...", "name": "Welcome Email", "version": 1 } }
```

**PUT /templates/{templateId}**

```json
// Request
{ "name": "Updated Welcome", "subject": "Hello {{firstName}}!", "mjml": "<mjml>...</mjml>" }

// Response 200
{ "template": { "id": "...", "name": "Updated Welcome", "version": 2 } }
```

**POST /templates/{templateId}/render**

```json
// Request
{ "data": { "firstName": "Jane", "email": "jane@example.com" } }

// Response 200
{ "html": "<html>...</html>" }
```

**POST /templates/{templateId}/revert**

```json
// Request
{ "version": 1 }

// Response 200
{ "template": { "id": "...", "version": 3 } }
// Note: revert creates a NEW version with the content of the target version
```

---

### Campaigns (`/campaigns/*`)

Campaign lifecycle management, analytics, and suppression. Most routes require `admin` or `editor` role. Unsubscribe route is public.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/campaigns` | Editor+ | List all campaigns |
| POST | `/campaigns` | Editor+ | Create a new campaign |
| GET | `/campaigns/{campaignId}` | Editor+ | Get campaign details |
| PUT | `/campaigns/{campaignId}` | Editor+ | Update a draft campaign |
| DELETE | `/campaigns/{campaignId}` | Editor+ | Delete a campaign |
| POST | `/campaigns/{campaignId}/schedule` | Editor+ | Schedule a campaign for future delivery |
| POST | `/campaigns/{campaignId}/cancel` | Editor+ | Cancel a scheduled campaign |
| POST | `/campaigns/{campaignId}/execute` | Editor+ | Send a campaign immediately |
| GET | `/campaigns/{campaignId}/audience` | Editor+ | Preview audience count for a campaign |
| GET | `/campaigns/{campaignId}/analytics` | Editor+ | Get per-campaign analytics |
| GET | `/campaigns/analytics` | Editor+ | Get global analytics across all campaigns |
| GET | `/campaigns/unsubscribe` | No | Process email unsubscribe (from email link) |
| POST | `/campaigns/unsubscribe` | No | Process email unsubscribe (RFC 8058 List-Unsubscribe-Post) |
| GET | `/campaigns/suppressions` | Editor+ | List suppressed email addresses |
| POST | `/campaigns/suppressions` | Editor+ | Manually add an email to suppression list |
| DELETE | `/campaigns/suppressions/{email}` | Editor+ | Remove an email from suppression list |

**POST /campaigns**

```json
// Request
{ "name": "April Newsletter", "subject": "Our April Update", "templateId": "tpl_123", "segmentId": "seg_456" }

// Response 201
{ "campaign": { "id": "...", "name": "April Newsletter", "status": "draft", "templateName": "...", "segmentName": "..." } }
```

**POST /campaigns/{campaignId}/schedule**

```json
// Request
{ "scheduledAt": "2026-04-15T09:00:00Z" }

// Response 200
{ "campaign": { "id": "...", "status": "scheduled", "scheduledAt": "2026-04-15T09:00:00Z" } }
```

**POST /campaigns/{campaignId}/execute**

```json
// Request: No body

// Response 200
{ "campaign": { "id": "...", "status": "executing" } }
```

**GET /campaigns/{campaignId}/analytics**

```json
// Response 200
{
  "analytics": {
    "totalRecipients": 500,
    "deliveredCount": 490,
    "openCount": 245,
    "uniqueOpenCount": 200,
    "clickCount": 80,
    "uniqueClickCount": 65,
    "bounceCount": 8,
    "complaintCount": 2,
    "failedCount": 10
  }
}
```

**GET /campaigns/analytics**

```json
// Query params: ?startDate=2026-03-01&endDate=2026-03-31&status=sent
// Response 200
{
  "analytics": {
    "totalCampaigns": 12,
    "totalRecipients": 6000,
    "deliveredCount": 5800,
    "openCount": 2900,
    "clickCount": 870,
    "bounceCount": 150,
    "complaintCount": 50
  }
}
```

**GET /campaigns/unsubscribe**

```
// Query params: ?token={hmac-token}&email={email}&campaignId={id}
// Response 200: HTML confirmation page
```

---

### Domain (`/domain/*`)

Email sending domain management. All routes require the `admin` role.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/domain/status` | Admin | Get domain verification status and DKIM records |
| GET | `/domain/reputation` | Admin | Get SES sending reputation metrics |
| POST | `/domain/verify` | Admin | Initiate or re-check domain verification |

**GET /domain/status**

```json
// Response 200
{
  "domain": "example.com",
  "verified": true,
  "dkimStatus": "SUCCESS",
  "dkimTokens": ["token1", "token2", "token3"]
}
```

**GET /domain/reputation**

```json
// Response 200
{
  "sendingEnabled": true,
  "reputationScore": 98.5,
  "bounceRate": 1.2,
  "complaintRate": 0.05
}
```

**POST /domain/verify**

```json
// Request: No body

// Response 200
{ "domain": "example.com", "status": "PENDING", "dkimTokens": ["token1", "token2", "token3"] }
// Idempotent: returns current status if already verified
```

---

### Integrations (`/integrations/*`)

CRM integration management. All routes require the `admin` role.

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| GET | `/integrations` | Admin | List all configured integrations |
| GET | `/integrations/hubspot` | Admin | Get HubSpot integration config |
| POST | `/integrations/hubspot/connect` | Admin | Connect HubSpot with API token |
| POST | `/integrations/hubspot/sync` | Admin | Sync contacts with HubSpot |
| DELETE | `/integrations/hubspot` | Admin | Disconnect HubSpot integration |
| GET | `/integrations/zoho` | Admin | Get Zoho CRM integration config |
| POST | `/integrations/zoho/connect` | Admin | Connect Zoho CRM with refresh token |
| POST | `/integrations/zoho/sync` | Admin | Sync contacts with Zoho CRM |
| DELETE | `/integrations/zoho` | Admin | Disconnect Zoho CRM integration |

**POST /integrations/hubspot/connect**

```json
// Request
{ "accessToken": "pat-na1-..." }

// Response 200
{ "integration": { "provider": "hubspot", "accessToken": "****na1-", "connectedAt": "..." } }
```

**POST /integrations/hubspot/sync**

```json
// Request: No body

// Response 200
{ "sync": { "pulled": 50, "pushed": 30, "errors": 0 } }
```

**POST /integrations/zoho/connect**

```json
// Request
{ "refreshToken": "1000.xxxx.yyyy", "apiDomain": "https://www.zohoapis.com", "accountsDomain": "https://accounts.zoho.com" }

// Response 200
{ "integration": { "provider": "zoho", "accessToken": "****xxxx", "connectedAt": "..." } }
```

**POST /integrations/zoho/sync**

```json
// Request: No body

// Response 200
{ "sync": { "pulled": 75, "pushed": 40, "errors": 2 } }
```

---

## Error Responses

All endpoints return errors in a consistent format:

```json
{
  "error": "Description of the error"
}
```

Common HTTP status codes:

| Status | Meaning |
|--------|---------|
| 400 | Bad Request -- Missing or invalid parameters |
| 401 | Unauthorized -- Missing or invalid Bearer token |
| 403 | Forbidden -- Insufficient role permissions |
| 404 | Not Found -- Resource does not exist |
| 409 | Conflict -- Resource state conflict (e.g., campaign already executing) |
| 500 | Internal Server Error -- Unexpected server error |
| 501 | Not Implemented -- Feature requires configuration (e.g., EMAIL_QUEUE_URL not set) |

## Rate Limits

The platform does not enforce application-level rate limits. AWS service limits apply:

- **API Gateway:** 10,000 requests/second (default)
- **Lambda:** 1,000 concurrent executions (default)
- **SES:** Varies by account (sandbox: 200/day, production: varies)
- **DynamoDB:** On-demand capacity (auto-scales)

Request limit increases through the AWS Service Quotas console if needed.
