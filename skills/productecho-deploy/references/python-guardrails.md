# Python Deployment Guidelines & Packaging Reference

This guide details packaging rules, production server requirements, and runtime specifications for deploying Python services (FastAPI, Django, Flask, Litestar) to **ProductEcho**.

---

## 1. Dependency Manifests & Python Version

### Supported Manifests:
- **`requirements.txt`**: Pinned production dependencies (recommended).
- **`pyproject.toml`**: Standard packaging specification.
- **`runtime.txt`**: Optional explicit Python version declaration (e.g. `python-3.12.x` or `3.12.*`).

### Virtualenv & Cache Exclusions:
- **NEVER** upload `.venv/`, `venv/`, `__pycache__/`, or `*.pyc` files.
- The build engine creates its own clean virtual environment matching the declared requirements.

---

## 2. Production ASGI / WSGI Server Required

**CRITICAL**: Never use development servers in production (`flask run`, `python manage.py runserver`, or `uvicorn --reload`). Always declare a production ASGI/WSGI server in `requirements.txt` and in your `Procfile` or entrypoint.

### Recommended Production Servers:
- **FastAPI / Starlette / ASGI**: `uvicorn[standard]>=0.28.0` or `granian>=1.2.0`
- **Django / Flask / WSGI**: `gunicorn>=21.2.0` (with `uvicorn.workers.UvicornWorker` for async Django)

### `Procfile` Examples:
Place a `Procfile` in the project root to define the process startup command. If the repository cannot carry one, pass the command as `settings={"start_command": "..."}` to `deploy_application` instead (an explicit value overrides a Procfile).

#### FastAPI:
```text
web: uvicorn main:app --host 0.0.0.0 --port ${PORT:-8000} --workers 2 --no-access-log
```

#### Django:
```text
web: gunicorn myproject.wsgi:application --bind 0.0.0.0:${PORT:-8000} --workers 2 --timeout 60
```

#### Flask:
```text
web: gunicorn "app:create_app()" --bind 0.0.0.0:${PORT:-8000} --workers 2
```

---

## 3. Port Ingress & Host Binding

- **Always bind to `0.0.0.0`**:
  Do **not** bind only to internal loopback (`127.0.0.1`), which prevents incoming traffic from reaching your application service.
- **Dynamic Port Injected by Control Plane**:
  Always parse `os.getenv("PORT", "8000")` or use the shell variable `${PORT:-8000}`.

### Python Code Example:
```python
import os
import uvicorn
from fastapi import FastAPI

app = FastAPI(title="ProductEcho Python Service")

@app.get("/healthz")
def health_check():
    return {"status": "ok"}

if __name__ == "__main__":
    port = int(os.getenv("PORT", "8000"))
    uvicorn.run("main:app", host="0.0.0.0", port=port)
```

---

## 4. Unbuffered Output & Structured Logging

Ensure Python standard output streams directly to log collectors without buffer delay:
- Set environment variable: `PYTHONUNBUFFERED=1`
- Emit structured logs to `sys.stdout` for real-time log streaming.

---

## 5. PostgreSQL Database Connection

When connecting to ProductEcho PostgreSQL instances:
- Use standard async / sync drivers: `asyncpg`, `psycopg[binary]>=3.1.0`, or `sqlalchemy[asyncio]`.
- Read connection URI from `DATABASE_URL` environment variable:
  ```python
  import os
  DATABASE_URL = os.getenv("DATABASE_URL")
  ```

---

## 6. Zip Packaging & Exclusions

When bundling Python source code into a `.zip` archive for upload:

### Exclude:
- Virtual environments: `.venv/`, `venv/`, `ENV/`
- Bytecode & cache: `__pycache__/`, `*.pyc`, `*.pyo`, `.pytest_cache/`, `.mypy_cache/`, `.ruff_cache/`
- Build outputs: `dist/`, `build/`, `*.egg-info/`, `.coverage`
- VCS & IDE: `.git/`, `.idea/`, `.vscode/`, `.DS_Store`
- Environment & Secret files: `.env`, `.env.*`, `*.pem`, `*.key`
