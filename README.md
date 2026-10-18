# ProductEcho plugin

Deploy apps and managed PostgreSQL from your AI agent and get a live HTTPS URL.

[ProductEcho](https://www.productecho.com) is an AI-native app runtime. This plugin connects Claude Code, Cursor or Codex to the ProductEcho MCP server and gives your agent four skills, so you can say "Deploy this application with PostgreSQL" and the agent provisions the database, sets the environment variables, deploys the code and returns the public URL. ProductEcho does not include an AI model; you bring your own agent. There is a free plan and no credit card is needed to start.

The MCP server runs at `https://api.productecho.com/mcp`. This repository holds the client side: the server connection and the skills that teach your agent how to use it.

---

## Install

### Claude Code

```bash
claude plugin marketplace add productecho/skills
claude plugin install productecho-plugin@productecho
```

Then run `/reload-plugins`. When your agent first calls a ProductEcho tool, a browser window opens so you can sign in and approve access. There is no API key to paste. Step by step: [Deploy from Claude Code](https://www.productecho.com/blog/integrations/claude).

### Cursor

Install the plugin from Cursor Directory: [cursor.directory/plugins/productecho-plugin](https://cursor.directory/plugins/productecho-plugin).

Or set it up by hand. Install the skills:

```bash
npx -y skills add productecho/skills --skill '*' --yes --global --agent cursor
```

Then add the server to `~/.cursor/mcp.json` (every project) or `.cursor/mcp.json` (one project):

```json
{
  "mcpServers": {
    "productecho": {
      "url": "https://api.productecho.com/mcp"
    }
  }
}
```

Cursor starts the sign-in the first time a ProductEcho tool is used. Step by step: [Deploy from Cursor](https://www.productecho.com/blog/integrations/cursor).

### Codex

```bash
npx -y skills add productecho/skills --skill '*' --yes --global --agent codex
codex mcp add productecho --url https://api.productecho.com/mcp
codex mcp login productecho
```

Step by step: [Deploy from Codex](https://www.productecho.com/blog/integrations/codex).

### Other MCP clients

Any client that supports remote HTTP MCP servers can use `https://api.productecho.com/mcp`; the sign-in starts on first use. Setup for each client, the stdio bridge and headless use: [Connect an MCP client](https://www.productecho.com/blog/connect-mcp-client).

---

## What your agent can do

- **Inspect a project** before deploying: framework, port, subprojects and the settings that need confirming.
- **Deploy applications** from an uploaded project archive or a connected GitHub repository, to containers or, for single-page apps built with Vite, React, Vue, Svelte or Angular, to a CDN. You get a public HTTPS URL back.
- **Manage PostgreSQL 17 and 18**: create, list, check status, grow storage without taking the database offline, pause, resume, delete, and read connection details.
- **Change settings**: update environment variables and the start command, then restart without a rebuild.
- **Pause, resume or delete** applications.
- **Add a custom domain** to a static app: publish the two DNS records the tool returns, then check until it is active.
- **See your workspace**: list applications and databases, and the resource sizes your plan allows.
- **Ask for help**: send a support question or bug report to the ProductEcho team.

Tools that delete, pause, deploy or change settings carry the MCP destructive hint, so clients that honour it ask before running them.

---

## Bundled skills

1. **`productecho-deploy`**
   - Source inspection and framework configuration.
   - Readiness checks for Node.js, Python, Go and Java projects (dependency separation, port binding, process declarations).
   - Clean source archive and upload.
   - Build and deployment status, including the reason when a deploy fails, and service lifecycle (pause, resume, delete).

2. **`productecho-postgres`**
   - Managed PostgreSQL provisioning.
   - Credential retrieval and connection strings (`DATABASE_URL`).
   - Storage growth, pause and resume.

3. **`productecho-connect`**
   - Connecting the MCP server and signing in.
   - Support requests.

4. **`productecho-agents`**
   - Autonomous AI agent workload packaging and rollout.
   - Built-in MCP sidecars (`mcp-postgres`, `mcp-redis`, `mcp-py-exec`, `mcp-web-search`).
   - Spend budgets, interactive sandbox invocation, and hibernation.

---

## What the plugin runs, sends and fetches

- **Server connection.** `.mcp.json` points your client at `https://api.productecho.com/mcp`. Every tool call goes there, authenticated with OAuth.
- **Your source code, only when you ask for a deploy from local files.** The deploy skill tells your agent to build a zip of the project, leaving out `.git`, `node_modules`, `.next`, `.venv`, `.env` and build caches, and to send it with an HTTP `PUT` to a presigned upload URL that the MCP server returns, which points at ProductEcho's storage. Deploying from a connected GitHub repository sends no archive.
- **Local helper scripts.** `skills/productecho-deploy/scripts/verify-packaging.sh` reads project files such as `package.json`, `requirements.txt` and `go.mod` and prints a readiness report. `skills/productecho-deploy/scripts/state-manager.py` writes `.productecho/state.json` (application name, database identifier, root directory, repository and update time), adds `.productecho/` to `.gitignore`, and runs `git config --get remote.origin.url` to find the repository. Neither script makes network calls.
- **Nothing else.** The plugin declares no hooks and starts no background processes.
- **Optional stdio bridge.** `mcp_config.json` is a fallback for clients without native HTTP MCP support. Only if you use it, `npx` downloads and starts `mcp-remote`, which connects to the same URL. The default `.mcp.json` setup downloads and runs nothing.

---

## Sign-in and permissions

Interactive clients use OAuth 2.1 with PKCE. The server answers the first request with `401` and discovery metadata (`/.well-known/oauth-protected-resource`), your client opens the ProductEcho consent screen, and a short-lived access token is issued and renewed for you. Access is limited to your own workspace.

The default scope is `mcp:fullwrite`, so one consent covers a complete full-stack workflow: workspace and application access, PostgreSQL management and database credentials. `mcp:fullread` gives read-only access. Support messages use the separate `mcp:support:write` scope.

### Configuration files

`.mcp.json` (native HTTP, used by Claude Code, Cursor, Codex and other MCP-native clients):

```json
{
  "mcpServers": {
    "productecho": {
      "type": "http",
      "url": "https://api.productecho.com/mcp"
    }
  }
}
```

`mcp_config.json` (stdio bridge, for clients without native HTTP MCP support):

```json
{
  "mcpServers": {
    "productecho": {
      "command": "npx",
      "args": [
        "-y",
        "mcp-remote",
        "https://api.productecho.com/mcp"
      ]
    }
  }
}
```

### API key (headless pipelines and scripts only)

OAuth is the recommended setup. For CI or scripts, a tenant API key can be sent in a header instead:

```bash
export PRODUCT_ECHO_API_KEY="pe_live_your_token_here"
```

Header format: `PRODUCT-ECHO-API-KEY: pe_live_...` or `Authorization: Bearer pe_live_...`.

---

## Sample prompts

- *"Deploy this application with PostgreSQL."*
- *"Show my ProductEcho workspace and what is running."*
- *"Provision a managed PostgreSQL database and give me the connection URL."*
- *"Expand my database storage to 20 GB."*
- *"Put this static site on my own domain."*
- *"Send ProductEcho support a message about my failed deploy."*

---

## Package layout

```
productecho-plugin/
├── .claude-plugin/
│   ├── marketplace.json     # Claude Code marketplace catalog
│   └── plugin.json          # Claude Code plugin manifest
├── .codex-plugin/
│   └── plugin.json          # Codex plugin manifest
├── plugin.json              # Universal root manifest
├── .mcp.json                # Native MCP HTTP config (used by all three manifests)
├── mcp_config.json          # Optional stdio-bridge config (mcp-remote)
├── .app.json                # Registered MCP application mapping
├── assets/
│   ├── icon.png             # Plugin composer icon
│   └── logo.png             # Plugin brand logo
├── rules/
│   └── AGENTS.md            # Multi-language deployment guardrails
├── skills/
│   ├── productecho-deploy/  # Application deployment and packaging
│   ├── productecho-postgres/# Managed PostgreSQL
│   ├── productecho-connect/ # Connection and support
│   └── productecho-agents/  # Autonomous agent workloads and MCP sidecars
├── LICENSE                  # Apache-2.0
├── .gitignore
└── README.md
```

---

## Links

- Website: [www.productecho.com](https://www.productecho.com)
- Guides: [Quickstart](https://www.productecho.com/blog/quickstart), [Connect an MCP client](https://www.productecho.com/blog/connect-mcp-client), [Deploy with an AI agent](https://www.productecho.com/blog/deploying-with-mcp)
- 32-second demo: [Claude Code to a live app](https://www.productecho.com/watch/claude-code-to-live-app)
- Sign up: [app.productecho.com/signup](https://app.productecho.com/signup)
- Support and security contact: [www.productecho.com/contact](https://www.productecho.com/contact)
- [Privacy policy](https://www.productecho.com/privacy) and [terms](https://www.productecho.com/terms)

## License

Apache-2.0. See [LICENSE](LICENSE).
