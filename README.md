# Docker Demo: Multi-Stage Image (FastAPI)

- [Docker Demo: Multi-Stage Image (FastAPI)](#docker-demo-multi-stage-image-fastapi)
  - [Project Overview](#project-overview)
  - [Result](#result)
  - [Key Concept: Multi-Stage Build](#key-concept-multi-stage-build)
  - [Key Code Explanation](#key-code-explanation)
    - [Set Environment Variables](#set-environment-variables)
    - [Install Dependencies into an Isolated Directory](#install-dependencies-into-an-isolated-directory)
    - [Copy Dependencies from the Builder Stage](#copy-dependencies-from-the-builder-stage)
    - [Why Not Install OS Build Tools](#why-not-install-os-build-tools)

---

## Project Overview

This project demonstrates the difference between a **single-stage** and a **multi-stage** Docker build using a minimal FastAPI application. The goal is to compare the resulting image sizes and highlight the key techniques that make a multi-stage build leaner.

---

## Result

| Build Type     | Tag                 | Size       |
| -------------- | ------------------- | ---------- |
| Single-stage   | `webapi:latest`     | 247 MB     |
| Multi-stage    | `webapi:multistage` | 231 MB     |
| **Difference** |                     | **−16 MB** |

- The **size reduction is modest** here because both images are based on `python:3.12-slim` and both dependencies (`fastapi`, `uvicorn`) ship as **pre-built wheels** — no compilation is needed, so there are no temporary build artifacts to discard.
- The **multi-stage** pattern delivers **a larger benefit when the builder stage requires heavy tools** (e.g., `gcc`, `Rust`, or `node_modules`) that have no place in the final runtime image.

---

## Key Concept: Multi-Stage Build

- `multi-stage build`
  - uses **multiple `FROM` instructions** in a single `Dockerfile`, each defining a separate stage. Files can be selectively copied from one stage into the next with `COPY --from=<stage>`.

```
Stage 1 — builder
├── Base image: python:3.12-slim
├── Install pip dependencies → /install
└── (discarded after build)

Stage 2 — runtime
├── Base image: python:3.12-slim (fresh, no build residue)
├── COPY --from=builder /install → /usr/local  (packages only)
└── COPY ./app → /app  (source code only)
```

- The `builder stage` can **install compilers, headers, and other build-time tools** freely — they never reach the final image.
- Only the **outputs (installed packages) are carried over**. This keeps the runtime image free of unnecessary tooling.

---

## Key Code Explanation

### Set Environment Variables

```dockerfile
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
```

| Variable                    | Effect                                                                                                       |
| --------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `PYTHONDONTWRITEBYTECODE=1` | Stops Python from writing `.pyc` / `.pyo` cache files — reduces image clutter                                |
| `PYTHONUNBUFFERED=1`        | Sends `stdout` / `stderr` straight to the terminal without buffering — essential for readable container logs |

---

### Install Dependencies into an Isolated Directory

```dockerfile
RUN pip install --no-cache-dir --upgrade pip \
    && pip install --no-cache-dir --prefix=/install -r requirements.txt
```

| Flag                | Effect                                                                                                                                                |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--no-cache-dir`    | Skips the pip download cache — nothing to discard later, keeps the layer smaller                                                                      |
| `--prefix=/install` | Installs all packages under `/install` instead of the system Python paths — makes the installed files easy to copy as a single unit in the next stage |

---

### Copy Dependencies from the Builder Stage

```dockerfile
COPY --from=builder /install /usr/local
```

| Part                    | Effect                                                                                           |
| ----------------------- | ------------------------------------------------------------------------------------------------ |
| `--from=builder`        | Pulls files from the named builder stage, not from the host filesystem                           |
| `COPY /install /usr/local` | Merges the isolated install directory into the standard Python library path of the runtime image |

This is the **core** of the `multi-stage pattern`: only the compiled/installed packages travel to the final image. The builder's base layer, pip cache, and any temporary files are left behind.

---

### Why Not Install OS Build Tools

A common pattern seen in Python Dockerfiles is installing `build-essential` to support packages that need to `compile C extensions`:

```dockerfile
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*
```

- `build-essential` pulls in the full GCC toolchain (~200 MB). It is only needed when `pip` cannot find a pre-built wheel and must compile from source.

- This project's two dependencies — `fastapi` and `uvicorn[standard]` — both ship pre-built wheels on PyPI for `python3.12` on `linux/amd64`. When pip finds a wheel it simply unpacks it; no C compiler is ever invoked. Adding `build-essential` would contribute ~200 MB to the image for **zero benefit**.

> The general rule: check whether your dependencies have wheels available for your target platform before reaching for `build-essential`. If they do, omit it entirely.

---
