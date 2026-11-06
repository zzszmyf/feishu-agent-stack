# feishu-agent-stack

[![cc-session CI](https://github.com/zzszmyf/cc-session/actions/workflows/ci.yml/badge.svg)](https://github.com/zzszmyf/cc-session/actions/workflows/ci.yml)
[![feishu-agent-scheduler CI](https://github.com/zzszmyf/feishu-agent-scheduler/actions/workflows/ci.yml/badge.svg)](https://github.com/zzszmyf/feishu-agent-scheduler/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

Run coding agents from Feishu/Lark: persistent interactive sessions, scheduled jobs, and managed agent sandboxes.

This repository is an index for a small, self-hosted toolchain. The components are independent — you can use one without the others — but they share the same design goals:

- The agent runs on your own machine, not inside a chat platform.
- Feishu is the interface, not the runtime.
- State lives in files and systemd units before it lives in a queue or a database.
- Notifications are best-effort; logs are the source of truth.

## Components

| Project | What it does | Use it when |
|---|---|---|
| **[lark-channel-bridge](https://github.com/zarazhangrui/lark-coding-agent-bridge)** (upstream) | Chat front end: bridges Feishu messages and streaming cards to a local Claude Code or Codex process, with sessions, tool progress, and group/topic support. | You want to talk to the agent in Feishu instead of a terminal. |
| **[cc-session](https://github.com/zzszmyf/cc-session)** | Runs interactive Claude Code sessions inside tmux, with attach/detach, `send`, logs, and an optional `ttyd` web terminal. | You want a long-running agent that survives SSH disconnects, or you want to check on it from a phone. |
| **[feishu-agent-scheduler](https://github.com/zzszmyf/feishu-agent-scheduler)** | Cron for coding agents: systemd user timers run `claude -p` in a workspace and push the result to Feishu with `lark-cli`. | You want a daily health check, report, or other non-interactive agent job delivered as a chat message. |
| **[pi-managed-agents](https://github.com/zzszmyf/pi-managed-agents)** | Reference implementation for managed agent sessions: harness, sandbox lifecycle, routines runner, and a Claude-compatible bridge adapter. | You are building a multi-session or multi-tenant agent service rather than driving a single local CLI. |

`lark-channel-bridge` is a third-party MIT project maintained upstream. It is included here because it is the chat entry point most users pair with the rest of the stack. The stack author also maintains a fork with a fix for progress streams that stall at completion; the upstream PR is [#347](https://github.com/zarazhangrui/lark-coding-agent-bridge/pull/347).

## Architecture

```text
Feishu / Lark
   │  messages, mentions, streaming cards
   ▼
lark-channel-bridge  (chat front end, third-party, MIT)
   │
   ├── cc-session ............ interactive tmux sessions (+ ttyd for mobile)
   ├── feishu-agent-scheduler  systemd timer → claude -p → lark-cli push
   └── pi-managed-agents ..... programmatic sessions, sandboxes, routines
```

The four projects do not call each other. They solve different layers of the same problem, and you can mix them with any other transport or runner.

## Which one do I need?

| I want to… | Start with |
|---|---|
| Keep an interactive agent alive after my SSH connection drops | `cc-session` |
| Open the same session from my phone | `cc-session web` + the nginx example |
| Run an agent every morning and get the result in Feishu | `feishu-agent-scheduler` |
| Chat with an agent in Feishu with streaming progress cards | `lark-channel-bridge` |
| Build a service that manages many agent sessions and sandboxes | `pi-managed-agents` |

## Demos

| cc-session | feishu-agent-scheduler |
|---|---|
| [![cc-session demo](https://raw.githubusercontent.com/zzszmyf/cc-session/main/docs/demo.gif)](https://github.com/zzszmyf/cc-session) | [![feishu-agent-scheduler demo](https://raw.githubusercontent.com/zzszmyf/feishu-agent-scheduler/main/docs/demo.gif)](https://github.com/zzszmyf/feishu-agent-scheduler) |

## Quick start

### 1. Talk to an agent in Feishu

```bash
npm i -g lark-channel-bridge
lark-channel-bridge run
```

The first run prints a QR code: scan it with Feishu/Lark, create or pick the bot app, and choose Claude Code or Codex. Once messaging works, stop the foreground process and run `lark-channel-bridge start` to install the background service. See the [upstream README](https://github.com/zarazhangrui/lark-coding-agent-bridge) for profiles, groups, and configuration.

### 2. An interactive session that survives disconnects

```bash
git clone https://github.com/zzszmyf/cc-session.git
install -m 755 cc-session/bin/cc-session ~/.local/bin/

cc-session start my-task "inspect this repo and propose a fix" ~/code/my-repo
cc-session attach my-task      # Ctrl-b d to detach; the agent keeps running
cc-session send my-task "continue"
```

### 3. A scheduled job that reports to Feishu

```bash
git clone https://github.com/zzszmyf/feishu-agent-scheduler.git
install -m 755 feishu-agent-scheduler/bin/feishu-agent-schedule ~/.local/bin/
install -m 755 feishu-agent-scheduler/bin/feishu-agent-run-job ~/.local/bin/

feishu-agent-schedule add daily-health \
  --calendar "*-*-* 09:00:00" \
  --prompt-file examples/jobs/daily-health.prompt.txt \
  --target user:ou_xxxxxxxxxxxxxxxx \
  --tools "Bash Read Grep Glob" \
  --push always
```

Normal runs can return `[SILENT]` to suppress the push; failures are delivered with the job log.

### 4. Managed sessions and sandboxes

See [`pi-managed-agents`](https://github.com/zzszmyf/pi-managed-agents) for the session store, harness abstraction, sandbox lifecycle, and routines runner.

## Design notes

- **Keep the agent outside the chat transport.** A dropped WebSocket should not kill a running task. `cc-session` uses tmux; the scheduler uses detached systemd units.
- **Push the result, not a dashboard.** The chat message is the notification. Logs stay on disk for debugging.
- **Files first.** Job definitions are env files, schedules are systemd timers, logs are plain files. Add a database only when the workload actually needs one.
- **Idempotency at the edge.** Scheduled pushes carry an idempotency key, and `flock` prevents overlapping runs of the same job.
- **Treat the agent as a remote shell.** Every tool allow-list and permission mode is a security boundary.

## Security

- Do not expose `ttyd` or an agent web UI without authentication and TLS.
- Default to read-only tool allow-lists for scheduled jobs.
- `bypassPermissions` / `--tools bypass` gives the agent full access to the host; use it only in a disposable environment.
- Session logs, job prompts, and Feishu targets can contain internal hostnames, tokens, or customer data. Keep them out of public repositories.

## License

MIT. Each linked project carries its own license; `lark-channel-bridge` is MIT and maintained upstream.
