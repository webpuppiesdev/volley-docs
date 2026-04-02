# Volley User Manual

## Getting Started

### Logging In

Open the dashboard URL provided by your administrator. Enter your email address and password on the login page, then click "Sign In."

If this is your first time logging in via an invitation, use the link in your invitation email to set your password before signing in.

### Navigating the Dashboard

The dashboard has a sidebar on the left with links to all major sections:

- **Dashboard** -- Overview and quick stats
- **Contacts** -- Manage your email recipients
- **Campaigns** -- Create and send email campaigns
- **Templates** -- Design email templates
- **Analytics** -- View delivery and engagement metrics
- **Suppression** -- Manage suppressed email addresses
- **Settings** -- User management, domain configuration, and integrations

Click any item in the sidebar to navigate to that section.

### First-Time Setup

After your first login as an administrator, complete these steps:

1. **Verify your sending domain** -- Go to Settings > Domains and confirm that your domain verification is complete. If DNS records are still pending, add the displayed DKIM and SPF records to your domain.
2. **Review starter templates** -- Go to Templates to see the pre-built starter templates. You can use these as-is or customize them.
3. **Import your contacts** -- Go to Contacts and use the Import feature to upload your existing contact list from a CSV file.

---

## Contacts

The Contacts section is where you manage all email recipients.

### Viewing Contacts

The contacts page displays a searchable, paginated list of all your contacts. Each row shows the contact's email, name, tags, and creation date. Use the search bar at the top to filter contacts by name or email.

### Adding a Single Contact

1. Click the "Add Contact" button.
2. Fill in the contact details: email (required), first name, last name, tags, and any custom fields.
3. Click "Save" to create the contact.

Email addresses must be unique. If you try to add an email that already exists, you will receive an error.

### Editing a Contact

1. Click on a contact in the list to open their detail page.
2. Edit any field (name, tags, custom fields).
3. Click "Save" to apply changes.

### Deleting a Contact

1. Open the contact's detail page.
2. Click the "Delete" button.
3. Confirm the deletion when prompted.

Deleting a contact is permanent and cannot be undone.

### CSV Import

To import contacts in bulk from a CSV file:

1. Click the "Import" button on the contacts page.
2. Select your CSV file. The file should have headers in the first row.
3. The import wizard displays a column mapping screen. Map each CSV column to the corresponding contact field (email, first name, last name, tags, custom fields).
4. Review the preview showing the first few rows with your mapping applied.
5. Click "Import" to begin the import.

The system processes contacts in batches of 100. Duplicate emails are automatically skipped. After import completes, you will see a summary showing how many contacts were imported and how many were skipped.

**Supported CSV format:** UTF-8 encoded, comma-separated, with a header row. Maximum recommended file size is 10MB.

### CSV Export

To export all contacts to a CSV file:

1. Click the "Export" button on the contacts page.
2. A CSV file will download containing all contacts with their email, name, tags, custom fields, and creation date.

---

## Segments

Segments are dynamic groups of contacts defined by filter rules. When you use a segment as a campaign audience, the platform evaluates the rules at send time to determine which contacts match.

### Creating a Segment

1. Navigate to Contacts > Segments.
2. Click "New Segment."
3. Enter a segment name.
4. Add one or more filter rules. Each rule consists of:
   - **Field** -- The contact field to filter on (e.g., tags, custom fields, engagement data).
   - **Operator** -- The comparison type (e.g., "contains," "equals," "greater than").
   - **Value** -- The value to compare against.
5. The member count updates as you add rules, showing how many contacts currently match.
6. Click "Save" to create the segment.

### Rule Examples

- **Tags contain "newsletter"** -- Matches contacts who have the "newsletter" tag.
- **Custom field "company" equals "Acme"** -- Matches contacts whose company field is "Acme."
- **Created after 2026-01-01** -- Matches contacts added after a specific date.

### Editing a Segment

1. Click on a segment in the list.
2. Modify the name or rules.
3. Click "Save." The member count will be recalculated.

### Deleting a Segment

1. Open the segment detail page.
2. Click "Delete" and confirm.

Deleting a segment does not delete the contacts in it. It only removes the segment definition.

---

## Templates

Templates define the content and layout of your emails. The platform supports two editing modes: a code editor for MJML markup and a visual drag-and-drop builder.

### Creating a Template

1. Navigate to Templates.
2. Click "New Template."
3. Choose your starting point:
   - **Blank template** -- Start from scratch.
   - **Starter template** -- Choose from pre-built templates and customize.
