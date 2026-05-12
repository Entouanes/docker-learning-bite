# Lab Overview

In this lab you'll take a simple Python web app from a bloated, fragile image to a lean, production-ready one — measuring the difference at each step so you can *see* the impact of what you've learned.

---

## What you'll build

A minimal Python Flask app. The app itself is trivial — the point is the **image**, not the application.

```
lab/
├── app.py
├── requirements.txt
└── Dockerfile          ← you'll evolve this through both parts
```

---

## Learning outcomes

By the end of the lab you will have:

- [x] Built an image and measured its size with `docker images`
- [x] Identified wasted layers and fixed them
- [x] Optimised layer ordering for fast rebuilds
- [x] Converted a single-stage Dockerfile to a multi-stage build
- [x] Seen a real before/after size comparison

---

## Prerequisites

- Docker installed and running
- A terminal and text editor
- Read the three concept pages (takes ~15 min)

---

## Setup

Create a working directory for the lab:

```bash
mkdir docker-lab && cd docker-lab
```

Create the app files:

=== "app.py"

    ```python
    from flask import Flask

    app = Flask(__name__)

    @app.route("/")
    def hello():
        return "Hello from Docker!"

    if __name__ == "__main__":
        app.run(host="0.0.0.0", port=8080)
    ```

=== "requirements.txt"

    ```
    flask==3.0.3
    gunicorn==22.0.0
    ```

---

## Lab parts

| Part | Topic | Time |
|---|---|---|
| [Part 1 · Layer Fundamentals](part1.md) | Build a bloated image, identify problems, fix them | ~15 min |
| [Part 2 · Multi-Stage Builds](part2.md) | Use a builder stage, shrink the image further | ~10 min |

[:octicons-arrow-right-24: Start Part 1](part1.md)
