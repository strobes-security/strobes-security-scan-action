# Strobes Security Scan Action

A reusable GitHub composite action that ships every push and pull request diff to a [Strobes](https://strobes.co) AI workflow for automated security review.

```
commit / PR  ──▶  GitHub Actions  ──▶  signed webhook  ──▶  Strobes workflow  ──▶  AI task
```

## Quick start

To integrate SAST scanning into your CI/CD pipeline, we use the Strobes **Automation** + **AI Workspace** features. Set up the Strobes side first, then drop the GitHub Action into your repo.

### 1. Set up the Strobes side

#### a. Create the AI Workspace

The AI Workspace is the long-lived context that scan agents run inside. It holds the standing review instructions and the credentials the agent uses to clone your repo.

1. Log in to Strobes and navigate to **AI → Workspaces**.
2. Click **Blank workspace**.
3. Provide a workspace description — this becomes the workspace's standing system prompt. For example:

   > Set up a CI/CD pipeline integration where incoming PRs are scanned for potential vulnerabilities. Review the code changes carefully and use the credentials attached to this workspace to clone the repository locally and perform an in-depth review when needed to verify whether the vulnerabilities in the PR can affect the system, then report findings. This workspace is used to spin up new agent tasks to scan PRs as they arrive.

4. Save. The workspace is now created.
5. Open the workspace's **Settings** and attach the **GitHub credentials** the agent will use to clone and review the target repo.

#### b. Create the Automation

The Automation receives the inbound webhook from GitHub Actions and dispatches a scan task into the workspace you just created.

1. Navigate to **Automation → New Automation → Create Manually**.
2. **Select the asset** that represents your repository.
3. Enable **Allow multiple actions to perform**.
4. Choose **Webhook** as the trigger.
5. Under **Filters**, select the asset you want this automation scoped to (the target asset created for your repo).
6. Under **Actions**, select **Launch AI task**, then click into it and configure:
   - **Workspace** — pick the workspace you just created from the dropdown.
   - **Prompt** — the per-task instruction. Keep it tightly scoped to the PR diff so the agent doesn't waste tokens re-scanning unchanged code. For example:

     > Review **only** the code changes in this incoming PR / commit (the unified diff in the payload). Identify and report any security vulnerabilities introduced or exposed by these changes. Stay strictly within the PR scope — examine the changed files, and consult related files in the cloned repo only when needed to verify exploitability of a finding. Do **not** perform a full-repository scan; this is incremental, per-PR review, not a re-scan of unchanged code. For each finding, include severity, file path, line number, a concise impact statement, and a suggested fix. Submit findings immediately on the platform.

7. Save the automation.

#### c. Copy the webhook credentials

Once the automation is saved, Strobes displays a **webhook URL** and an **HMAC secret**. Copy both — you'll wire them into the GitHub repo in the next steps. The webhook is what GitHub Actions hits on every push / PR; Strobes verifies the HMAC signature and triggers a fresh agent task in your workspace.

### 2. Add the secret to your repository

```bash
gh secret set STROBES_WEBHOOK_SECRET --body "<secret-from-strobes>"
```

### 3. Drop a workflow into the consuming repo

Save as `.github/workflows/strobes-security-scan.yml`:

```yaml
name: Strobes Security Scan

on:
  push:
    branches: ['**']
  pull_request:
    types: [opened, synchronize, reopened]
  workflow_dispatch:

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: 0xavis/strobes-security-scan-action@v1
        with:
          webhook-url: https://<strobes-host>/api/v1/webhooks/receive/<uuid>
          webhook-secret: ${{ secrets.STROBES_WEBHOOK_SECRET }}
```

That's it. Every push and PR now triggers an AI security review.

To re-scan on demand: `gh workflow run "Strobes Security Scan"`.

## Inputs

| Input            | Required | Default | Description                                                                |
| ---------------- | -------- | ------- | -------------------------------------------------------------------------- |
| `webhook-url`    | yes      | —       | Strobes inbound webhook URL (`https://<host>/api/v1/webhooks/receive/<uuid>`) |
| `webhook-secret` | yes      | —       | HMAC-SHA256 secret issued by your Strobes workflow                         |
| `fail-on-error`  | no       | `true`  | Fail the job if Strobes returns a non-2xx response                         |

## Payload

```json
{
  "event": "push",
  "data": {
    "repository": "owner/repo",
    "branch": "main",
    "commit": "<sha>",
    "before": "<parent-sha>",
    "diff": "diff --git a/... b/...\n..."
  }
}
```

For pull requests, `event` is `"pull_request"` and the payload also includes `pr_number`. `before`/`commit` are the PR's base and head SHAs respectively, so the diff covers the full PR scope.

## HMAC signature

- Algorithm: **HMAC-SHA256**
- Secret: the value passed in `webhook-secret`
- Message: the **raw body bytes** that will be sent
- Header: `X-Strobes-Signature: sha256=<hex>`

The action signs the payload file with `openssl dgst` and posts it with `curl --data-binary @file`, so the bytes sent match the bytes signed. Any re-serialization invalidates the signature and Strobes returns `401`.

## Troubleshooting

| Symptom                          | Likely cause                                  | Fix                                                            |
| -------------------------------- | --------------------------------------------- | -------------------------------------------------------------- |
| `HTTP 401`                       | Wrong secret, or body re-serialized           | Re-check `STROBES_WEBHOOK_SECRET`; ensure the action posts with `--data-binary` (default behavior) |
| `HTTP 404`                       | Wrong webhook UUID                            | Copy the URL directly from the Strobes workflow page           |
| `HTTP 000` / timeout             | Receiver unreachable                          | Confirm the Strobes host is reachable from GitHub-hosted runners |
| Webhook accepted, no AI task     | Workflow not in *Launch AI task* mode         | Set `triggers.workflow_mode: 3` on the Strobes workflow        |
| Agent can't post back to GitHub  | Strobes workspace lacks GitHub credentials    | Attach a workspace with a GitHub App on the target org         |
| Diff missing or empty            | Shallow checkout                              | Use `actions/checkout@v4` with `fetch-depth: 0`                |

## Security notes

- The HMAC secret must live **only** in repository secrets — never commit it.
- Diffs may include code the developer didn't intend to publish (secrets, internal URLs). Treat the payload as sensitive.
- Rotate the HMAC secret from the Strobes workflow settings and update `STROBES_WEBHOOK_SECRET` at the same time.
- Commit content is attacker-controllable. When tuning the AI prompt, assume the diff may contain prompt-injection attempts.

## License

[MIT](LICENSE)
