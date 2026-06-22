# Agentforce Today Remodel

A modern React + Express application that delivers AI-powered sales briefings using Salesforce data. Built with the Model Context Protocol (MCP) for secure Salesforce connectivity and integrated with AI models for intelligent insights.

![Dashboard Preview](docs/screenshots/dashboard.png)

## Features

- **OAuth 2.0 PKCE Authentication** — Secure browser-based login with Salesforce
- **Real-Time Dashboard** — KPIs, charts, and metrics from your Salesforce org
- **AI Sales Briefings** — Natural language summaries of opportunities, leads, and pipeline
- **Interactive Chat** — Ask questions about your data, request updates
- **Record Updates** — AI can update Salesforce records via the Models API
- **MCP Integration** — Uses Salesforce's Model Context Protocol for data access
- **Dual LLM Support** — Switch between Anthropic Claude and Salesforce Models API

## Screenshots

| Dashboard | AI Chat | Charts |
|-----------|---------|--------|
| ![Dashboard](docs/screenshots/dashboard-thumb.png) | ![Chat](docs/screenshots/chat-thumb.png) | ![Charts](docs/screenshots/charts-thumb.png) |

## Architecture

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│   React App     │ ───▶ │  Express API    │ ───▶ │   Salesforce    │
│   (Vite)        │      │  (Node.js)      │      │   MCP Gateway   │
└─────────────────┘      └────────┬────────┘      └─────────────────┘
                                  │                        │
                                  │                   ECA / JWT
                                  ▼                   (internal)
                         ┌─────────────────┐
                         │  Anthropic /    │
                         │  Models API     │
                         └─────────────────┘
```

### Authentication Flow

This app uses **External Client Apps (ECA)** with OAuth 2.0 PKCE:

| Component | ECA | OAuth Scope | Purpose |
|-----------|-----|-------------|---------|
| **MCP Data Access** | Primary ECA | `mcp_api` | Query Salesforce data via MCP gateway |
| **Models API / Updates** | Secondary ECA | `sfap_api api` | LLM access + record updates |

**Two separate ECAs are required** because different OAuth scopes cannot share the same token without invalidating each other.

> **Note on Observability:** ECA authentication has limited visibility in real-time event monitoring — `LoginEvent` fires, but `ApiEvent` does not fire for MCP-routed queries. Activity is captured in daily EventLogFile CSVs but not in queryable Event Log Objects. See the [Event Monitoring documentation](https://developer.salesforce.com/docs/atlas.en-us.platform_events.meta/platform_events/sforce_api_objects_apievent.htm) for details.

## Prerequisites

- **Node.js** 18+ 
- **Salesforce Org** with API access
- **Two External Client Apps (ECA)** configured for OAuth 2.0 PKCE:
  - Primary ECA with `mcp_api` scope (for MCP data access)
  - Secondary ECA with `sfap_api api` scope (for Models API / record updates)
- **Anthropic API Key** (optional) — for external LLM, or use Salesforce Models API

## Quick Start

### 1. Clone and Install

```bash
git clone https://github.com/gyantsos123/agentforce-today-remodel.git
cd agentforce-today-remodel
npm install
```

### 2. Configure External Client Apps (ECA)

Create **two ECAs** in your Salesforce org — they need separate Client IDs because the scopes differ:

#### Primary ECA (MCP Data Access)

1. **Setup** → **External Client Apps** → **New**
2. Configure OAuth:
   - **Callback URL**: `http://localhost:3335/oauth/callback`
   - **OAuth Scopes**: `mcp_api`, `refresh_token`
3. Enable **PKCE**
4. Copy the **Client ID** → use as `SF_CLIENT_ID`

#### Secondary ECA (Models API + Record Updates)

1. **Setup** → **External Client Apps** → **New**
2. Configure OAuth:
   - **Callback URL**: `http://localhost:3335/oauth/models-callback`
   - **OAuth Scopes**: `sfap_api`, `api`, `refresh_token`
3. Enable **PKCE**
4. Copy the **Client ID** → use as `SF_MODELS_CLIENT_ID`

