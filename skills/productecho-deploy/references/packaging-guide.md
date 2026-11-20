# ProductEcho Application Packaging & Zip Archive Guide

This guide details how to bundle an application into a clean zip archive suitable for ProductEcho S3 presigned upload.

---

## 1. Archive Structure Rules

1. **Root-Level Manifests**:
   The root of the archive must contain the project entry files:
   - For Node: `package.json`, `package-lock.json` or `pnpm-lock.yaml`, `src/`
   - For Python: `requirements.txt` or `pyproject.toml`, application source code
   - For Docker: `Dockerfile`, `.dockerignore`

2. **No Double-Nesting**:
   When creating the zip archive, do not include an unnecessary outer folder if possible, or ensure the archive root contains only the single application folder (which ProductEcho's source inspector automatically unwraps).

---

## 2. Exclusion Checklist

Always exclude the following patterns from the archive:

| Category | Patterns to Exclude |
| :--- | :--- |
| **Secrets & Keys** | `.env`, `.env.*`, `*.pem`, `*.key`, `credentials.json`, `secrets.yaml` |
| **Dependencies** | `node_modules/`, `vendor/`, `.venv/`, `venv/`, `__pycache__/`, `*.pyc` |
| **Build Outputs** | `.next/`, `dist/`, `build/`, `out/`, `target/`, `.turbo/`, `coverage/` |
| **VCS & IDE** | `.git/`, `.github/`, `.idea/`, `.vscode/`, `.DS_Store` |

---

## 3. Creating the Zip Archive

### Via Terminal Command (macOS / Linux):
```bash
zip -r app-source.zip . \
  -x "*.git*" \
  -x "*node_modules*" \
  -x "*.next*" \
  -x "*.venv*" \
  -x "*venv*" \
  -x "*dist*" \
  -x "*build*" \
  -x "*coverage*" \
  -x "*.env*" \
  -x "*.DS_Store"
```

### Via Python Script:
```python
import zipfile
import os

EXCLUDE_DIRS = {".git", "node_modules", ".next", ".venv", "venv", "dist", "build", "__pycache__"}
EXCLUDE_FILES = {".DS_Store"}

def package_app(source_dir: str, output_zip: str):
    with zipfile.ZipFile(output_zip, "w", zipfile.ZIP_DEFLATED) as zip_file:
        for root, dirs, files in os.walk(source_dir):
            dirs[:] = [d for d in dirs if d not in EXCLUDE_DIRS]
            for file in files:
                if file in EXCLUDE_FILES or file.startswith(".env"):
                    continue
                file_path = os.path.join(root, file)
                arcname = os.path.relpath(file_path, source_dir)
                zip_file.write(file_path, arcname)
```

---

## 4. Uploading to S3

`inspect_application_source` or `deploy_application` returns an S3 upload form: `upload_url`, `upload_fields`, and `max_size_bytes`, the workspace's zip limit (250 MB unless ProductEcho raised it). Check the zip's size first. If it is larger, do not upload it: tell the user its size and the limit, and that they can raise it through https://www.productecho.com/contact.

Send every `upload_fields` entry as its own form field, then the zip as the last field, named `file`:
```bash
curl -X POST "<upload_url>" \
  --form-string "key=<upload_fields.key>" \
  --form-string "policy=<upload_fields.policy>" \
  --form-string "<other field>=<value>" \
  -F "file=@app-source.zip"
```
A `204` response means the upload worked. A `400` containing `EntityTooLarge` means the zip is over the limit.
