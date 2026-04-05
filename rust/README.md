# 🦞 Claw Code — Rust Implementation

A high-performance Rust rewrite of the Claw Code CLI agent harness. Built for speed, safety, and native tool execution.

For a task-oriented guide with copy/paste examples, see [`../USAGE.md`](../USAGE.md).

## Quick Start

```bash
# Inspect available commands
cd rust/
cargo run -p rusty-claude-cli -- --help

# Build the workspace
cargo build --workspace

# Run the interactive REPL
cargo run -p rusty-claude-cli -- --model claude-opus-4-6

# One-shot prompt
cargo run -p rusty-claude-cli -- prompt "explain this codebase"

# JSON output for automation
cargo run -p rusty-claude-cli -- --output-format json prompt "summarize src/main.rs"

# Inspect registered hooks and whether they are enabled
cargo run -p rusty-claude-cli -- hook list
```

## Configuration

Set your API credentials:

```bash
export ANTHROPIC_API_KEY="sk-ant-..."
# Or use a proxy
export ANTHROPIC_BASE_URL="https://your-proxy.com"
```

Or point the CLI at an OpenAI-compatible endpoint:

```bash
export OPENAI_API_KEY="local-or-real-token"
export OPENAI_BASE_URL="http://127.0.0.1:8000/v1"
```

Or authenticate via OAuth and let the CLI persist credentials locally:

```bash
cargo run -p rusty-claude-cli -- login
```

`claw login` is Claude/Anthropic OAuth only. For local models such as Qwen, use the OpenAI-compatible environment variables instead.

## One-shot local Qwen 0.6B test run

This is the fastest copy-paste path for trying the Rust CLI against a local Qwen model through vLLM's OpenAI-compatible server. It assumes Linux, an installed Rust toolchain, and a machine where vLLM can run.

```bash
set -euo pipefail

# Install uv if needed
if ! command -v uv >/dev/null 2>&1; then
  curl -LsSf https://astral.sh/uv/install.sh | sh
  export PATH="$HOME/.local/bin:$PATH"
else
  export PATH="$HOME/.local/bin:$PATH"
fi

# Pick the model you want to test. Replace this if your server/model id differs.
MODEL_ID="${MODEL_ID:-Qwen/Qwen3-0.6B}"
API_KEY="${API_KEY:-local-qwen}"
VLLM_PORT="${VLLM_PORT:-8000}"
WORK_DIR="${WORK_DIR:-$HOME/qwen-test}"

mkdir -p "$WORK_DIR"
cd "$WORK_DIR"

# Create a uv-managed Python environment and install vLLM
uv python install 3.11
uv venv --python 3.11 .venv

# Verify the environment exists where you expect it to
ls -la
test -f .venv/bin/activate && echo "venv ready: $PWD/.venv"

# Activate it before installing packages or starting vLLM
. .venv/bin/activate
uv pip install vllm

# Start an OpenAI-compatible local server in the background.
# These flags matter for Claw's prompt envelope:
# - --enable-auto-tool-choice and --tool-call-parser hermes are required because
#   the CLI sends tool definitions with tool_choice=auto
# - --enforce-eager avoids the torch.compile/CUDAGraph startup path that was flaky
#   in local WSL testing
# - --max-model-len 16384 gives the CLI's large system prompt and tool schema enough room
vllm serve "$MODEL_ID" \
  --host 127.0.0.1 \
  --port "$VLLM_PORT" \
  --api-key "$API_KEY" \
  --gpu-memory-utilization 0.55 \
  --max-model-len 16384 \
  --enforce-eager \
  --enable-auto-tool-choice \
  --tool-call-parser hermes \
  >/tmp/claw-qwen-vllm.log 2>&1 &
VLLM_PID=$!
trap 'kill "$VLLM_PID" 2>/dev/null || true' EXIT

# Wait for the server to come up and capture the exact served model id
until curl -fsS "http://127.0.0.1:${VLLM_PORT}/v1/models" \
  -H "Authorization: Bearer ${API_KEY}" \
  >/tmp/claw-qwen-models.json; do
  sleep 2
done

# Build and run the Rust CLI against the local OpenAI-compatible endpoint
cd /home/toor/claw-code-parity/rust
cargo build --release

# Important: unset Anthropic credentials so Qwen routes to the OpenAI-compatible provider
unset ANTHROPIC_API_KEY
unset ANTHROPIC_AUTH_TOKEN
unset ANTHROPIC_BASE_URL

export OPENAI_API_KEY="$API_KEY"
export OPENAI_BASE_URL="http://127.0.0.1:${VLLM_PORT}/v1"

MODEL_ID="$(python3 - <<'PY'
import json
with open('/tmp/claw-qwen-models.json', 'r', encoding='utf-8') as handle:
    payload = json.load(handle)
print(payload['data'][0]['id'])
PY
)"

./target/release/claw --model "$MODEL_ID" prompt "Summarize this repository"
```

