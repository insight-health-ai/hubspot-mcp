# HubSpot MCP Server

A [Model Context Protocol (MCP)](https://modelcontextprotocol.io/introduction) server for the [HubSpot](https://hubspot.com/) API, built for AI agents to safely read and write CRM data. Forked from [@shinzolabs/hubspot-mcp](https://github.com/shinzo-labs/hubspot-mcp) with safety guardrails and composite workflow tools.

## Features

- Complete coverage of the HubSpot CRM API (112+ tools)
- **Safety guardrails** for AI agent usage:
  - `dryRun` parameter on all destructive tools (preview before executing)
  - Batch operations capped at 10 items per call
  - Destructive tool descriptions warn agents to confirm with users
  - HTTP transport disabled by default (opt-in via `ENABLE_HTTP=true`)
  - Telemetry disabled by default (opt-in via `TELEMETRY_ENABLED=true`)
  - Tightened schema validation (`.catchall(z.string())`)
- **Composite workflow tools** for common multi-step operations:
  - `workflow_onboard_client` — create company + contact + association + deal in one call
  - `workflow_update_deal_value` — fetch current deal, show diff, then update
  - `workflow_link_contact_to_company` — verify both records, then associate
- Support for all standard CRM objects (companies, contacts, deals, etc.)
- Advanced association management with CRM Associations v4
- Batch operations for efficient data management
- Advanced search and filtering capabilities
- Type-safe parameter validation with [Zod](https://zod.dev/)

## Prerequisites

If you don't have an API key, follow the steps [here](https://developers.hubspot.com/docs/guides/api/overview) to obtain an access token. OAuth support is planned as a future enhancement.

## Setup

### 1. Get a HubSpot Access Token

If you don't already have one, create a [HubSpot private app](https://developers.hubspot.com/docs/guides/api/overview) and copy the access token. Grant it scopes for the CRM objects you need (contacts, companies, deals, etc.).

### 2. Build from Source

```bash
git clone https://github.com/insight-health-ai/hubspot-mcp.git
cd hubspot-mcp
pnpm install
pnpm build
```

### 3. Configure Your MCP Client

Add the following to your MCP client config:

**Cursor** (`.cursor/mcp.json`):
```json
{
  "mcpServers": {
    "hubspot": {
      "command": "node",
      "args": ["/path/to/hubspot-mcp/dist/index.js"],
      "env": {
        "HUBSPOT_ACCESS_TOKEN": "your-access-token-here"
      }
    }
  }
}
```

**Claude Desktop** (`~/Library/Application Support/Claude/claude_desktop_config.json`):
```json
{
  "mcpServers": {
    "hubspot": {
      "command": "node",
      "args": ["/path/to/hubspot-mcp/dist/index.js"],
      "env": {
        "HUBSPOT_ACCESS_TOKEN": "your-access-token-here"
      }
    }
  }
}
```

## Config Variables

| Variable               | Description                                             | Required | Default |
|------------------------|---------------------------------------------------------|----------|---------|
| `HUBSPOT_ACCESS_TOKEN` | Access token for your HubSpot private app               | Yes      |         |
| `TELEMETRY_ENABLED`    | Send anonymous telemetry to Shinzo Labs                 | No       | `false` |
| `ENABLE_HTTP`          | Start the Streamable HTTP transport (exposes port)      | No       | `false` |
| `PORT`                 | Port for HTTP transport (only used if `ENABLE_HTTP=true`) | No       | `3000`  |

## Safety Guardrails

All destructive operations (archive, delete, unsubscribe) include built-in safety mechanisms:

- **`dryRun` parameter** — pass `dryRun: true` to any destructive tool to preview what would happen without executing. Returns a summary of the action that would be taken.
- **Batch caps** — batch archive operations are limited to 10 items per call to prevent accidental mass deletion.
- **Description warnings** — all destructive tools are prefixed with `DESTRUCTIVE:` in their descriptions, instructing AI agents to confirm with the user before executing.
- **No silent HTTP** — the HTTP transport only starts if you explicitly set `ENABLE_HTTP=true`. By default, only the stdio transport is active.

### dryRun Example

```json
{
  "tool": "crm_archive_object",
  "params": {
    "objectType": "contacts",
    "objectId": "12345",
    "dryRun": true
  }
}
```

Returns:
```json
{
  "dryRun": true,
  "action": "archive",
  "objectType": "contacts",
  "objectId": "12345",
  "message": "Would archive contacts object 12345"
}
```

## Workflow Tools

Composite tools that handle multi-step CRM operations in a single call:

### `workflow_onboard_client`

Creates a company, contact, associates them, and optionally creates a deal — all in one call.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `company.name` | Yes | Company name |
| `company.domain` | No | Company domain |
| `company.industry` | No | Industry |
| `contact.email` | Yes | Contact email |
| `contact.firstname` | Yes | First name |
| `contact.lastname` | Yes | Last name |
| `deal.dealname` | No | Deal name (creates deal if provided) |
| `deal.amount` | No | Deal amount |
| `deal.dealstage` | No | Deal stage ID |
| `dryRun` | No | Preview without executing |

### `workflow_update_deal_value`

Fetches the current deal, shows a diff of proposed changes, then applies the update.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `dealId` | Yes | Deal ID to update |
| `amount` | No | New deal amount |
| `dealstage` | No | New deal stage ID |
| `dealname` | No | New deal name |
| `closedate` | No | New expected close date (ISO format) |
| `dryRun` | No | Preview changes without applying |

### `workflow_link_contact_to_company`

Verifies both the contact and company exist, then creates the association between them.

| Parameter | Required | Description |
|-----------|----------|-------------|
| `contactId` | Yes | Contact ID |
| `companyId` | Yes | Company ID |
| `dryRun` | No | Preview without executing |

## Supported Tools

### Core CRM Objects

  - `crm_list_objects`: List CRM objects with optional filtering and pagination
  - `crm_get_object`: Get a single CRM object by ID
  - `crm_create_object`: Create a new CRM object
  - `crm_update_object`: Update an existing CRM object
  - `crm_archive_object`: Archive (delete) a CRM object
  - `crm_search_objects`: Search CRM objects using advanced filters
  - `crm_batch_create_objects`: Create multiple objects in a single request
  - `crm_batch_read_objects`: Read multipl objects in a single request
  - `crm_batch_update_objects`: Update multiple objects in a single request
  - `crm_batch_archive_objects`: Archive (delete) multiple objects in a single request

### Companies

  - `crm_create_company`: Create a new company with validated properties
  - `crm_update_company`: Update an existing company
  - `crm_get_company`: Get a single company by ID
  - `crm_search_companies`: Search companies with specific filters
  - `crm_batch_create_companies`: Create multiple companies in a single request
  - `crm_batch_update_companies`: Update multiple companies in a single request
  - `crm_get_company_properties`: Get all available company properties
  - `crm_create_company_property`: Create a new company property

### Contacts

  - `crm_create_contact`: Create a new contact with validated properties
  - `crm_update_contact`: Update an existing contact's information
  - `crm_get_contact`: Get a single contact by ID
  - `crm_search_contacts`: Search contacts with specific filters
  - `crm_batch_create_contacts`: Create multiple contacts in a single request
  - `crm_batch_update_contacts`: Update multiple contacts in a single request
  - `crm_get_contact_properties`: Get all available contact properties
  - `crm_create_contact_property`: Create a new contact property

### Leads

  - `crm_create_lead`: Create a new lead with validated properties
  - `crm_update_lead`: Update an existing lead's information
  - `crm_get_lead`: Get a single lead by ID
  - `crm_search_leads`: Search leads with specific filters
  - `crm_batch_create_leads`: Create multiple leads in a single request
  - `crm_batch_update_leads`: Update multiple leads in a single request
  - `crm_get_lead_properties`: Get all available lead properties
  - `crm_create_lead_property`: Create a new lead property

### Engagement Management

  - `engagement_details_get`: Get details of a specific engagement
  - `engagement_details_create`: Create a new engagement
  - `engagement_details_update`: Update an existing engagement
  - `engagement_details_archive`: Archive (delete) an engagement
  - `engagement_details_list`: List all engagements with filtering
  - `engagement_details_get_associated`: Get associated engagements

### Calls

  - `calls_create`: Create a new call record
  - `calls_get`: Get call details
  - `calls_update`: Update a call record
  - `calls_archive`: Archive a call
  - `calls_list`: List all calls
  - `calls_search`: Search calls
  - `calls_batch_create`: Create multiple calls
  - `calls_batch_read`: Read multiple calls
  - `calls_batch_update`: Update multiple calls
  - `calls_batch_archive`: Archive multiple calls

### Emails

  - `emails_create`: Create a new email record
  - `emails_get`: Get email details
  - `emails_update`: Update an email
  - `emails_archive`: Archive an email
  - `emails_list`: List all emails
  - `emails_search`: Search emails
  - `emails_batch_create`: Create multiple emails
  - `emails_batch_read`: Read multiple emails
  - `emails_batch_update`: Update multiple emails
  - `emails_batch_archive`: Archive multiple emails

### Meetings

  - `meetings_create`: Create a new meeting
  - `meetings_get`: Get meeting details
  - `meetings_update`: Update a meeting
  - `meetings_archive`: Archive (delete) a meeting
  - `meetings_list`: List all meetings
  - `meetings_search`: Search meetings
  - `meetings_batch_create`: Create multiple meetings
  - `meetings_batch_update`: Update multiple meetings
  - `meetings_batch_archive`: Archive multiple meetings

### Notes

  - `notes_create`: Create a new note
  - `notes_get`: Get note details
  - `notes_update`: Update a note
  - `notes_archive`: Archive a note
  - `notes_list`: List all notes
  - `notes_search`: Search notes
  - `notes_batch_create`: Create multiple notes
  - `notes_batch_read`: Read multiple notes
  - `notes_batch_update`: Update multiple notes
  - `notes_batch_archive`: Archive multiple notes

### Tasks

  - `tasks_create`: Create a new task
  - `tasks_get`: Get task details
  - `tasks_update`: Update a task
  - `tasks_archive`: Archive a task
  - `tasks_list`: List all tasks
  - `tasks_search`: Search tasks
  - `tasks_batch_create`: Create multiple tasks
  - `tasks_batch_read`: Read multiple tasks
  - `tasks_batch_update`: Update multiple tasks
  - `tasks_batch_archive`: Archive multiple tasks

### Associations and Relationships

  - `crm_list_association_types`: List available association types
  - `crm_get_associations`: Get all associations between objects
  - `crm_create_association`: Create an association
  - `crm_archive_association`: Archive (delete) an association
  - `crm_batch_create_associations`: Create multiple associations
  - `crm_batch_archive_associations`: Archive (delete) multiple associations

### Communication Preferences

  - `communications_get_preferences`: Get contact preferences
  - `communications_update_preferences`: Update contact preferences
  - `communications_unsubscribe_contact`: Global unsubscribe
  - `communications_subscribe_contact`: Global subscribe
  - `communications_get_subscription_definitions`: Get subscription definitions
  - `communications_get_subscription_status`: Get status for multiple contacts
  - `communications_update_subscription_status`: Update status for multiple contacts

### Products

  - `products_create`: Create a product with the given properties and return a copy of the object, including the ID.
  - `products_read`: Read an Object identified by ID
  - `products_update`: Perform a partial update of an Object identified by ID. Read-only and non-existent properties will result in an error. Properties values can be cleared by passing an empty string.
  - `products_archive`: Move an Object identified by ID to the recycling bin.
  - `products_list`: Read a page of products. Control what is returned via the `properties` query param. `after` is the paging cursor token of the last successfully read resource will be returned as the `paging.next.after` JSON property of a paged response containing more results.
  - `products_search`: Search products
  - `products_batch_create`: Create a batch of products
  - `products_batch_read`: Read a batch of products by internal ID, or unique property values. Retrieve records by the `idProperty` parameter to retrieve records by a custom unique value property.
  - `products_batch_update`: Update a batch of products by internal ID, or unique values specified by the `idProperty` query param.
  - `products_batch_archive`: Archive a batch of products by ID

## Data Collection and Privacy

Telemetry is **disabled by default**. If enabled via `TELEMETRY_ENABLED=true`, limited anonymous telemetry is sent to Shinzo Labs (the upstream project maintainer). No PII, IP addresses, or tool arguments are collected. See [PRIVACY.md](./PRIVACY.md) for details.

## Upstream

Forked from [@shinzolabs/hubspot-mcp](https://github.com/shinzo-labs/hubspot-mcp). Original project by [Austin Born](https://github.com/shinzo-labs).

## License

MIT
