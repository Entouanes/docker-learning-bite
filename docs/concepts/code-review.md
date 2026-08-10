# How to Use GitHub Copilot Code Review

GitHub Copilot Code Review helps you catch issues faster by adding AI feedback directly to your pull request flow.

---

## 1. Open a focused pull request

Copilot gives better feedback when your PR is small and scoped to one topic.

For strong results, include:

- A clear summary of what changed
- Why the change was needed
- How reviewers can validate it
- Any known tradeoffs or risks

---

## 2. Start Copilot Code Review on the PR

From the pull request page:

1. Open your PR in GitHub
2. In the **Reviewers** panel, request **Copilot** as a reviewer (or use the PR review action menu and select **Copilot**)
3. Wait for Copilot to post review comments on changed files

Copilot reviews diff context, flags likely issues, and proposes concrete follow-ups.

---

## 3. Triage Copilot comments

Treat Copilot output like a fast first-pass reviewer:

- **Accept** comments that point to real defects or risky patterns
- **Question** comments that lack context
- **Dismiss** comments that are not applicable to your architecture

Always make the final decision with human judgment.

---

## 4. Apply fixes and re-run review

After you push updates:

- Re-run Copilot Code Review on the PR
- Confirm previous issues are resolved
- Check for newly introduced concerns in the updated diff

Iterate until comments are either resolved or intentionally dismissed with rationale.

---

## 5. Combine AI review with Docker checks

Copilot should complement, not replace, project checks:

- Build the image
- Run container smoke tests
- Verify layer ordering and image size impact
- Confirm non-root and `.dockerignore` safety expectations

Use Copilot for speed, and validation steps for confidence.

---

## Quick workflow checklist

- PR is focused and well-described
- Copilot Code Review has run on the latest commit
- Valid Copilot findings are fixed
- Remaining Copilot comments are addressed or dismissed with reason
- Local/CI validation still passes
