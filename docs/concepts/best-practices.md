# Best Practices

Three techniques separate a bloated, fragile image from a lean, production-ready one.

---

## 1. Order layers: slow-changing → fast-changing

Docker caches each layer. **Once a layer changes, all layers below it are rebuilt.** This means the order of your instructions has a huge impact on build speed.

**The rule:** Put things that change rarely near the top. Put things that change often near the bottom.

```
Most stable                                        Least stable
──────────────────────────────────────────────────────────────►
FROM base  →  OS packages  →  dependency manifest  →  install deps  →  source code
```

=== "❌ Slow — deps reinstall on every code change"

    ```dockerfile
    FROM python:3.12-slim
    WORKDIR /app
    COPY . .                          # copies ALL source code
    RUN pip install -r requirements.txt  # reinstalls everything if ANY file changed
    ```

=== "✅ Fast — deps only reinstall when requirements.txt changes"

    ```dockerfile
    FROM python:3.12-slim
    WORKDIR /app
    COPY requirements.txt .           # copy only the manifest
    RUN pip install -r requirements.txt  # cached until requirements.txt changes
    COPY . .                          # copy source code last
    ```

!!! tip "Why this matters"
    On a real project, `pip install` or `npm install` can take 2–5 minutes. With the correct ordering, that step runs from cache on every routine code change — making rebuilds nearly instant.

---

## 2. Choose a minimal base image

The base image is the foundation everything else builds on. Smaller base = smaller final image = faster pulls, fewer CVEs.

| Base image | Approximate size | Best for |
|---|---|---|
| `scratch` | 0 MB | Statically compiled Go or Rust binaries |
| Google Distroless | ~2–20 MB | Production Java, Python, Node.js runtimes |
| `alpine:3.20` | ~7 MB | General-purpose minimal images |
| `python:3.12-slim` | ~130 MB | Most Python apps ✅ |
| `python:3.12` | ~1 GB | Only when you need the full Debian toolchain |

!!! note "Alpine caveats"
    Alpine uses `musl libc` instead of `glibc`. Most Python packages work fine, but some C extensions (numpy, pandas, certain database drivers) behave differently or require recompilation. Test before committing to Alpine.

---

## 3. Use multi-stage builds

Multi-stage builds let you use a heavy "builder" image for compilation and only copy the result into a lightweight "runtime" image.

```dockerfile
# Stage 1: build
FROM python:3.12 AS builder          # full image — has pip, compilers
WORKDIR /app
COPY requirements.txt .
RUN pip install --prefix=/install -r requirements.txt

# Stage 2: runtime
FROM python:3.12-slim                # slim image — just what we need to run
WORKDIR /app
COPY --from=builder /install /usr/local   # copy only the installed packages
COPY src/ .
CMD ["python", "app.py"]
```

The `builder` stage is **completely discarded** — it never makes it into the final image. Only the `COPY --from=builder` lines survive.

!!! success "Real-world size reductions"
    | Language | Without multi-stage | With multi-stage |
    |---|---|---|
    | Java | ~880 MB | ~428 MB |
    | Go | ~800 MB | ~5–20 MB (into `scratch`) |
    | Node.js | ~900 MB | ~150 MB |

---

## 4. Use `.dockerignore`

When you run `docker build .`, Docker sends your **entire current directory** to the daemon before the build starts. A `.dockerignore` file tells Docker what to exclude — similar to `.gitignore`.

```
# .dockerignore
.git/
.venv/
__pycache__/
*.pyc
node_modules/
.env
*.log
```

!!! warning "Two reasons to always use `.dockerignore`"
    1. **Speed** — sending `node_modules/` or `.venv/` can add seconds or minutes before a single layer is built.
    2. **Security** — without it, `.env` files, SSH keys, or other secrets in your directory can accidentally end up in the image.

---

## 5. Run as non-root

Never ship a production image where your app runs as root. Create a dedicated system user and switch to it before the final `CMD` or `ENTRYPOINT`.

```dockerfile
RUN useradd --system --no-create-home appuser
USER appuser
```

See [Dockerfile Guide → USER](dockerfile-guide.md#user-run-as-non-root) for details.

---

## Quick reference

| Practice | Why it matters |
|---|---|
| Order: stable → volatile | Keeps expensive layers cached |
| Minimal base image | Fewer CVEs, faster pulls |
| Multi-stage builds | Discard build tools from the final image |
| `.dockerignore` | Faster builds, no accidental secrets |
| Non-root `USER` | Limits blast radius of a compromise |
| Pin image tags | Reproducible, no surprise changes |
| Cleanup in same `RUN` | Phantom deletions don't reduce size |

---

!!! success "You're ready for the lab"
    You now have the mental model and the techniques. Time to put them into practice.

    [:octicons-arrow-right-24: Start the Lab](../lab/index.md)