> **Why two ECAs?** OAuth tokens are scoped. A single token cannot hold both `mcp_api` and `api` scopes without one invalidating the other. Separate ECAs allow both MCP queries and REST API updates to work simultaneously.

### 3. Configure Environment

Copy the example environment file and fill in your values:

```bash
cp .env.example .env
```

Edit `.env`:

```env
# Salesforce MCP Server URL
MCP_SERVER_URL=https://api.salesforce.com/platform/mcp/v1/platform/sobject-all

# Primary ECA — MCP data access (mcp_api scope)
SF_CLIENT_ID=your_primary_eca_client_id

# Secondary ECA — Models API + record updates (sfap_api, api scopes)
SF_MODELS_CLIENT_ID=your_secondary_eca_client_id

# Salesforce URLs
SF_LOGIN_URL=https://your-org.my.salesforce.com
SF_TOKEN_URL=https://login.salesforce.com
CALLBACK_URL=http://localhost:3335/oauth/callback

# Salesforce Models API model name
SF_MODELS_API_MODEL=sfdc_ai__DefaultOpenAIGPT4OmniMini

# Optional: Anthropic API key (used if no Models API)
ANTHROPIC_API_KEY=

# Server port
PORT=3335
```

### 4. Run the Application

```bash
npm run dev
```

This starts both the Express server and Vite dev server concurrently.

Open **http://localhost:3335** in your browser.

## Usage

### Dashboard

After authenticating, the dashboard displays:
- **KPI Tiles** — Total pipeline, close rate, new leads, etc.
- **Pipeline Chart** — Opportunities by stage
- **Revenue Chart** — Revenue by product category
- **Lead Sources** — Distribution of lead origins

### AI Chat

Click the chat tab to interact with your data:

- *"What are my top 5 opportunities by amount?"*
- *"Summarize the deals closing this month"*
- *"Update the stage on Acme Corp to Negotiation"*
- *"Create a follow-up task for the GlobeTech opportunity"*

### Model Selection

Toggle between:
- **Anthropic Claude** — External AI with rich reasoning
- **Salesforce Models API** — Built-in AI with Trust Layer and audit logging

## Project Structure

```
agentforce-today-remodel/
├── server.js              # Express backend (OAuth, MCP, LLM proxy)
├── src/
│   ├── App.jsx            # Main React app
│   ├── App.css            # Styling
│   ├── components/
│   │   ├── AuthGate.jsx       # OAuth login flow
│   │   ├── DashboardPanel.jsx # KPIs and charts
│   │   ├── ForecastChat.jsx   # AI chat interface
│   │   ├── Header.jsx         # Navigation header
│   │   ├── KpiTile.jsx        # Metric display
│   │   └── charts/            # Recharts components
├── force-app/             # Salesforce metadata
│   └── main/default/connectedApps/
├── docs/                  # Documentation
├── .env.example           # Environment template
└── package.json
```

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/oauth/login` | GET | Initiates OAuth 2.0 PKCE flow |
| `/oauth/callback` | GET | OAuth callback handler |
| `/oauth/token` | GET | Returns current access token |
| `/api/today` | GET | Fetches dashboard data via MCP |
| `/api/chat` | POST | AI chat with context |
| `/api/update-record` | POST | Update Salesforce record |
| `/api/create-record` | POST | Create Salesforce record |

## Security Notes

- **Never commit `.env`** — Contains secrets
- **PKCE** is required — No client secret needed
- **Tokens are session-scoped** — Stored in server memory only
- **Models API** uses Einstein Trust Layer for audit

## Troubleshooting

### OAuth Errors

- Verify `SF_CLIENT_ID` matches your Connected App
- Ensure callback URL is exactly `http://localhost:3335/oauth/callback`
- Check that PKCE is enabled on the Connected App

### MCP Connection Issues

- Confirm your user has API access
- Verify the MCP gateway URL is correct for your org
- Check that the Connected App has required OAuth scopes

### AI Chat Not Responding

- Verify `ANTHROPIC_API_KEY` is valid
- For Models API, ensure the app has `sfap_api` scope

## License

MIT

## Contributing

Pull requests welcome! Please open an issue first to discuss proposed changes.

---

Built with Salesforce MCP, React, and a lot of coffee.
