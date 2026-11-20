---
name: productecho-agents
description: >-
  Use when packaging, building, configuring, deploying, invoking, or managing autonomous AI agent workloads
  and MCP sidecars on ProductEcho Cloud. Covers agent deployment endpoints (/api/v1/agents), BYOK LLM credentials,
  resource sizing, daily spend budgets, sidecar tool configuration, interactive playground execution, and hibernation.
---

# ProductEcho Autonomous Agent Deployment & Management Skill

This skill provides comprehensive workflows, architectural guidance, and API specifications for packaging, deploying, executing, and scaling **autonomous AI agent workloads** on ProductEcho Cloud.

> **Prerequisite**: this skill calls the `/api/v1/agents` REST API directly — not MCP tools. It requires a Bearer JWT (`PRODUCT_ECHO_TOKEN`) obtained from the ProductEcho web console, distinct from the MCP OAuth connection the other ProductEcho skills use (see the `productecho-connect` skill for that flow).

ProductEcho Autonomous Agents run as isolated, containerized Kubernetes microservices backed by Model Context Protocol (MCP) sidecars (databases, memory, sandboxes, web groundings), configurable daily token budgets, and native interactive sandboxes.

---

## 🌟 Core Features & Value Proposition

- **Containerized Agent Workloads**: Deploy Python (FastMCP, LangGraph, CrewAI, AutoGen, or custom ASGI) agents directly from GitHub or source zip archives.
- **Built-in MCP Sidecars**: Seamlessly connect agents to managed sidecars including `mcp-postgres`, `mcp-redis`, `mcp-py-exec`, `mcp-web-search`, `mcp-figma`, and external custom MCP servers.
- **BYOK (Bring Your Own Key)**: Securely inject provider keys (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GEMINI_API_KEY`) via encrypted environment variables.
- **Budget & Resource Governance**: Enforce max daily spend limits (`daily_budget_usd`) and configure pod CPU/memory requests and limits.
- **Interactive Sandbox & Playground**: Test and validate agents interactively via `/invoke` endpoints with full execution tracing, tool call auditing, and cost tracking.
- **Smart Cost Hibernation**: Scale idle agent pods to 0 replicas with `pause` and bring them back instantly with `resume`.

---

## 🛠️ REST API Specification (`/api/v1/agents`)

All endpoints require Bearer JWT authentication (`Authorization: Bearer <access_token>`). Tenant identity is securely inferred from the JWT context.

| Endpoint | Method | Description | Request / Query | Response Model |
| :--- | :--- | :--- | :--- | :--- |
| `/api/v1/agents` | `GET` | List all tenant agent workloads | `?status=<status>` | `list[AgentResourceSummary]` |
| `/api/v1/agents/presigned-upload-url` | `POST` | Generate an S3 upload form for the source zip, with the workspace's size limit | `{"agent_name": "..."}` | `AgentSourceUploadResponse` |
| `/api/v1/agents/deploy` | `POST` | Trigger agent build, sidecar wiring, and K8s rollout | `AgentDeployRequest` | `AgentDeploymentState` (HTTP 202) |
| `/api/v1/agents/{agent_name}/status` | `GET` | Query agent build, container status, and MCP sidecars | Path: `agent_name` | `AgentDeploymentState` |
| `/api/v1/agents/{agent_name}/invoke` | `POST` | Execute interactive prompt in agent sandbox | `AgentInvokeRequest` | `AgentInvokeResponse` |
| `/api/v1/agents/{agent_name}/pause` | `POST` | Hibernate agent pod (scale replicas to 0) | Path: `agent_name` | `AgentDeploymentState` |
| `/api/v1/agents/{agent_name}/resume` | `POST` | Wake hibernated agent pod (scale replicas to 1) | Path: `agent_name` | `AgentDeploymentState` |
| `/api/v1/agents/{agent_name}` | `DELETE` | Delete agent workload, service, ingress, and ECR repo | Path: `agent_name` | `AgentDeleteResponse` |

---

## 📦 Packaging Guidelines for Autonomous Agents

1. **Root Manifests**: Ensure the project contains a standard Python manifest:
   - `requirements.txt` declaring dependencies (e.g. `fastapi`, `uvicorn`, `langgraph`, `anthropic`, `openai`, `mcp`).
   - `main.py` exposing the application entrypoint (e.g. `app`).
2. **Entrypoint Convention**: Default entrypoint is `main.py:app`. If using a custom file or object, specify it during deploy (e.g. `server.py:mcp_app`).
3. **Port Binding**: Agent services must listen on `0.0.0.0` at the port specified in `container_port` (default `8080`, read via `PORT` environment variable).
4. **Archive Cleanliness**: Exclude `.git/`, `.venv/`, `__pycache__/`, `.env*`, and local caches from the zip upload:
   ```bash
   zip -r agent_source.zip . -x ".git/*" ".venv/*" "__pycache__/*" "*.pyc" "*.env*"
   ```

---

## 🚀 Standard Agent Deployment Workflows

### 1. Uploading Agent Source Code

1. Request an upload form:
   ```bash
   curl -X POST "https://api.productecho.com/api/v1/agents/presigned-upload-url" \
     -H "Authorization: Bearer $PRODUCT_ECHO_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"agent_name": "customer-support-agent"}'
   ```
2. Check that the zip is no larger than `max_size_bytes` from step 1 (250 MB unless ProductEcho raised the workspace's limit; to raise it, contact https://www.productecho.com/contact). Then upload it to S3: every `upload_fields` entry as its own form field, then the zip as the last field, named `file`:
   ```bash
   curl -X POST "$UPLOAD_URL" \
     --form-string "key=<upload_fields.key>" \
     --form-string "policy=<upload_fields.policy>" \
     --form-string "<other field>=<value>" \
     -F "file=@agent_source.zip"
   ```
   A `204` response means the upload worked; a `400` containing `EntityTooLarge` means the zip is over the limit.
3. Pass the `s3_key` from the step 1 response to the deploy call below.

---

### 2. Deploying the Agent with MCP Sidecars & BYOK Keys

Call `POST /api/v1/agents/deploy`:

```json
{
  "agent_name": "customer-support-agent",
  "source_type": "zip",
  "s3_key": "builds/tenant_xyz/customer-support-agent/upl-8f4b2.zip",
  "entrypoint": "main.py:app",
  "container_port": 8080,
  "daily_budget_usd": 25.0,
  "cpu_request": "250m",
  "memory_request": "512Mi",
  "cpu_limit": "1000m",
  "memory_limit": "1Gi",
  "system_prompt": "You are a professional customer support assistant for ProductEcho Cloud.",
  "env_vars": {
    "OPENAI_API_KEY": "sk-proj-...",
    "ANTHROPIC_API_KEY": "sk-ant-...",
    "LOG_LEVEL": "info"
  },
  "mcp_tools": [
    {
      "name": "mcp-postgres",
      "display_name": "PostgreSQL DB",
      "description": "Read and query customer records",
      "category": "database",
      "enabled": true,
      "status": "active",
      "env_vars": {
        "DATABASE_URL": "postgresql://pguser:secret@postgres-x7k9p2.service.internal:5432/app_production"
      }
    },
    {
      "name": "mcp-web-search",
      "display_name": "Google Search",
      "description": "Ground customer inquiries with current documentation",
      "category": "search",
      "enabled": true,
      "status": "active"
    }
  ],
  "is_public": true
}
```

---

### 3. Polling Deployment Status

Call `GET /api/v1/agents/customer-support-agent/status`:

```json
{
  "deployment_id": "dep-a1b2c3d4",
  "tenant_id": "tenant_xyz",
  "agent_name": "customer-support-agent",
  "status": "READY",
  "status_reason": "Agent container deployed and ingress routing active.",
  "domain_url": "https://customer-support-agent.tenant-xyz.productecho.app",
  "container_port": 8080,
  "mcp_tools": [
    {
      "name": "mcp-postgres",
      "display_name": "PostgreSQL DB",
      "status": "active",
      "enabled": true
    }
  ]
}
```

Wait until `status` transitions to `READY` (live domain URL available) or `FAILED` / `BUILD_FAILED`.

---

### 4. Interactive Testing in Agent Playground (`/invoke`)

Execute prompts against the live agent sandbox to test tool calling and reasoning:

```bash
curl -X POST "https://api.productecho.com/api/v1/agents/customer-support-agent/invoke" \
  -H "Authorization: Bearer $PRODUCT_ECHO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "Find all open support tickets created in the last 24 hours.",
    "session_id": "session-test-01"
  }'
