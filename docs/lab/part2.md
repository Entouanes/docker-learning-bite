# Part 2 · Multi-Stage Builds

In Part 1 you reduced the image size by fixing layer order and cleanup. In this part, you'll use a **multi-stage build** to go further — separating the build environment from the runtime image entirely.

---

## The idea

The build tools you need to compile or package your app (compilers, pip, build headers) don't need to be in the final image. With multi-stage builds, you use a heavy builder image and copy only the output into a lightweight runtime image.

```
Stage 1 (builder) ──install deps──► /install/
                                         │
                                    COPY --from=builder
                                         │
Stage 2 (runtime) ◄──────────────────── /usr/local/
```

The builder stage is **completely discarded** — not tagged, not pushed, not visible in `docker images`.

---

## Step 1: Convert to a multi-stage Dockerfile

Replace your `Dockerfile` with this:

```dockerfile title="Dockerfile (v4 — multi-stage)"
# ── Stage 1: builder ──────────────────────────────────────────
FROM python:3.12 AS builder           # (1) full image — has pip and build tools

WORKDIR /app

COPY requirements.txt .
RUN pip install --prefix=/install --no-cache-dir -r requirements.txt  # (2) install into /install

# ── Stage 2: runtime ──────────────────────────────────────────
FROM python:3.12-slim                 # (3) lightweight runtime image

WORKDIR /app

COPY --from=builder /install /usr/local   # (4) only the installed packages — no pip, no build tools
COPY . .

RUN useradd --system --no-create-home appuser  # (5) non-root user
USER appuser

EXPOSE 8080
CMD ["gunicorn", "--bind", "0.0.0.0:8080", "app:app"]
```

Annotations:

1. The `AS builder` alias names this stage so we can reference it with `COPY --from=builder`
2. `--prefix=/install` installs packages into `/install` instead of the system prefix — easier to copy
3. The final stage starts fresh from the slim base; it has no memory of the builder stage
4. `COPY --from=builder` is the only bridge between stages
5. We also add a non-root user — a production best practice

---

## Step 2: Build and compare

```bash
docker build -t myapp:v4 .
docker images myapp --format "table {{.Tag}}\t{{.Size}}"
```

Expected output (approximate):

```
TAG    SIZE
v4     ~160 MB
v3     ~200 MB
v2     ~190 MB
v1     ~1.3 GB
```

!!! success "What changed in v4?"
    The final image contains only the installed Python packages and your source code. The full `python:3.12` image (with pip, build headers, and all of Debian's toolchain) was used to install dependencies — then discarded. It never made it into v4.

---

## Step 3: Verify it works

Run the container to make sure everything actually works:

```bash
docker run -p 8080:8080 myapp:v4
```

Open [http://localhost:8080](http://localhost:8080) — you should see `Hello from Docker!`.

Stop the container with ++ctrl+c++.

---

## Step 4: Inspect what's inside

Notice that build tools are gone from the final image:

```bash
# Start a shell in the container
docker run --rm -it myapp:v4 sh

# Inside the container — try these:
pip --version          # should fail — pip is not installed
python --version       # should work — Python is there
gunicorn --version     # should work — gunicorn was installed via COPY --from
exit
```

---

## Final comparison

| Version | Change | Size |
|---|---|---|
| v1 | Bloated — full base, phantom deletion | ~1.3 GB |
| v2 | Fixed layer ordering | ~190 MB |
| v3 | Fixed cleanup + slim base | ~200 MB |
| v4 | Multi-stage + non-root | ~160 MB |

The biggest wins were:
1. **Switching to `python:3.12-slim`** — dropped from 1 GB+ to ~190 MB
2. **Multi-stage build** — removed pip and build tooling from the final image

---

!!! success "Congratulations — you're done!"

    You've learned and applied every key technique for writing clean Docker images:

    - [x] The layer mental model and phantom deletion
    - [x] Optimised layer ordering for fast incremental builds
    - [x] Choosing a minimal base image
    - [x] Multi-stage builds to discard build tools
    - [x] Running as a non-root user

    **Next steps:** Try applying these techniques to a real project. Start with base image selection and layer ordering — those two changes alone often reduce image size by 80%.
