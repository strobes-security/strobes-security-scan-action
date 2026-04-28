# Strobes Security Scan Action

A reusable GitHub composite action that ships every push and pull request diff to a [Strobes](https://strobes.co) AI workflow for automated security review.

```
commit / PR  ──▶  GitHub Actions  ──▶  signed webhook  ──▶  Strobes workflow  ──▶  AI task
```

## Quick start

### 1. Configure the Strobes side

Create a workflow in **Automation → Workflows** with:

- `hooks.on_inbound_webhook: true`
- `triggers.workflow_mode: 3` _(Launch AI task)_
- `triggers.launch_ai_task.initial_message`: your review prompt, e.g. _"perform secure code review on the commit code received and report any vulnerabilities identified immediately."_
- `triggers.launch_ai_task.workspace_id` _(optional)_: a workspace with GitHub credentials if you want the agent to comment on PRs / open issues.

Save the workflow and copy the **webhook URL** and **HMAC secret**.

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