```

Response:
```json
{
  "agent_name": "customer-support-agent",
  "session_id": "session-test-01",
  "response": "Found 3 open tickets created in the last 24 hours. Here is the summary...",
  "tool_calls": [
    {
      "tool": "mcp-postgres",
      "action": "query",
      "query": "SELECT * FROM tickets WHERE created_at >= NOW() - INTERVAL '24 hours';"
    }
  ],
  "tokens_used": 1420,
  "cost_usd": 0.0042
}
```

---

### 5. Cost-Saving Hibernation & Teardown

- **Pause Agent**: Scales agent pod to 0 replicas when inactive (e.g. dev/staging):
  ```bash
  curl -X POST "https://api.productecho.com/api/v1/agents/customer-support-agent/pause" \
    -H "Authorization: Bearer $PRODUCT_ECHO_TOKEN"
  ```
- **Resume Agent**: Brings agent pod back online to 1 replica:
  ```bash
  curl -X POST "https://api.productecho.com/api/v1/agents/customer-support-agent/resume" \
    -H "Authorization: Bearer $PRODUCT_ECHO_TOKEN"
  ```
- **Delete Agent**: Destroys Deployment, Service, Ingress, and ECR repository:
  ```bash
  curl -X DELETE "https://api.productecho.com/api/v1/agents/customer-support-agent" \
    -H "Authorization: Bearer $PRODUCT_ECHO_TOKEN"
  ```

---

## 📚 Supporting References

- [Standard MCP Sidecars & Custom Integrations](./references/agent-mcp-sidecars.md)
