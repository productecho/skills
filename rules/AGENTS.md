# ProductEcho Multi-Language Deployment Guardrails & Rules

When writing code, configuring project manifests, creating build scripts, or packaging applications for deployment to ProductEcho Cloud:

### 1. Node.js & Frontend Guardrails
- **Strict Dependency Separation**: Production server runtime dependencies (`next`, `vinxi`, `express`, `fastify`, `@nestjs/core`, `koa`, `remix`, `astro`, `@sveltejs/kit`, `hono`) MUST be listed under `"dependencies"`, never `"devDependencies"`.
- **Relative Binary Execution**: Avoid naked commands in `package.json` scripts. Use `"start": "npx <cmd>"` or `"start": "./node_modules/.bin/<cmd>"`.
- **Specify Engine Version**: Always declare supported Node range in `package.json`: `"engines": { "node": ">=20.0.0" }`.
- **Next.js 16+ Turbopack vs Webpack**: Next.js 16+ defaults to Turbopack which breaks under buildpacks. Append `--webpack` to build scripts (`"build": "NODE_ENV=production next build --webpack"`) or provide a `Dockerfile`.

### 2. Python (FastAPI / Django / Flask) Guardrails
- **Production Server**: Include a production ASGI/WSGI server (`uvicorn`, `gunicorn`, `granian`) in `requirements.txt`.
- **`Procfile`**: Provide a root `Procfile` (e.g. `web: uvicorn main:app --host 0.0.0.0 --port ${PORT:-8000} --workers 2`), or pass `settings={"start_command": "..."}` to `deploy_application`.
- **Unbuffered Logging**: Ensure `PYTHONUNBUFFERED=1` is set.
- **Port Ingress**: Bind to `0.0.0.0` and read `os.getenv("PORT", "8000")`.

### 3. Golang Guardrails
- **Manifests**: Include `go.mod` and `go.sum` at the project root with declared Go version (`go 1.22+`).
- **Port Ingress**: Bind to `"0.0.0.0:" + os.Getenv("PORT")` (default `8080`).
- **CGO Disabled**: Buildpacks compile with `CGO_ENABLED=0` by default; avoid dynamic C library dependencies.
- **Graceful Shutdown**: Implement `SIGTERM` / `SIGINT` signal catching.

### 4. Java & JVM (Spring Boot / Quarkus / Micronaut) Guardrails
- **Build Tools & Wrappers**: Include `./mvnw` or `./gradlew` and make them executable.
- **JDK Version**: Declare Java 17 or 21 LTS in `pom.xml` or `build.gradle`.
- **Port Binding**: Set `server.port=${PORT:8080}` and `server.address=0.0.0.0` in configuration.
- **Memory**: Allow automatic JVM memory calculation; avoid hardcoding static `-Xmx4g` flags that exceed container limits.

### 5. Generic Workload Requirements, State Linking & Zip Exclusions
- **Strict Project State Schema (`.productecho/state.json`)**:
  Project identity and configuration are tracked at `<target_root>/.productecho/state.json`.
  When writing or modifying this file, agents MUST write ONLY the canonical 6 identity attributes:
  ```json
  {
    "version": "1.0",
    "application_name": "<name>",
    "db_identifier": "<db_id>" | null,
    "root_directory": "<dir>" | null,
    "remote_repo": "<owner/repo>" | null,
    "updated_at": "<iso_timestamp>"
  }
  ```
  **FORBIDDEN ATTRIBUTES**: NEVER write dynamic runtime or cloud-managed attributes like `"domain_url"`, `"status"`, `"container_port"`, or `"env_vars"` into `.productecho/state.json`. Cloud status, runtime URLs, and ingress states belong exclusively in the cloud database (`tenant_project_link` / `resources`) and must be queried dynamically via `get_application_status` or the dashboard.
- **Bidirectional Project State Linking**: Always ensure `.productecho/` is added to `.gitignore` by default so machine-specific credentials or temporary runtime states are not committed.
- **Port Ingress**: Bind to `process.env.PORT` or `os.getenv("PORT")` (default 3000 / 8000 / 8080) on host `0.0.0.0`.
- **Zip Exclusions**: Exclude `.git`, `node_modules`, `.next`, `.venv`, `target`, `build`, `.env*`, `.productecho`, and build outputs from source archives.

