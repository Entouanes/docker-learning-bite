# How Docker Works

Docker solves the *"it works on my machine"* problem by packaging your app and everything it needs into a portable, isolated unit called a **container**.

---

## Containers vs. Virtual Machines

The most important distinction: containers are **not** mini VMs.

| | Virtual Machine | Docker Container |
|---|---|---|
| **Isolates** | Hardware (full guest OS) | Process (shared host kernel) |
| **Boot time** | Minutes | Milliseconds |
| **Size** | Gigabytes | Megabytes |
| **Density** | ~10s per host | 100s per host |

A VM virtualizes hardware and runs a complete OS. A container virtualizes the OS userspace — it's just an isolated process with its own filesystem view. Multiple containers share the same Linux kernel on the host.

!!! tip "Real-world setup"
    In the cloud, your VMs *host* many containers. Docker runs on top of the VM — not instead of it.

---

## Images and Containers

Think of it like a class and its instances:

| Concept | Analogy | Description |
|---|---|---|
| **Dockerfile** | Recipe | Instructions for building the image |
| **Image** | Mold / class | Immutable, read-only blueprint |
| **Container** | Cookie / instance | A running image with a writable layer on top |

One image → many containers. Each container gets its own thin writable layer; the underlying image is shared and never modified.

!!! warning "Containers are ephemeral"
    Any data written inside a container is **lost when the container is removed**. Use volumes to persist data.

---

## The Layer System

A Docker image is not a single flat file — it's a **stack of immutable filesystem layers**, each representing the changes made by one build step.

```
Layer 4 (top) ← COPY app/ /app                  [your source code]
Layer 3       ← RUN pip install -r requirements  [your dependencies]
Layer 2       ← COPY requirements.txt /app/      [dependency manifest]
Layer 1       ← RUN apt-get install python3      [Python runtime]
Layer 0 (base)← FROM ubuntu:22.04               [base OS]
```

Each layer stores only the **diff** from the previous one — additions, deletions, modifications. Layers are identified by a SHA256 hash and shared across images: if two images share a base, they share those layers on disk.

### Which instructions create layers?

Only three Dockerfile instructions create new filesystem layers. Everything else is just metadata.

| Creates a layer | Metadata only |
|---|---|
| `RUN`, `COPY`, `ADD` | `CMD`, `ENTRYPOINT`, `ENV`, `ARG`, `EXPOSE`, `LABEL`, `USER`, `WORKDIR`, `VOLUME`, `HEALTHCHECK` |

---

## The Phantom Deletion Problem

This is the most important thing to understand about layers — and the most common mistake.

!!! danger "Deleting a file in a later layer does NOT reduce image size"
    The original bytes still exist in the earlier layer. Docker just adds a "whiteout marker" saying the file is hidden — but it's still transferred when you push or pull the image.

    ```dockerfile
    # ❌ WRONG — the downloaded file is still in Layer 1
    RUN wget https://example.com/big-file.tar.gz
    RUN rm big-file.tar.gz

    # ✅ CORRECT — cleanup in the same RUN; only the diff is committed
    RUN wget https://example.com/big-file.tar.gz \
        && tar -xzf big-file.tar.gz \
        && rm big-file.tar.gz
    ```

---

## What happens when you run `docker run`

1. Docker checks if the image exists locally; if not, **pulls it** from the registry.
2. A **new container** is created with a thin writable layer on top of the image.
3. A **network interface** is configured.
4. The specified **process starts** inside the isolated environment.
5. When the process exits, the container **stops** (but is not deleted until you `docker rm`).

---

!!! success "Key takeaways"
    - Containers share the host kernel — they're lightweight isolated processes
    - Images are stacks of immutable layers; only `RUN`, `COPY`, `ADD` create layers
    - Cleanup **must** happen in the same `RUN` instruction as the install
    - One image can run as many containers; each gets its own writable layer
