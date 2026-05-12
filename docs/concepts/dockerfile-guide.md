# Dockerfile Guide

A Dockerfile is a plain-text list of instructions that Docker executes **top to bottom** to build an image. Here are the instructions you'll use most — what they do, and what to watch out for.

---

## `FROM` — Choose your base

Every Dockerfile starts with `FROM`. It sets the base image all subsequent layers build on.

```dockerfile
FROM python:3.12-slim
```

!!! tip "Pick the right variant"
    Most official images come in several flavours. Start with `slim` or `alpine` — they're much smaller than the default.

    | Tag | Approximate size | Use when |
    |---|---|---|
    | `python:3.12` | ~1 GB | You need full Debian toolchain |
    | `python:3.12-slim` | ~130 MB | Most Python apps ✅ |
    | `python:3.12-alpine` | ~55 MB | Size is critical; aware of musl differences |

!!! warning "Never use `latest`"
    `FROM python:latest` is a moving target — your image will silently change whenever the publisher updates the tag. Always pin to a specific version: `FROM python:3.12.3-slim`.

---

## `WORKDIR` — Set the working directory

Sets the current directory for all following instructions. Creates the directory if it doesn't exist.

```dockerfile
WORKDIR /app
```

!!! tip
    Always use `WORKDIR` instead of `RUN cd /path && ...`. A `cd` in `RUN` only lasts for that one instruction. `WORKDIR` persists.

---

## `COPY` vs `ADD` — Bring files into the image

**Use `COPY` by default.** It does exactly one thing: copies files from your build context into the image.

```dockerfile
COPY requirements.txt .
COPY src/ ./src/
```

**Use `ADD` only when you specifically need its extras:**

| Need | Use |
|---|---|
| Copy local files | `COPY` |
| Copy from another build stage | `COPY --from=builder` |
| Auto-extract a local `.tar.gz` | `ADD archive.tar.gz /dest/` |
| Download from a URL | `ADD https://example.com/file.tar.gz /dest/` |

!!! warning
    `ADD` silently extracts tar archives. That can be surprising. Prefer `COPY` unless you need the extras.

---

## `RUN` — Execute commands at build time

Runs a shell command and commits the result as a new layer.

```dockerfile
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*
```

The `&&` chaining is intentional. Remember the [phantom deletion rule](how-docker-works.md#the-phantom-deletion-problem): cleanup must happen **in the same `RUN`**.

!!! tip "Use `--no-install-recommends`"
    With apt, this skips suggested packages and significantly reduces layer size.

---

## `ENV` vs `ARG` — Variables

```dockerfile
ARG APP_VERSION=1.0       # Build-time only — not in the running container
ENV APP_ENV=production    # Persists into the running container
```

| Feature | `ENV` | `ARG` |
|---|---|---|
| Available during build | ✅ | ✅ |
| Available at runtime | ✅ | ❌ |
| Stored in image metadata | ✅ | ❌ |
| Safe for secrets | ❌ | ❌ |

!!! danger "Never put secrets in `ENV` or `ARG`"
    `ENV` values appear in `docker inspect`. `ARG` values appear in `docker history`. Both are readable by anyone with access to the image. Use `RUN --mount=type=secret` (BuildKit) for build-time secrets, or secrets managers for runtime.

---

## `EXPOSE` — Document ports (documentation only)

```dockerfile
EXPOSE 8080
```

`EXPOSE` does **not** open any ports. It's documentation — it tells users and tooling what port the app uses. To actually publish a port, use `-p 8080:8080` when running the container.

---

## `CMD` vs `ENTRYPOINT` — What runs when the container starts

These two are the most confusing pair in Docker. Here's the simple version:

| Instruction | Purpose | Can be overridden? |
|---|---|---|
| `CMD` | Default command (easily replaceable) | Yes — by passing a command to `docker run` |
| `ENTRYPOINT` | Fixed executable (the container *is* this program) | Only with `--entrypoint` flag |

**Common patterns:**

```dockerfile
# Pattern 1: Simple app — just use CMD
CMD ["python", "app.py"]

# Pattern 2: Fixed service — use ENTRYPOINT for the binary, CMD for default flags
ENTRYPOINT ["gunicorn"]
CMD ["--bind", "0.0.0.0:8080", "app:app"]

# Pattern 3: CLI tool
ENTRYPOINT ["mytool"]
# Users run: docker run myimage --help
```

!!! danger "Always use exec form for ENTRYPOINT"
    Shell form (`ENTRYPOINT python app.py`) wraps your command in `/bin/sh -c`, making the shell PID 1. Your app never receives `SIGTERM` from `docker stop`, so graceful shutdown is impossible.

    ```dockerfile
    # ❌ Shell form — shell becomes PID 1
    ENTRYPOINT python app.py

    # ✅ Exec form — your app is PID 1
    ENTRYPOINT ["python", "app.py"]
    ```

---

## `USER` — Run as non-root

By default, container processes run as root. In production, always switch to a non-privileged user.

```dockerfile
RUN useradd --system --no-create-home appuser
USER appuser
```

!!! tip "Why non-root matters"
    If an attacker exploits a vulnerability in your app, running as root inside the container gives them much more power — especially in container escape scenarios. A non-root user limits the blast radius.

---

!!! success "Key takeaways"
    - Pin your `FROM` tag — never use `latest`
    - Use `WORKDIR`, not `RUN cd`
    - Prefer `COPY` over `ADD` unless you need the extras
    - Chain install+cleanup in the **same `RUN`**
    - Use exec form for `ENTRYPOINT`
    - Never put secrets in `ENV` or `ARG`
    - Always switch to a non-root `USER` in production
