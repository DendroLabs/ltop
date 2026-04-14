# ltop

A `top`-like terminal display for **local LLM processes** and **Claude Code sessions**, in a single stdlib-only Python file.

```
 ltop  |  4 processes  |  CPU: 38.2%  |  GPU: 47%  |  MEM: 7.0G  |  14:22:07

    PID   CPU%   MEM%     RSS     ELAPSED   TOKENS   TYPE                  DETAILS
  92046   17.0    0.9    604M    01:28:18     124K █ Claude Code           ~/code/api-server [opus-4.6]  [##::--------] 2/8 Writing integration tests  (3 agents)
          +- TASK [done] Scaffolded /v1/users endpoint
          +- TASK [done] Wired Postgres fixture into pytest
          +- TASK [ >> ] Writing integration tests
          +- TASK [    ] Add rate-limit middleware
          +- TASK [    ] Hook /v1/users into OpenAPI spec
          +- SUB  [Explore] Map existing auth middleware usage  (00:00:42)
          +-agent 4f9a1b27  [###:------] 1/3 Rewriting conftest
                                                      [done] Delete stale pg_dump files
                                                      [ >> ] Rewrite conftest to use session scope
                                                      [    ] Verify xdist workers pass
  89639   11.1    1.1    728M    04:08:10      82K █ Claude Code           ~/code/web-client [sonnet-4.5]  [####::------] 3/8 Wiring up auth provider
          +- TASK [done] Pick OIDC library
          +- TASK [done] Stub /login route
          +- TASK [done] Stub /callback route
          +- TASK [ >> ] Wiring up auth provider
          +- TASK [    ] Persist tokens to IndexedDB
  10816    2.6    2.3    1.4G  01-07:19:20      18K █ Claude Code           ~/code/cli-tool [opus-4.6]
  14089    0.1    6.7    4.3G  01-02:29:44       -  █ Claude Code           ~/code/data-pipeline [opus-4.6]   (idle)

 LED: █ tokens flowing   █ idle
 q:quit  p:cpu m:mem t:time k:tok  space:pause  sort:CPU  interval:3s
```

