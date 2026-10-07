# ProductEcho Multi-Language Deployment & Production Readiness Guidelines

This reference documents the best practices and runtime specifications for building and deploying production-grade web applications and backend APIs on **ProductEcho**.

---

## 1. Node.js & Frontend Workloads

### Strict Dependency Separation
Production builders prune development dependencies to create lightweight, secure runtime images (`NODE_ENV=production` or `npm prune --production`).

- **Required in `dependencies`**:
  - Web frameworks: `next`, `vinxi`, `express`, `fastify`, `@nestjs/core`, `koa`, `remix`, `astro`, `@sveltejs/kit`, `hono`
  - Core libraries: `react`, `react-dom`, `vue`, `svelte`
  - Database clients: `pg`, `postgres`, `prisma`, `@prisma/client`, `drizzle-orm`, `typeorm`, `ioredis`
  - Utility runtime libs: `zod`, `dotenv`, `cors`, `helmet`, `jsonwebtoken`
- **Only in `devDependencies`**:
  - Linters/Formatters: `eslint`, `prettier`
  - Type definitions: `@types/node`, `@types/react`, `@types/express`, etc.
  - Compilers/Build tools: `typescript`, `ts-node`, `tailwindcss`, `postcss`, `autoprefixer`
  - Test suites: `vitest`, `jest`, `playwright`, `cypress`

### Relative Binary Execution
- In `package.json` scripts, avoid executing naked global commands.
- Always use `npx <command>` or `./node_modules/.bin/<command>`:
  ```json
  "scripts": {
    "dev": "npx next dev",
    "build": "NODE_ENV=production next build --webpack",
    "start": "npx next start -p ${PORT:-3000}"
  }
  ```

### Engine Declaration
- Declare target Node.js version in `package.json`:
  ```json
  "engines": {
    "node": ">=20.0.0",
    "npm": ">=10.0.0"
  }
  ```

---

## 2. Python Backend Workloads

- **Manifests**: Provide `requirements.txt` or `pyproject.toml` at the project root.
- **Production ASGI/WSGI Server**:
  - Always include a production server (`uvicorn`, `gunicorn`, `granian`) in `requirements.txt`.
  - Provide a `Procfile` at the root:
    ```text
    web: uvicorn main:app --host 0.0.0.0 --port ${PORT:-8000} --workers 2
    ```
  - No Procfile? Pass the command as `settings={"start_command": "..."}` to `deploy_application`; it overrides a Procfile.
- **Port Ingress & Host Binding**:
  - Always bind to `0.0.0.0` reading `int(os.getenv("PORT", 8000))`.
- **Unbuffered Logging**:
  - Set `PYTHONUNBUFFERED=1` to ensure real-time log output.

---

## 3. Golang Workloads

- **Module Manifests**:
  - Ensure `go.mod` and `go.sum` are present at the project root with declared Go version (`go 1.22+`).
- **Entrypoint**:
  - Root `main.go` or standard `cmd/<app>/main.go`.
- **Port Ingress & Host Binding**:
  - Bind to `0.0.0.0:<PORT>` or `:<PORT>` where `PORT = os.Getenv("PORT")` (default `8080`).
- **Graceful Shutdown**:
  - Catch `SIGTERM` and `SIGINT` signals to gracefully drain requests before shutdown.

---

## 4. Java & JVM Workloads

- **Build Wrappers**:
  - Include Maven (`pom.xml`) with `./mvnw` wrapper or Gradle (`build.gradle`) with `./gradlew` wrapper.
- **JDK Version**:
  - Declare Java LTS target version (Java 17 or 21) in `pom.xml` / `build.gradle`.
- **Server Port Configuration**:
  - Spring Boot: `server.port=${PORT:8080}` and `server.address=0.0.0.0` in `application.properties` or `application.yml`.
  - Quarkus / Micronaut: Bind port to `${PORT:8080}` and host to `0.0.0.0`.

---

## 5. Generic Workload Requirements

### Dynamic Port Ingress Mapping
- The platform dynamically routes incoming HTTPS traffic to the application's listening container port (`container_port`, default 3000 for Node, 8000 for Python, 8080 for Go/Java).
- The application runtime **must** bind to the `PORT` environment variable.
