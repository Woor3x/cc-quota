# cc-quota

Personal token-quota tracker and auto-stop gate for [Claude Code](https://code.claude.com).

Watches your local Claude Code session JSONL files, aggregates token usage from local-midnight today to now, and **blocks new prompts** once you cross a daily USD threshold you set yourself. No backend, no telemetry — everything runs on your machine.

## Features

- 💰 **USD-based limits** — set thresholds in dollars. Per-model pricing (Opus / Sonnet / Haiku 3, 3.5, 3.7, 4, 4.5, 4.6, 4.7) covering input, output, cache_creation (5m & 1h), cache_read. ccusage-compatible dedup by `messageId+requestId`.
- 📊 **Statusline** — always-visible today's cost bar (no LLM, no token cost)
- ⚡ **Zero-LLM `!quota` prompts** — type `!quota set daily_usd=50` (no leading `/`) right in the CC chat. The UserPromptSubmit hook intercepts, runs the command, prints the result, and blocks the prompt before it ever reaches the model. No tokens spent.
- 🛑 Auto-stop: when usage ≥ your limit, the next prompt is blocked
- 🚪 Override hatch: `CC_QUOTA_OVERRIDE=1 claude` for one-off bypass
- 🐚 Pure shell path: `quota-cli status` works directly from terminal — zero LLM tokens

**Cost calculation**: USD across all token types (input + output + cache_creation_5m/1h + cache_read), priced per model. Duplicates from resumed sessions are deduplicated by `messageId+requestId` (same as ccusage).

## Requirements

- Claude Code with plugin support
- Python 3.8+ available as `python3`

## Install

### From GitHub

In a Claude Code session:

```text
/plugin marketplace add Woor3x/cc-quota
/plugin install cc-quota@cc-quota-marketplace
/reload-plugins
```

### Local development

```bash
git clone https://github.com/Woor3x/cc-quota.git
# then in CC:
/plugin marketplace add /path/to/cc-quota
/plugin install cc-quota@cc-quota-marketplace
```

After install, run `/reload-plugins` if needed.

## Configure

Threshold lives at `~/.claude/cc-quota/config.json`. Set it three ways:

**A. Zero-LLM (recommended):** type directly in the CC chat:

```
!quota set daily_usd=50 warn_at_pct=70 on_exceed=block
```

The `!quota` prefix is intercepted by the UserPromptSubmit hook. The command executes locally, the result is shown, and the prompt is **never sent to the model**.

**B. Pure shell (zero LLM, outside CC):**

```bash
quota-cli set daily_usd=50
```

Available keys:

| Key            | Type         | Meaning                                       |
| :------------- | :----------- | :-------------------------------------------- |
| `daily_usd`    | float / null | USD cap for today (local-midnight → now)      |
| `warn_at_pct`  | int          | Soft warning threshold (default 70)           |
| `on_exceed`    | str          | `block` or `warn`                             |

Set a key to `null` to disable. With **no config file**, the plugin does nothing.

## Use

### Statusline (recommended — always visible, zero LLM cost)

Add this to `~/.claude/settings.json`:

```json
{
  "statusLine": {
    "type": "command",
    "command": "bash \"$HOME/.claude/plugins/cache/cc-quota-marketplace/cc-quota/0.1.0/bin/quota-statusline\""
  }
}
```

(Adjust the path to wherever CC installed the plugin — check `~/.claude/plugins/installed_plugins.json` for the exact `installPath`.)

Or, for a clone you point at directly:

```json
{
  "statusLine": {
    "type": "command",
    "command": "bash \"/abs/path/to/cc-quota/bin/quota-statusline\""
  }
}
```

You'll see a line like:

```
💰 today ██▊─────────── $19.21/$100.0 19%
```

Refreshes on every CC statusline tick. A 5s on-disk cache keeps it fast.

### `!quota` prompt prefix (zero LLM, in CC chat)

```
!quota status
!quota set daily_usd=50
!quota set on_exceed=warn
```

The UserPromptSubmit hook intercepts the prefix, runs the CLI in-process, and blocks the prompt with the result as `reason`. No tokens spent.

> Slash commands (`/cc-quota:...`) were removed because CC resolves them before
> the hook fires, which forces them through the LLM. Use `!quota` instead.

### Pure shell (zero LLM, outside CC)

`bin/` is on PATH while the plugin is enabled, so you can also run:

```bash
quota-cli status
quota-cli set daily_usd=50
quota-cli used --raw
```

Sample `status` output:

```
cc-quota status
──────────────────────────────────────────────────────────────────────
today          $19.21 /     $100.0  ( 19.2%)  ████░░░░░░░░░░░░░░░░  ok
policy: on_exceed=block, warn_at=70%
config: /home/you/.claude/cc-quota/config.json
```

When you cross the limit, the next prompt is blocked:

```
🛑 cc-quota: limit exceeded — today $101.20/$100.00 (101.2%). Blocked.
   Raise limit with !quota set ... or wait until tomorrow (local midnight).
   Override once: CC_QUOTA_OVERRIDE=1 claude
```

## How it works

- A `UserPromptSubmit` hook calls `bin/quota-check` before every prompt.
- `quota-check` invokes `bin/quota-cli hook`, which scans `~/.claude/projects/*/*.jsonl` and aggregates `usage` fields whose `timestamp` falls between today's local midnight and now.
- If the limit is crossed, the hook emits `{"decision":"block","reason":...}` JSON and Claude Code shows the message instead of sending the prompt.

No network calls. No external Python deps.

## Caveats

- Counts only what's recorded in your **local** session JSONL files. If you run Claude Code on multiple machines under the same account, each machine sees only its own slice.
- Window is **calendar day in local TZ** (`midnight local → now`), matching `ccusage`'s daily bucket. Resets at local midnight.

## Layout

```
cc-quota/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── hooks/
│   └── hooks.json
├── bin/
│   ├── quota-check        # bash hook, emits JSON decision
│   ├── quota-cli          # python CLI, stdlib only
│   └── quota-statusline   # bash + python, single-line summary for statusLine
└── README.md
```

## License

MIT