A few things worth pointing out in that snapshot (they're easier to spot live, where color and blink do the work):

- **TOKENS column** — current context size on the most recent assistant turn, formatted like `124K` / `1.2M`. Rows that don't map to a Claude session show `-`. Sortable with `k`.
- **LED** between TOKENS and TYPE — blinks green when the token count just changed (tokens flowing), steady white otherwise. The idle `14089` row is also rendered dimmed in the TUI.
- **Task progress bar** — `[##::--------]` = 2 done (`#`), 1 in-progress (`:`), 9 pending (`-`). The `2/8 Writing integration tests` after it is `done/total` plus the active task's `activeForm`.
- **Expanded task list** — every TodoWrite task for that session is shown below its process with `[done]` / `[ >> ]` / `[    ]` markers.
- **`+- SUB` line** — a sub-agent that the main session has invoked and is currently waiting on, parsed from the transcript (`tool_use` with no matching `tool_result` yet). Shows `subagent_type`, its `description`, and how long it's been running.
- **`+-agent <id>` block** — a sub-agent's own todo list (from `~/.claude/todos/<sessionId>-agent-*.json`) with its own mini progress bar and the individual todos indented underneath.
- **`(3 agents)` tag** — the main session plus any sub-agents with their own persisted todo files.

## Why another htop?

Existing tools cover one half of the picture:

- **htop / bottom / tiptop** show every process but know nothing about LLMs.
- **llmtop** (different project) scrapes Prometheus metrics from vLLM/SGLang clusters — great for servers, irrelevant on a laptop.
- **claude-dashboard / ccboard / agents-observe** monitor Claude Code sessions but ignore the rest of your local inference stack.

`ltop` sits in the middle: one screen that shows **Ollama, llama.cpp, MLX, vLLM, LM Studio, Claude Code, Aider, diffusion, whisper** — and enriches Claude Code rows with model, cwd, subagent tree, and idle/active state.

## Features

- **htop-style live view** of local LLM-adjacent processes with CPU%, MEM%, RSS, elapsed time.
- **Claude Code session introspection** — resolves PID → session, shows cwd, model id (e.g. `opus-4.6`), task list with progress bar, subagent tree with per-agent todos.
- **opencode session introspection** — resolves PID → cwd → session via opencode's SQLite store (`~/.local/share/opencode/opencode.db`), shows the same TOKENS column and `[model]` tag. Works regardless of the underlying provider (Anthropic direct, GitHub Copilot, etc.).
- **Idle/active dimming** — Claude and opencode sessions whose transcripts haven't been written in the last 60s render dimmed, so the session that's actually working right now jumps out.
- **macOS GPU utilization** in the title bar, sudoless, via `ioreg`.
- **Sort toggles** — `p` CPU, `m` memory, `t` elapsed time. **Pause** with space.
- **Ollama loaded-model banner** via `ollama ps`.
- **Zero dependencies** — single Python file, Python 3.10+ stdlib only. No `pip install`, no build, no config.

## Install

`ltop` is a single Python file. "Installing" means downloading it to a directory on your `PATH` and marking it executable. Pick whichever of the below matches your OS.

### macOS (Apple Silicon)

`/opt/homebrew/bin` is already on your `PATH` and writable by your user:

```sh
curl -fsSL https://raw.githubusercontent.com/DendroLabs/ltop/main/ltop -o /opt/homebrew/bin/ltop
chmod +x /opt/homebrew/bin/ltop
```

### macOS (Intel) / Linux — system-wide

```sh
sudo curl -fsSL https://raw.githubusercontent.com/DendroLabs/ltop/main/ltop -o /usr/local/bin/ltop
sudo chmod +x /usr/local/bin/ltop
```

### Linux / macOS — user-local (no sudo)

Good if you don't have root, or you just prefer keeping stuff in your home directory:

```sh
mkdir -p ~/.local/bin
curl -fsSL https://raw.githubusercontent.com/DendroLabs/ltop/main/ltop -o ~/.local/bin/ltop
chmod +x ~/.local/bin/ltop
```

If `~/.local/bin` isn't already on your `PATH`, add it:

```sh
# bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc && source ~/.bashrc

# zsh (macOS default, many Linux distros)
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc

# fish
fish_add_path ~/.local/bin
```

### Windows

Windows needs two extra pip packages so curses and process listing work:

```powershell
pip install psutil windows-curses
```

Then download the script and put it somewhere on `PATH` (PowerShell):

```powershell
# pick a dir that's on PATH, e.g. %USERPROFILE%\bin
$dest = "$env:USERPROFILE\bin"
New-Item -ItemType Directory -Force -Path $dest | Out-Null
Invoke-WebRequest -Uri https://raw.githubusercontent.com/DendroLabs/ltop/main/ltop `
                  -OutFile "$dest\ltop.py"
```

Add the dir to your user `PATH` once (takes effect in new shells):

```powershell
[Environment]::SetEnvironmentVariable(
    "PATH",
    "$env:PATH;$env:USERPROFILE\bin",
    [EnvironmentVariableTarget]::User)
```

Then run with `python ltop.py` (or create a one-line `ltop.cmd` shim:  
`@python "%USERPROFILE%\bin\ltop.py" %*`).

### Verify

```sh
ltop --help
ltop --once
```

`--once` prints a single snapshot and exits — handy for confirming everything works without dropping into the full TUI.

## Usage

```
ltop                 # live TUI (default refresh: 3s)
ltop -n1             # 1s refresh
ltop --interval=5    # 5s refresh
ltop --once          # print once and exit (scriptable)
ltop --help
```

### Keys

| Key     | Action                    |
|---------|---------------------------|
| `q` / Esc | Quit                    |
| `p`     | Sort by CPU               |
| `m`     | Sort by memory            |
| `t`     | Sort by elapsed time      |
| `k`     | Sort by token count       |
| `space` | Pause / resume refresh    |

## What it recognizes

Any process matching one of the built-in patterns gets a type label and a color:

- Claude Code, opencode, Aider, Copilot (`gh copilot`), Continue
- Ollama (server, model, inference), llama.cpp (server, cli, main+gguf)
- vLLM, MLX LM, LM Studio, TGI, KoboldCpp, LocalAI, TabbyAPI
- Whisper, Stable Diffusion / ComfyUI

Patterns live in `LLM_PATTERNS` at the top of the script — easy to add your own.

## Claude Code enrichment

If a process is Claude Code, `ltop` cross-references:

- `~/.claude/sessions/<pid>.json` → cwd, sessionId
- `~/.claude/projects/<encoded-cwd>/<sessionId>.jsonl` → most recent `message.model`, last-activity mtime
- `~/.claude/tasks/<sessionId>/*.json` → main task list with progress
- `~/.claude/todos/<sessionId>-agent-*.json` → subagent todo trees

Nothing to configure — if the session files exist, they show up.

## Requirements

- Python 3.10+ (uses `str | None` unions)
- A POSIX `ps` with `-eo pid,ppid,%cpu,%mem,rss,etime,command` — **or** `psutil` (required on Windows, optional elsewhere)
- macOS for GPU display (Linux/NVIDIA users: see below)

## Platform notes

- **macOS**: fully supported. GPU utilization comes from `ioreg -c IOAccelerator` (sudoless).
- **Linux**: processes and Claude Code enrichment work. The GPU title-bar segment will be blank — patches welcome to parse `nvidia-smi` or ROCm equivalents.
- **Windows**: supported via `psutil` + `windows-curses` (see install section). `ps` isn't used on Windows — process enumeration falls back to `psutil`. GPU title-bar segment is blank.

## Related projects

- [llmtop](https://news.ycombinator.com/item?id=47421736) — htop for vLLM/SGLang/LMCache inference clusters. Different niche.
- [tokentop](https://github.com/tokentopapp/tokentop) — htop for LLM token usage and spending.
- [ccboard](https://github.com/FlorianBruniaux/ccboard) — Rust TUI + web dashboard focused on Claude Code.
- [claude-esp](https://github.com/phiat/claude-esp) — streams Claude Code's hidden output to a second terminal.
- [agents-observe](https://github.com/simple10/agents-observe) — web UI for Claude Code agent hierarchy.

## License

MIT