4. Enter a template name.
5. Choose your editor:
   - **Code Editor** -- Write MJML markup directly with syntax highlighting.
   - **Visual Builder** -- Use the drag-and-drop Unlayer editor.

### Using the Code Editor

The code editor provides a Monaco-based editing experience with syntax highlighting for MJML. MJML is a markup language that compiles to responsive HTML email.

Write your template using MJML tags. The editor supports real-time preview.

### Using the Visual Builder

The visual builder provides a drag-and-drop interface powered by Unlayer. Drag content blocks (text, images, buttons, dividers) from the sidebar into your email layout. Click any block to edit its content and styling.

### Handlebars Variables

Use Handlebars variables to personalize emails with contact data. Variables are enclosed in double curly braces:

| Variable | Description |
|----------|-------------|
| `{{firstName}}` | Contact's first name |
| `{{lastName}}` | Contact's last name |
| `{{email}}` | Contact's email address |
| `{{tags}}` | Contact's tags |
| `{{customFields.fieldName}}` | Any custom field value |

Example: "Hello {{firstName}}, thank you for subscribing!"

### Previewing a Template

1. Open a template.
2. Click "Preview."
3. The preview renders the template with sample contact data so you can see how variables are replaced.

### Version History

Every time you save a template, a new version is created. To view or restore previous versions:

1. Open the template.
2. Click "History" to see all saved versions with timestamps.
3. Click "Revert" on any version to restore it. This creates a new version with the content from the selected version (non-destructive).

### Duplicating a Template

1. Open the template you want to copy.
2. Click "Duplicate."
3. A new template is created with the same content and a "(Copy)" suffix in the name.

---

## Campaigns

Campaigns are the core of email delivery. A campaign combines a template with an audience (segment) and delivers personalized emails to all matching contacts.

### Creating a Campaign

1. Navigate to Campaigns.
2. Click "New Campaign."
3. Fill in the campaign details:
   - **Name** -- Internal name for this campaign.
   - **Subject** -- Email subject line (supports Handlebars variables).
   - **Template** -- Select the email template to use.
   - **Audience** -- Select a segment to define who receives the email.
4. The audience preview shows how many contacts will receive the email.
5. Click "Save" to create the campaign as a draft.

### Previewing Audience

When creating or editing a campaign, the audience count shows how many contacts match the selected segment. This count is calculated in real time based on current contact data.

### Sending Immediately

1. Open a draft campaign.
2. Review the details (template, audience, subject line).
3. Click "Send Now."
4. Confirm when prompted.

The campaign status changes to "Executing." Emails are queued and sent in the background. You can monitor progress on the campaign detail page.

### Scheduling for Future Delivery

1. Open a draft campaign.
2. Click "Schedule."
3. Select the date and time for delivery (in UTC).
4. Click "Confirm Schedule."

The campaign status changes to "Scheduled." At the scheduled time, the platform automatically begins sending.

### Canceling a Scheduled Campaign

1. Open a scheduled campaign.
2. Click "Cancel Schedule."
3. Confirm the cancellation.

The campaign returns to "Draft" status. You can edit it and reschedule.

### Campaign Statuses

| Status | Description |
|--------|-------------|
| Draft | Campaign created but not yet sent or scheduled |
| Scheduled | Campaign scheduled for future delivery |
| Executing | Campaign is actively sending emails |
| Sent | All emails have been processed |
| Cancelled | Scheduled campaign was cancelled |

---

## Analytics

The Analytics section provides insights into your email campaign performance.

### Per-Campaign Analytics

Open any sent campaign to see its detailed analytics:

- **Delivery Rate** -- Percentage of emails successfully delivered.
- **Open Rate** -- Percentage of delivered emails that were opened. Shows both unique opens and total opens.
- **Click Rate** -- Percentage of delivered emails where a link was clicked. Shows both unique clicks and total clicks.
- **Bounce Rate** -- Percentage of emails that bounced (hard or soft bounce).
- **Complaint Rate** -- Percentage of recipients who reported the email as spam.

### Global Analytics Dashboard

Navigate to Analytics to see aggregated metrics across all campaigns:

- Total campaigns sent
- Total emails delivered
- Overall open rate, click rate, bounce rate, and complaint rate
- Trends over time

Use the date range filter to narrow results to a specific period. Filter by campaign status to see only sent, scheduled, or draft campaigns.

---

## Suppression List

The suppression list contains email addresses that should not receive any future campaigns. Addresses are automatically added when:

- A recipient reports your email as spam (complaint)
- An email hard bounces (invalid address)
- A recipient clicks the unsubscribe link in an email

### Viewing the Suppression List

Navigate to the Suppression page to see all suppressed email addresses with the reason and date they were added.