If your chosen Qwen model needs custom Hugging Face code at load time, add `--trust-remote-code` to the `vllm serve` line above.

If you reduce `--max-model-len`, keep it high enough for Claw's large system prompt and tool schema. Values like `1024` are usually too small; `8192` is a more realistic lower bound if `16384` is too ambitious for your GPU.

If you ever lose track of the environment later, the activation script should be at `~/qwen-test/.venv/bin/activate` (or `$WORK_DIR/.venv/bin/activate` if you changed `WORK_DIR`).

## Mock parity harness

The workspace now includes a deterministic Anthropic-compatible mock service and a clean-environment CLI harness for end-to-end parity checks.

```bash
cd rust/

# Run the scripted clean-environment harness
./scripts/run_mock_parity_harness.sh

# Or start the mock service manually for ad hoc CLI runs
cargo run -p mock-anthropic-service -- --bind 127.0.0.1:0
```

Harness coverage:

- `streaming_text`
- `read_file_roundtrip`
- `grep_chunk_assembly`
- `write_file_allowed`
- `write_file_denied`
- `multi_tool_turn_roundtrip`
- `bash_stdout_roundtrip`
- `bash_permission_prompt_approved`
- `bash_permission_prompt_denied`
- `plugin_tool_roundtrip`

Primary artifacts:

- `crates/mock-anthropic-service/` — reusable mock Anthropic-compatible service
- `crates/rusty-claude-cli/tests/mock_parity_harness.rs` — clean-env CLI harness
- `scripts/run_mock_parity_harness.sh` — reproducible wrapper
- `scripts/run_mock_parity_diff.py` — scenario checklist + PARITY mapping runner
- `mock_parity_scenarios.json` — scenario-to-PARITY manifest

## Features

| Feature                                           | Status         |
| ------------------------------------------------- | -------------- |
| Anthropic API + streaming                         | ✅             |
| OAuth login/logout                                | ✅             |
| Interactive REPL (rustyline)                      | ✅             |
| Tool system (bash, read, write, edit, grep, glob) | ✅             |
| Web tools (search, fetch)                         | ✅             |
| Sub-agent orchestration                           | ✅             |
| Todo tracking                                     | ✅             |
| Notebook editing                                  | ✅             |
| CLAUDE.md / project memory                        | ✅             |
| Config file hierarchy (.claude.json)              | ✅             |
| Permission system                                 | ✅             |
| MCP server lifecycle                              | ✅             |
| Session persistence + resume                      | ✅             |
| Extended thinking (thinking blocks)               | ✅             |
| Cost tracking + usage display                     | ✅             |
| Git integration                                   | ✅             |
| Markdown terminal rendering (ANSI)                | ✅             |
| Model aliases (opus/sonnet/haiku)                 | ✅             |
| Slash commands (/status, /compact, /clear, etc.)  | ✅             |
| Hooks (PreToolUse/PostToolUse)                    | 🔧 Config only |
| Plugin system                                     | 📋 Planned     |
| Skills registry                                   | 📋 Planned     |

## Model Aliases

Short names resolve to the latest model versions:

| Alias    | Resolves To                 |
| -------- | --------------------------- |
| `opus`   | `claude-opus-4-6`           |
| `sonnet` | `claude-sonnet-4-6`         |
| `haiku`  | `claude-haiku-4-5-20251213` |

