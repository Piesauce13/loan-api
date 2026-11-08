# Production Container Requirements Checklist

Evidence was collected from:

```powershell
docker build -f Dockerfile.prod -t loan-api-prod-check .
```

Build result:

```text
#17 DONE 0.2s
Successfully tagged loan-api-prod-check:latest
```

## 1. Slim base image

**Requirement:** Use a slim base image to reduce packages and vulnerabilities.

**Command:**

```powershell
Select-String -Path Dockerfile.prod -Pattern '^FROM'
```

**Output:**

```text
Dockerfile.prod:2:FROM python:3.12-slim AS builder
Dockerfile.prod:11:FROM python:3.12-slim
```

Both stages use `python:3.12-slim`.

## 2. Multi-stage build

**Requirement:** Keep build tools and training code in the builder stage; ship only runtime files.

**Command:**

```powershell
docker history --no-trunc --format '{{.CreatedBy}}' loan-api-prod-check
docker run --rm --entrypoint sh loan-api-prod-check -c "find /app -maxdepth 1 -type f -printf '%f\n' | sort"
```

**Output:**

```text
COPY --from=builder /opt/venv /opt/venv
COPY --from=builder /app/model.joblib .
COPY schemas.py main.py ./

main.py
model.joblib
schemas.py
```

The runtime image contains the virtual environment, model, and application files, but not `train.py`.

## 3. Separate development dependencies

**Requirement:** Production installs only `requirements.txt`; test dependencies are in `requirements-dev.txt`.

**Command:**

```powershell
Get-Content requirements.txt
Get-Content requirements-dev.txt
docker run --rm --entrypoint sh loan-api-prod-check -c "command -v pytest || echo absent"
```

**Output:**

```text
requirements.txt:
fastapi[standard]==0.150.0
scikit-learn==1.9.1
pandas==3.0.6
numpy==2.5.4
joblib==1.6.0

requirements-dev.txt:
-r requirements.txt
pytest==9.1.1

absent
```

`pytest` is absent from the production image.

## 4. Run as a non-root user

**Requirement:** A compromised app must not run with system administrator privileges.

**Command:**

```powershell
docker run --rm --entrypoint id loan-api-prod-check
docker image inspect loan-api-prod-check --format 'User={{.Config.User}}'
```

**Output:**

```text
uid=1000(appuser) gid=1000(appuser) groups=1000(appuser)
User=appuser
```

The production container runs as `appuser`, not root.

## 5. Pin versions

**Requirement:** Use a repeatable Python base and pinned package versions.

**Command:**

```powershell
Get-Content Dockerfile.prod
Get-Content requirements.txt
```

**Output:**

```text
FROM python:3.12-slim AS builder
FROM python:3.12-slim

fastapi[standard]==0.150.0
scikit-learn==1.9.1
pandas==3.0.6
numpy==2.5.4
joblib==1.6.0
```

The Python image and application package versions are explicitly specified.

## 6. HEALTHCHECK

**Requirement:** Docker and hosting platforms must detect an unhealthy or stuck container.

**Command:**

```powershell
docker image inspect loan-api-prod-check --format 'Healthcheck={{json .Config.Healthcheck}}'
docker run --rm --entrypoint sh loan-api-prod-check -c "python -c \"import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health')\""
```

**Output:**

```text
Healthcheck={"Test":["CMD-SHELL","python -c \"import urllib.request; urllib.request.urlopen('http://127.0.0.1:8000/health')\""],"Interval":30000000000,"Timeout":3000000000,"Retries":3}
<successful command; exit code 0>
```

Live endpoint verification:

```text
HTTP/1.1 200 OK
{"status":"ok","model_version":"1.0.0"}
```

## 7. Scan regularly

**Requirement:** Scan the image because new vulnerabilities are published every day.

**Command:**

```powershell
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock `
  -v ${HOME}/.cache:/root/.cache `
  aquasec/trivy:0.68.1 image --severity HIGH,CRITICAL --ignore-unfixed loan-api-prod-check
```

**Output:**

```text
loan-api-prod-check (debian 13.7)
Total: 2 (HIGH: 2, CRITICAL: 0)

zlib1g | CVE-2026-76844 | HIGH | fixed
Installed: 1:1.3.dfsg+really1.3.1-1+b1
Fixed:     1:1.3.dfsg+really1.3.1-1+deb13u1

zlib1g | CVE-2026-85091 | HIGH | fixed
Installed: 1:1.3.dfsg+really1.3.1-1+b1
Fixed:     1:1.3.dfsg+really1.3.1-1+deb13u1
```

**HIGH finding notes:**

- `CVE-2026-76844`: A fixed version is available, but rebuilding with the current `python:3.12-slim` base still reports the vulnerable Debian zlib package. Use a newer patched base image or explicitly update the runtime OS package.
- `CVE-2026-85091`: Same remediation: rebuild after the base image includes `zlib1g` version `1:1.3.dfsg+really1.3.1-1+deb13u1` or later.

No CRITICAL findings were reported.
