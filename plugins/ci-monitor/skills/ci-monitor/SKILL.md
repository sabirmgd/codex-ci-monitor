---
name: ci-monitor
description: Monitor one exact github.com pull request or Actions run, or one exact gitlab.com merge request or pipeline, without model polling; queue one terminal message to the originating Codex session so it continues automatically. Use when CI is the next external gate after a push, PR, MR, or manual workflow dispatch.
---

# CI Monitor

Use the bundled watcher as the only waiting loop. It polls the provider without model turns, validates the expected commit, then runs `codex queue --thread <SESSION_ID> --message <EVENT>` once. Queueing is the native active-session path; never use `codex exec resume`, which conflicts with the session writer.

Never foreground-poll CI, wait on the background cell, or rely on `notify()` to wake Codex.

## Before arming

Resolve all of these first:

- Exact PR, MR, Actions-run, or GitLab-pipeline URL.
- Exact expected head SHA.
- Concrete next action after success or failure.
- Authenticated `gh` or `glab`, as appropriate.
- A non-empty `CODEX_SESSION_ID` (or `CODEX_THREAD_ID`) and a `codex` executable whose `queue` command is available.

Only credential-free `https://github.com/...` and `https://gitlab.com/...` targets are supported. Self-hosted GitHub Enterprise and GitLab are intentionally unsupported until their authentication and hostname routing are tested.

Before first use on a Codex host, run one live acceptance: queue a uniquely marked message to the current active thread, end the turn, and require the next turn to begin with that marker. A throwaway `codex exec` session is not sufficient proof because it releases its writer after exit.

Run one read-only check with `scripts/monitor_ci.py <url> --expected-head <sha> --once`. If access, URL parsing, or head validation fails, report it and do not arm the monitor.

If the check is already terminal, continue immediately in the current turn. Do not start a watcher.

## Arm once

Start the watcher inside a yielded `functions.exec` cell so its managed shell session survives the current turn. Resolve `scripts/monitor_ci.py` relative to this `SKILL.md`; do not hardcode a user's home directory.

```javascript
const quote = value => `'${value.replaceAll("'", `'"'"'`)}'`;
const script = "<absolute path to this skill>/scripts/monitor_ci.py";
const target = "<exact PR, MR, Actions-run, or GitLab-pipeline URL>";
const head = "<exact expected SHA>";
const nextAction = "<one concrete success/failure continuation>";

const run = await tools.exec_command({
  cmd: `python3 ${quote(script)} ${quote(target)} --expected-head ${quote(head)} --next-action ${quote(nextAction)}`,
  yield_time_ms: 1000,
  max_output_tokens: 2000,
});

if (!run.session_id) {
  text(run.output);
  throw new Error("CI monitor exited before it could be armed");
}

text(JSON.stringify({ state: "monitor-armed", target, expected_head: head, session_id: run.session_id }));
yield_control();
```

After the cell yields, end the turn with one short monitor-armed handoff. Do not call `functions.wait`, `write_stdin`, or read CI again.

At terminal state the helper:

1. Rechecks the exact head and run.
2. Creates a deterministic event ID.
3. Suppresses duplicate delivery with a local receipt.
4. Queues the terminal result and next action to the recorded active Codex session.
5. Leaves the provider unchanged; it never merges, retries, cancels, or approves CI.

On the queued continuation turn, verify the terminal state once, continue the pending workflow, and do not re-arm the completed run.

## Fail closed

Do not claim autonomous continuation when any prerequisite is missing. A desktop notification, WhatsApp message, detached process, cron job, or buffered tool output is not a thread wake-up.

The helper stores mode-`0600` delivery receipts under `$XDG_CACHE_HOME/codex-ci-monitor/` or `~/.cache/codex-ci-monitor/`. It rejects target URLs containing credentials, queries, or fragments. If Codex cannot queue the message after bounded retries, it writes a failure receipt and attempts a local desktop notification; that alert is failure evidence, not a successful continuation.
