# How to Run Code Review

Code review keeps Docker changes safe, readable, and production-ready before merge.

---

## 1. Open a focused pull request

Keep each PR small and scoped to one concern (for example: layer ordering, base image hardening, or multi-stage refactor).

Include in the PR description:

- What changed
- Why it changed
- How to validate it
- Risk and rollback notes (if relevant)

---

## 2. Review with a checklist

Use a repeatable checklist for every Docker-related change:

- **Correctness:** Does the image still run and expose the expected behavior?
- **Layer efficiency:** Are stable layers above volatile ones for cache reuse?
- **Image size:** Were unnecessary packages, files, and build artifacts removed?
- **Security:** Is the image using a minimal base and non-root user where possible?
- **Secrets safety:** Are `.env`, keys, and local artifacts excluded by `.dockerignore`?
- **Reproducibility:** Are base tags and key dependencies pinned where needed?

---

## 3. Validate locally before approval

Before approving, run quick checks:

1. Build the image
2. Run the container
3. Smoke-test the app behavior
4. Compare image size against the previous version
5. Confirm no sensitive files are copied into the image

---

## 4. Give actionable feedback

Prefer specific, testable comments:

- Point to exact Dockerfile lines or build output
- Explain the impact (speed, size, security, reliability)
- Suggest the expected outcome after the fix

---

## 5. Re-review after updates

When the author pushes fixes:

- Re-check only changed files first
- Re-run key validations if Dockerfile or dependencies changed
- Approve only when both functionality and container quality criteria pass

---

## Quick approval rubric

Approve when all are true:

- Build succeeds
- Runtime behavior is correct
- No obvious Docker anti-patterns remain
- No security red flags in the image setup
- PR description and reasoning are clear
