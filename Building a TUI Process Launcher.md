# Building a TUI Process Launcher

*Existing tools, framework options and recommendations for a local tool-launcher TUI*

## The idea

A TUI app configured through declarative templates that lists your local tools with their status, CPU/RAM usage and other statistics. You can start, stop, pause, enable and disable each one. Tools that run web servers automatically show their URL and port. A later nice-to-have is sharing those links over the internet with ngrok or cloudflared.

## Existing tools

Tools like this already exist, so try them before building.

- **[process-compose](https://github.com/F1bonacc1/process-compose)** (Go) is the closest to your idea. It has declarative YAML config, a TUI with process status, CPU/RAM, logs, start/stop/restart, dependencies between processes, and health checks. Think "docker-compose, but for plain processes".
- **[mprocs](https://github.com/pvolok/mprocs)** (Rust) is simpler: YAML config, a process list on the left, and live logs on the right.
- **pm2** (Node) has `pm2 monit` and an ecosystem config file. It is more oriented toward production Node apps.
- **Overmind / honcho / foreman** run Procfile-style process lists. They are lighter and less visual.
- **Tilt** is a dev-environment orchestrator with a web UI and link support, but it is heavier and Kubernetes-oriented.
- **supervisord** is the classic option, with a basic CLI and web UI.

None of them (as far as I know) does the following well: **automatic URL/port detection**, **pause/resume**, and **one-key tunnel sharing** via ngrok or cloudflared. That is where your idea could stand out.

## Frameworks for building it

- **Python: Textual.** The fastest path to a polished UI. CSS-like styling, mouse support, widgets, animations. Pair it with `psutil` for CPU/RAM. It can even serve the same app in a browser.
- **Go: Bubble Tea** (with Lip Gloss and Bubbles). Very popular, elegant, and ships as a single static binary. Great for distributing tools.
- **Rust: Ratatui.** The fastest and most flexible, but more verbose.
- **Node/TypeScript: Ink.** React for the terminal. Good if you like React.
- **Go: tview.** Higher-level widgets, less pretty by default.

For a "real nice, fluid interface" with the least effort, pick **Textual**. If you want a single binary you can ship, pick **Bubble Tea**.

## What I think of the idea

The need is real, and the pain you describe is why process-compose and mprocs exist. But the space is somewhat crowded, so I would suggest:

1. **Try process-compose for an hour.** If it covers 80% of your needs, build only the missing pieces on top, or contribute them.
2. **If you build your own, focus on the differentiators:**
   - **Port detection** by inspecting the listening sockets of the process tree (`psutil.net_connections` or `lsof`), which beats parsing stdout.
   - **Pause/resume** with `SIGSTOP` / `SIGCONT`.
   - **Tunnels:** spawn `cloudflared tunnel --url http://localhost:PORT` and parse the generated URL.
   - **Enable/disable** as persistent state, separate from running/stopped.
3. **Split it into a daemon plus a TUI client.** Your tools keep running if you close the UI, and you can later add a CLI or web frontend on the same API.

## Open question

Which language are you most comfortable with? That settles the framework choice.

## Procboard: specification

*Version 0.1, draft. Language: Go. Target platforms: Linux and macOS first.*

### 1. Overview

Procboard is a local process dashboard for the terminal. You describe your tools in a declarative YAML file, and Procboard shows them in a TUI with live status, CPU and RAM usage, logs and URLs. You start, stop, pause and enable or disable each tool with a key or a mouse click.

**Goals**

- Launch and manage many local tools from one place instead of many terminals.
- Show the state and resource usage of each tool at a glance.
- Detect the ports a tool listens on and show clickable URLs automatically.
- Keep tools running when the UI is closed.
- Feel fast, fluid and intuitive, with full keyboard and mouse support.

**Non-goals (v1)**

- Production process supervision, clustering or remote hosts.
- Container or Kubernetes orchestration.
- Windows support (pause/resume relies on POSIX signals).

### 2. Architecture

Procboard is split into three parts that share one codebase and one binary.

- **Daemon** (`procboard daemon`): owns the child processes, collects stats, detects ports, manages tunnels and persists state. It keeps running when the UI closes.
- **TUI client** (`procboard`, the default command): a Bubble Tea app that connects to the daemon and renders the board. It starts the daemon automatically if it is not running.
- **CLI client** (`procboard start api`, and so on): scriptable commands that talk to the same daemon API.

The daemon exposes a local API over a Unix socket (HTTP with JSON, plus a Server-Sent Events stream for live updates). This keeps room for a future web UI on the same API.

### 3. Configuration

Configuration is a YAML file. Procboard looks for `./procboard.yaml` first, then `~/.config/procboard/config.yaml`. The daemon reloads the file on change or on `procboard reload`.

```yaml
version: 1

defaults:
  restart: never          # never | on-failure | always
  shell: /bin/sh

tools:
  api:
    description: Backend API
    cmd: npm run dev
    dir: ~/projects/api
    env:
      PORT: "3000"
    enabled: true
    autostart: true
    group: backend
    restart: on-failure
    depends_on: [db]
    ports: [3000]          # optional: expected ports, otherwise auto-detected
    health:
      http: http://localhost:3000/health
      interval: 5s
    tunnel:
      provider: cloudflared   # cloudflared | ngrok

  db:
    description: Local Postgres
    cmd: docker compose up postgres
    dir: ~/projects/api
    group: backend
    autostart: true

  docs:
    cmd: mkdocs serve
    dir: ~/projects/docs
    enabled: false
```

**Fields**

- `cmd` (required): command line to run.
- `dir`: working directory.
- `env`: extra environment variables. `env_file` can load a dotenv file.
- `enabled`: whether the tool can be started at all (default `true`).
- `autostart`: start when the daemon starts (default `false`).
- `group`: label used to group and filter tools in the UI.
- `restart`: restart policy.
- `depends_on`: tools that must be running (and healthy, if a health check exists) first.
- `ports`: expected ports. If omitted, ports are detected automatically.
- `health`: optional `http`, `tcp` or `cmd` check with `interval` and `timeout`.
- `tunnel`: optional tunnel settings for sharing.

### 4. Process model

Each tool has two independent kinds of state.

**Persistent state:** `enabled` or `disabled`. A disabled tool is greyed out, cannot be started and is skipped by autostart and by "start all". This state survives daemon restarts.

**Runtime state:**

- `stopped`: not running.
- `starting`: launched, waiting for the process to be alive (and healthy, if configured).
- `running`: alive and healthy.
- `paused`: frozen with `SIGSTOP`.
- `stopping`: graceful shutdown in progress.
- `failed`: exited with a non-zero code, or failed its health check.

**Transitions**

- Start: `stopped` or `failed` → `starting` → `running`.
- Stop: send `SIGTERM` to the process group, wait for a grace period (default 5 s), then `SIGKILL`.
- Pause: `running` → `paused` using `SIGSTOP` on the whole process group.
- Resume: `paused` → `running` using `SIGCONT`.
- Restart: stop, then start.

Every tool runs in its own process group (`Setpgid: true`) so that signals reach child processes too.

### 5. Features

**Lifecycle control.** Start, stop, restart, pause, resume, enable and disable for a single tool, a selection, a group or all tools. Dependencies are respected when starting and stopping.

**Statistics.** Per tool: status, uptime, PID, CPU %, memory (RSS), number of child processes and restart count. Collected every second with `gopsutil`, aggregated across the whole process tree.

**Logs.** The daemon captures stdout and stderr in a ring buffer (default 10,000 lines per tool) and optionally writes them to disk. The TUI shows a live, scrollable, searchable log view for the selected tool.

**Port and URL detection.** The daemon inspects the listening sockets of each tool's process tree and lists the ports. For HTTP-like ports it shows `http://localhost:PORT`, which can be opened in the browser or copied with one key. Ports from the `ports` field are used as hints and as a fallback.

**Health checks.** Optional HTTP, TCP or command checks. A failing check marks the tool `failed` and can trigger the restart policy.

**Tunnels (phase 3).** One key starts a tunnel for the selected tool by spawning `cloudflared tunnel --url http://localhost:PORT` or the ngrok equivalent. The public URL is parsed from the output, shown in the UI and copyable. Stopping the tool also stops the tunnel. Tunnels are never started automatically, and the UI must show a clear warning that the tool becomes reachable from the internet.

**Persistence.** The daemon stores enabled/disabled state and the last known runtime state in `~/.local/state/procboard/`.

### 6. TUI design

**Layout**

- **Left or top:** the tool list, grouped by `group`, with a status icon, name, CPU, RAM, uptime and URL for each tool.
- **Right or bottom:** a detail pane for the selected tool with tabs for Logs, Info (command, directory, env, ports, health) and Tunnel.
- **Footer:** context-sensitive key hints and daemon connection status.
- **Header:** totals such as running count, total CPU and total RAM.

**Interaction**

- Full mouse support: click to select, click on action buttons, scroll logs, click a URL to open it.
- Full keyboard support with discoverable hints and a help overlay.
- Live updates through the SSE stream, with smooth, flicker-free redraws.
- Colors with a light and dark theme, and a no-color fallback.

**Default key bindings**

- `↑` `↓` or `j` `k`: move selection.
- `s`: start or stop the selected tool.
- `r`: restart.
- `p`: pause or resume.
- `e`: enable or disable.
- `o`: open the URL in the browser.
- `y`: copy the URL.
- `t`: toggle a tunnel.
- `l`: focus the logs.
- `/`: search or filter.
- `a`: start all enabled tools. `A`: stop all.
- `?`: help. `q`: quit the UI (the daemon keeps running). `Q`: quit and stop the daemon.

### 7. CLI

- `procboard`: open the TUI.
- `procboard daemon`: run the daemon in the foreground.
- `procboard up` and `procboard down`: start or stop all enabled tools.
- `procboard start|stop|restart|pause|resume <tool>`
- `procboard enable|disable <tool>`
- `procboard status`: print a table of tools, states, CPU, RAM and URLs. Supports `--json`.
- `procboard logs <tool> [-f]`
- `procboard tunnel <tool> [--stop]`
- `procboard reload`: re-read the configuration.
- `procboard init`: create a starter config file.

### 8. Daemon API (draft)

All endpoints are served over a Unix socket as JSON.

- `GET /tools`: list tools with config and current state.
- `GET /tools/{name}`: details for one tool.
- `POST /tools/{name}/start|stop|restart|pause|resume|enable|disable`
- `GET /tools/{name}/logs?follow=true`: log lines, streamed when following.
- `POST /tools/{name}/tunnel` and `DELETE /tools/{name}/tunnel`
- `GET /events`: SSE stream of state changes, stats ticks and log lines.
- `POST /reload`: reload configuration.

### 9. Technical stack

- **UI:** Bubble Tea, Bubbles and Lip Gloss.
- **CLI:** Cobra.
- **Config:** `yaml.v3`, with validation and clear error messages that include line numbers.
- **Process and stats:** `os/exec`, `syscall`, `gopsutil`.
- **Daemon API:** `net/http` over a Unix socket.
- **Release:** GoReleaser, with binaries for Linux and macOS (amd64 and arm64) and a Homebrew tap.

### 10. Non-functional requirements

- Startup of the TUI in under 200 ms when the daemon is already running.
- Daemon overhead below 1% CPU and about 30 MB RAM when idle with 20 tools.
- No orphan processes: on daemon exit, tools are stopped according to config (`on_exit: stop | keep`, default `stop`).
- Safe by default: the API socket has user-only permissions, and nothing listens on the network.
- Graceful handling of crashes, missing commands and unreadable config files, with errors shown in the UI.

### 11. Roadmap

- **Phase 1, MVP:** config loading, daemon and TUI, start/stop/restart, status, CPU and RAM, logs.
- **Phase 2:** port and URL detection, enable/disable, pause/resume, groups, dependencies, restart policies, health checks, CLI commands.
- **Phase 3:** tunnels with cloudflared and ngrok, persistent log files, themes, config hot reload.
- **Later:** web UI on the same API, per-tool metrics history and sparklines, config templates and presets, plugin hooks (notifications on crash).

### 12. Open questions

- Should the daemon be started automatically by the TUI, or managed by the user (for example through systemd or launchd)?
- Should tools be able to run inside their own terminal (PTY) for interactive programs, or is non-interactive log capture enough for v1?
- One global config file, or a per-project file discovered from the current directory, or both?
- Which tunnel provider should come first: cloudflared (free, no account for quick tunnels) or ngrok?
