# Part 1 · Layer Fundamentals

In this part, you'll build a deliberately bad image, see why it's bad, and fix it step by step.

---

## Step 1: Build the bloated image

Create this `Dockerfile` in your `docker-lab/` directory:

```dockerfile title="Dockerfile (v1 — bloated)"
FROM python:3.12

WORKDIR /app

# Copy everything first (BAD order)
COPY . .

# Install dependencies, download a big file, then try to delete it
RUN pip install -r requirements.txt
RUN apt-get update && apt-get install -y curl
RUN curl -o bigfile.bin https://releases.ubuntu.com/24.04/ubuntu-24.04.2-desktop-amd64.iso || \
    dd if=/dev/zero of=bigfile.bin bs=1M count=50
RUN rm bigfile.bin

EXPOSE 8080
CMD ["python", "app.py"]
```

!!! note
    The `dd` command creates a 50 MB dummy file if the download fails — simulating a large build artifact. The point is to see that deleting it in a later layer does nothing.

Build it:

```bash
docker build -t myapp:v1 .
```

Check the size:

```bash
docker images myapp
```

Note the size. It will be **much larger** than expected.

---

## Step 2: Inspect the layers

```bash
docker history myapp:v1
```

You'll see each layer and its size. Notice:

- The `RUN rm bigfile.bin` layer is tiny — but the layer that created the file is still large
- The large full `python:3.12` base (~1 GB) is the starting point

---

## Step 3: Fix problem 1 — Layer ordering

Every time you change *any* file in your project, `COPY . .` at the top invalidates the cache, forcing `pip install` to run again. That's slow.

Fix: copy the dependency manifest first, install, then copy the source.

```dockerfile title="Dockerfile (v2 — correct ordering)"
FROM python:3.12-slim              # (1) also switched to slim

WORKDIR /app

COPY requirements.txt .            # (2) manifest only — stable
RUN pip install -r requirements.txt  # cached until requirements.txt changes
COPY . .                           # (3) source code — changes often

EXPOSE 8080
CMD ["python", "app.py"]
```

Build it:

```bash
docker build -t myapp:v2 .
```

Now make a tiny change to `app.py` (e.g., change the greeting text) and rebuild:

```bash
docker build -t myapp:v2 .
```

Watch the output — `pip install` uses the cache. Rebuild is near-instant.

---

## Step 4: Fix problem 2 — Phantom deletion

The `rm bigfile.bin` in v1 didn't help because the file was created in an *earlier* layer. Cleanup must happen in the **same `RUN`** instruction.

```dockerfile title="Dockerfile (v3 — cleanup in same RUN)"
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt   # (1) --no-cache-dir avoids pip's own cache

# Simulate a build artifact — create AND delete in the same RUN
RUN apt-get update \
    && apt-get install -y --no-install-recommends curl \
    && dd if=/dev/zero of=bigfile.bin bs=1M count=50 \
    && rm bigfile.bin \                              # (2) delete in same RUN
    && rm -rf /var/lib/apt/lists/*                  # (3) clean apt cache too

COPY . .

EXPOSE 8080
CMD ["python", "app.py"]
```

Build and compare sizes:

```bash
docker build -t myapp:v3 .
docker images myapp
```

---

## Step 5: Before / After comparison

Run this to see all three versions side by side:

```bash
docker images myapp --format "table {{.Tag}}\t{{.Size}}"
```

You should see something like:

```
TAG    SIZE
v3     ~200 MB
v2     ~190 MB
v1     ~1.3 GB
```

!!! success "What you fixed"
    - Switched from `python:3.12` (~1 GB) to `python:3.12-slim` (~130 MB) — the single biggest win
    - Fixed layer ordering — `pip install` now runs from cache on routine changes
    - Fixed phantom deletion — cleanup happens in the same `RUN`, so deleted files don't bloat the image

---

## Checkpoint

Before moving on, make sure you understand:

- [x] Why `RUN rm` in a separate step doesn't reduce image size
- [x] Why `COPY requirements.txt` before `COPY . .` speeds up rebuilds
- [x] Why switching from `python:3.12` to `python:3.12-slim` is the biggest single win

[:octicons-arrow-right-24: Continue to Part 2](part2.md)
