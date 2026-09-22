# Codex CI Monitor

A native Codex plugin that waits for one exact GitHub or GitLab CI target without spending model turns on polling, then queues one terminal message back to the originating Codex session with `codex queue`.

## Install

```bash
codex plugin marketplace add sabirmgd/codex-ci-monitor
codex plugin add ci-monitor@codex-ci-monitor
```

Start a new Codex thread after installation, then ask Codex to monitor a PR, merge request, GitHub Actions run, or GitLab pipeline.

## Requirements

- Codex CLI with the `codex queue` command.
- Authenticated `gh` for GitHub targets or `glab` for GitLab targets.
- A credential-free `https://github.com/...` or `https://gitlab.com/...` target URL.
- The exact expected 40- or 64-character lowercase commit SHA.

## How it works

1. Checks the exact target and commit once.
2. If CI is pending, starts one provider-polling process in a managed Codex tool cell.
3. Uses no model turns while CI remains pending.
4. Revalidates the exact head at terminal state.
5. Queues one idempotent terminal event to the original Codex session.

The monitor is read-only. It does not merge, retry, approve, cancel, or otherwise mutate CI.

## Supported targets

- `github.com` pull requests and Actions runs.
- `gitlab.com` merge requests and pipelines.

Self-hosted GitHub Enterprise and GitLab instances are intentionally unsupported until their hostname and authentication routing are validated.

## Development

```bash
python3 plugins/ci-monitor/skills/ci-monitor/scripts/test_monitor_ci.py
python3 ~/.codex/skills/.system/skill-creator/scripts/quick_validate.py \
  plugins/ci-monitor/skills/ci-monitor
```

The test suite covers exact-head validation, terminal head rechecks, provider failures, credential-free URL enforcement, prompt-injection containment, delivery idempotency, queue timeouts, missing binaries, lock recovery, and failure receipts.

## License

MIT