## CLI Flags

```text
claw [OPTIONS] [COMMAND]

Options:
  --model MODEL                    Override the active model
  --dangerously-skip-permissions   Skip all permission checks
  --permission-mode MODE           Set read-only, workspace-write, or danger-full-access
  --allowedTools TOOLS             Restrict enabled tools
  --output-format FORMAT           Non-interactive output format (text or json)
  --resume SESSION                 Re-open a saved session or inspect it with slash commands
  --version, -V                    Print version and build information locally

Commands:
  prompt <text>      One-shot prompt (non-interactive)
  login              Authenticate via OAuth
  logout             Clear stored credentials
  init               Initialize project config
  status             Show the current workspace status snapshot
  sandbox            Show the current sandbox isolation snapshot
  agents             Inspect agent definitions
  mcp                Inspect configured MCP servers
  skills             Inspect installed skills
  system-prompt      Render the assembled system prompt
```

For the current canonical help text, run `cargo run -p rusty-claude-cli -- --help`.

## Slash Commands (REPL)

Tab completion expands slash commands, model aliases, permission modes, and recent session IDs.

| Command             | Description                               |
| ------------------- | ----------------------------------------- |
| `/help`             | Show help                                 |
| `/status`           | Show session status (model, tokens, cost) |
| `/cost`             | Show cost breakdown                       |
| `/compact`          | Compact conversation history              |
| `/clear`            | Clear conversation                        |
| `/model [name]`     | Show or switch model                      |
| `/permissions`      | Show or switch permission mode            |
| `/config [section]` | Show config (env, hooks, model)           |
| `/memory`           | Show CLAUDE.md contents                   |
| `/diff`             | Show git diff                             |
| `/export [path]`    | Export conversation                       |
| `/resume [id]`      | Resume a saved conversation               |
| `/session [id]`     | Resume a previous session                 |
| `/version`          | Show version                              |

See [`../USAGE.md`](../USAGE.md) for examples covering interactive use, JSON automation, sessions, permissions, and the mock parity harness.

## Workspace Layout

```text
rust/
├── Cargo.toml              # Workspace root
├── Cargo.lock
└── crates/
    ├── api/                # Anthropic API client + SSE streaming
    ├── commands/           # Shared slash-command registry
    ├── compat-harness/     # TS manifest extraction harness
    ├── mock-anthropic-service/ # Deterministic local Anthropic-compatible mock
    ├── plugins/            # Plugin registry and hook wiring primitives
    ├── runtime/            # Session, config, permissions, MCP, prompts
    ├── rusty-claude-cli/   # Main CLI binary (`claw`)
    ├── telemetry/          # Session tracing and usage telemetry types
    └── tools/              # Built-in tool implementations
```

### Crate Responsibilities

- **api** — HTTP client, SSE stream parser, request/response types, auth (API key + OAuth bearer)
- **commands** — Slash command definitions and help text generation
- **compat-harness** — Extracts tool/prompt manifests from upstream TS source
- **mock-anthropic-service** — Deterministic `/v1/messages` mock for CLI parity tests and local harness runs
- **plugins** — Plugin metadata, registries, and hook integration surfaces
- **runtime** — `ConversationRuntime` agentic loop, `ConfigLoader` hierarchy, `Session` persistence, permission policy, MCP client, system prompt assembly, usage tracking
- **rusty-claude-cli** — REPL, one-shot prompt, streaming display, tool call rendering, CLI argument parsing
- **telemetry** — Session trace events and supporting telemetry payloads
- **tools** — Tool specs + execution: Bash, ReadFile, WriteFile, EditFile, GlobSearch, GrepSearch, WebSearch, WebFetch, Agent, TodoWrite, NotebookEdit, Skill, ToolSearch, REPL runtimes

## Stats

- **~20K lines** of Rust
- **9 crates** in workspace
- **Binary name:** `claw`
- **Default model:** `claude-opus-4-6`
- **Default permissions:** `danger-full-access`

## License

See repository root.