### Manually Adding an Address

1. Click "Add to Suppression List."
2. Enter the email address.
3. Click "Add."

Use this for contacts who have requested removal through channels outside the email (phone, support ticket, etc.).

### Removing an Address

1. Find the address in the suppression list.
2. Click "Remove."
3. Confirm the removal.

After removal, the address will be eligible to receive campaigns again. Use this only when you are certain the person wants to receive emails (e.g., they re-subscribed).

---

## Settings

### User Management (Admin Only)

Administrators can manage team members who have access to the dashboard.

**Creating a user:**

1. Go to Settings > Users.
2. Click "Add User."
3. Enter the user's email, name, and assign a role (viewer, editor, or admin).
4. Click "Create."

**Inviting a user:**

1. Go to Settings > Users.
2. Click "Invite User."
3. Enter the email address and select a role.
4. Click "Send Invitation."

The user receives an email with a link to set their password and activate their account.

**Roles:**

| Role | Can View | Can Create/Edit | Can Manage Settings |
|------|----------|-----------------|-------------------|
| Viewer | Contacts, templates, campaigns, analytics | No | No |
| Editor | Everything | Contacts, templates, campaigns | No |
| Admin | Everything | Everything | Users, domains, integrations |

**Disabling a user:**

1. Open the user's row in the user list.
2. Click "Disable."
3. The user can no longer log in but their account data is preserved.

### Domain Settings (Admin Only)

The Domain settings page shows your sending domain's verification status.

**Verification status:**
- **Verified** -- Your domain is ready to send emails.
- **Pending** -- DNS records have been submitted but not yet verified. Ensure your DKIM CNAME records and SPF TXT record are correctly configured.
- **Failed** -- Verification failed. Check your DNS records and try again.

The page displays the exact DNS records you need to add. Copy them directly into your domain's DNS management panel.

**Sending reputation:**
The domain page also shows your SES sending reputation score, bounce rate, and complaint rate. Keep your bounce rate below 5% and complaint rate below 0.1% to maintain good deliverability.

### Integrations (Admin Only)

The Integrations settings page allows you to connect external CRM systems for contact synchronization.

**Connecting HubSpot:**

1. Go to Settings > Integrations.
2. In the HubSpot section, click "Connect."
3. Enter your HubSpot Private App token. To create one:
   - Log in to HubSpot.
   - Go to Settings > Integrations > Private Apps.
   - Create a new app with "Contacts" read/write scope.
   - Copy the access token.
4. Click "Save." The platform verifies the token before saving.

**Connecting Zoho CRM:**

1. Go to Settings > Integrations.
2. In the Zoho CRM section, click "Connect."
3. Enter your Zoho refresh token and select your data center region:
   - **US** -- zohoapis.com / accounts.zoho.com
   - **EU** -- zohoapis.eu / accounts.zoho.eu
   - **IN** -- zohoapis.in / accounts.zoho.in
   - **AU** -- zohoapis.com.au / accounts.zoho.com.au
   - **CN** -- zohoapis.com.cn / accounts.zoho.com.cn
   - **JP** -- zohoapis.jp / accounts.zoho.jp
4. Click "Save."

**Syncing contacts:**

After connecting a CRM:

1. Click "Sync" on the integration card.
2. The sync pulls new contacts from the CRM into the platform and pushes platform contacts to the CRM.
3. A summary shows how many contacts were pulled, pushed, and any errors.

Sync deduplicates by email address. Contacts that exist in both systems are matched and updated rather than duplicated.

**Disconnecting:**

Click "Disconnect" on any integration card to remove the connection. This does not delete any contacts that were previously synced.

---

## Branding

The dashboard appearance can be customized per deployment through environment variables. Branding is configured at deploy time, not through the dashboard UI.

**What can be customized:**

| Setting | Variable | Effect |
|---------|----------|--------|
| Application name | `VITE_APP_NAME` | Displayed in the sidebar header and browser tab title |
| Logo | `VITE_LOGO_URL` | Displayed in the sidebar header. If empty, the app name is shown as text. |
| Primary color | `VITE_PRIMARY_COLOR` | Accent color used for buttons, links, and highlights throughout the UI |

To change branding, update the corresponding variables in your `.env` file and redeploy the frontend:

```bash
bash scripts/deploy-frontend.sh
```

Transactional emails (verification, password reset, invitation) use separate branding variables (`APP_NAME`, `LOGO_URL`, `PRIMARY_COLOR`) that are configured on the backend Lambda functions. These require a full redeployment to update:

```bash
bash scripts/deploy.sh
```
