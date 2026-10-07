# ProductEcho MCP Tool Quick Reference

This document provides a concise reference for all ProductEcho Model Context Protocol (MCP) tools.

---

## 1. Application Management Tools

| Tool | Purpose & Value | Primary Arguments |
| :--- | :--- | :--- |
| `inspect_application_source` | Inspect source tree and return `recommended_deployment_target` (`container` or `static_cdn`) with build facts. For monorepos it also returns every project with `role`, `deployable` and `requires`, plus `requires_project_selection` and `workspace_tool`. It also returns `settings` (start command, build command, output directory and, for Maven and Gradle projects, the Java version, each detected, suggested or missing) and `blocking_settings` that must be confirmed before deploying. | `application_name`, `s3_key`, `upload_id`, `git_url`, `git_branch`, `root_directory`, `env_vars` |
| `deploy_application` | Build and deploy using `auto`, `container`, or an inspection-approved `static_cdn` target. Returns a presigned S3 upload URL if source is omitted. Refused with the suggested values until `blocking_settings` are provided in `settings`. | `application_name`, `deployment_target`, `s3_key`, `upload_id`, `git_url`, `container_port`, `env_vars`, `settings` |
| `list_applications` | List active application deployments for authenticated tenant. | `deployment_status` (optional filter) |
| `get_application_status` | Query live status, deployment logs, and public HTTPS domain endpoint. | `application_name` |
| `pause_application` | Scale a container to zero or remove a static app's CloudFront KVS route. | `application_name` |
| `resume_application` | Resume a container or restore and health-check a static app's active KVS route. | `application_name` |
| `delete_application` | Remove the public route first, then deprovision target-specific resources and artifacts. | `application_name` |
| `link_project` | Bind a local codebase/monorepo to a ProductEcho application and optional database identity. | `application_name`, `root_directory`, `db_identifier`, `remote_repo` |
| `get_project_link` | Retrieve cloud deployment state and project identity for an application or repository (use during Step 0 state discovery). | `application_name`, `remote_repo`, `root_directory` |
| `get_project_state` | Deprecated alias for `get_project_link` — retained for backward compatibility. | `application_name`, `remote_repo`, `root_directory` |
| `get_application_env` | Retrieve configured environment variables for a deployed application. | `application_name` |
| `update_application_env` | Update environment variables and optionally trigger a rolling restart without rebuilding. | `application_name`, `env_vars`, `redeploy` |
| `update_application_settings` | Change the start command of a buildpack-built container and roll it out without rebuilding; `null` removes the stored value. | `application_name`, `settings`, `redeploy` |

---

## 2. PostgreSQL Database Management Tools

| Tool | Purpose & Value | Primary Arguments |
| :--- | :--- | :--- |
| `launch_postgres` | Provision dedicated managed PostgreSQL instance with persistent storage. | `db_identifier`, `db_name`, `db_user`, `storage_gb` |
| `list_postgres` | List all provisioned PostgreSQL databases for tenant. | None |
| `get_postgres_status` | Check database status, operational health, endpoint, and port. | `db_identifier` |
| `get_postgres_credentials` | Retrieve secure database credentials, connection URL (`DATABASE_URL`), and CLI command. | `db_identifier` |
| `pause_postgres` | Hibernate database compute to reduce costs while keeping all storage safely preserved. | `db_identifier` |
| `resume_postgres` | Resume a hibernated database with all data intact. | `db_identifier` |
| `resize_postgres` | Dynamically expand storage volume capacity with zero downtime. | `db_identifier`, `storage_gb` |
| `delete_postgres` | Permanently deprovision database and release cloud storage. | `db_identifier` |

---

## 3. Workspace & Priority Support Tools

| Tool | Purpose & Value | Primary Arguments |
| :--- | :--- | :--- |
| `get_workspace_info` | Query workspace name, slug, tenant ID, quotas, and owner details. | None |
| `submit_admin_query` | Submit a priority ticket, technical inquiry, or quota request directly to `admin@productecho.com`. | `message`, `subject`, `topic`, `error_details`, `metadata` |

---

## 4. Custom Domains for Static CDN Applications

| Tool | Purpose & Value | Primary Arguments |
| :--- | :--- | :--- |
| `check_domain_availability` | Verify a subdomain prefix is available before deploying. | `domain_prefix`, `custom_domain`, `application_name`, `deployment_target` |
| `list_custom_domains` | List a static CDN application's custom domains with their DNS records and status. | `application_name` |
| `add_custom_domain` | Add a custom domain to a deployed static CDN application; returns the two DNS records to publish (an ownership TXT and a routing CNAME) and a `next_action` to relay. | `application_name`, `hostname` |
| `check_custom_domain` | Look at the domain's DNS and certificate now and move it along; repeat until `domain.status` is `active`. | `application_name`, `domain_id` |
| `remove_custom_domain` | Remove a custom domain and release its hostname. Safe to repeat. | `application_name`, `domain_id` |
