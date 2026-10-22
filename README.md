# omarchy-agent-usage-opencode

A drop-in collector for the Omarchy agents panel that shows **OpenCode** usage stats in the taskbar — token counts, per-model breakdown, and a 7-day activity chart — alongside Claude Code, Codex, and Fireworks.

## What it does

When installed, a new **OpenCode** tab appears in the Omarchy agents panel (the same panel that already shows Claude, Codex, and Fireworks). The panel auto-discovers the agent from a JSON record written to `~/.local/state/omarchy/agents/usage/opencode.json`, so no bar or QML changes are needed.

The collector reads per-message token data from OpenCode's SQLite database (`~/.local/share/opencode/opencode.db`) and aggregates:

- Today's prompt count, session count, and total tokens
- A 7-day token-usage chart
- An all-time breakdown by model (input, output, reasoning, cache read/write)
- Active days and session history

OpenCode has no public rate-limit or billing API, so the limits and balance sections are omitted — the panel hides them automatically.

## Installation

### Install the collector

```bash
# Place the collector on PATH (already done — it's in ~/.local/bin/)
cp omarchy-agent-usage-opencode ~/.local/bin/
chmod +x ~/.local/bin/omarchy-agent-usage-opencode
```

### Add icons to your cloned agents plugin

```bash
PLUGIN_DIR="$HOME/.config/omarchy/plugins/wealth.agents"
mkdir -p "$PLUGIN_DIR/assets"
cp assets/opencode.svg "$PLUGIN_DIR/assets/"
cp assets/opencode-light.svg "$PLUGIN_DIR/assets/"
```

### Generate initial state

```bash
~/.local/bin/omarchy-agent-usage-opencode --force > ~/.local/state/omarchy/agents/usage/opencode.json
```

### Restart the shell

```bash
omarchy restart shell
```

### Note on auto-refresh

The `omarchy-agent-usage-update` orchestrator only scans `/usr/share/omarchy/bin/` for collectors, so it won't auto-run the OpenCode collector on its 15-minute timer. To refresh manually, run:

```bash
~/.local/bin/omarchy-agent-usage-opencode --force > ~/.local/state/omarchy/agents/usage/opencode.json
```

Or open the agents panel and press `r` (shows cached data; the panel timer doesn't re-scan Opencode automatically).

To install system-wide (so the orchestrator picks it up):

```bash
sudo cp omarchy-agent-usage-opencode /usr/share/omarchy/bin/
sudo chmod +x /usr/share/omarchy/bin/omarchy-agent-usage-opencode
```

## How it works

1. `omarchy-agent-usage-update` runs all collectors in parallel, including the new `opencode` one.
2. The collector queries `session_message` in the opencode database (read-only) and writes a JSON record to `~/.local/state/omarchy/agents/usage/opencode.json`.
3. The agents panel watches that directory; when the file appears, the OpenCode tab shows up automatically.
4. Results are cached in `~/.cache/omarchy/agent-usage/` for 20 seconds to avoid repeated DB scans.

## CLI flags

| `--force` | Invalidate cache and re-scan the full database |
| `--limits-only` | Reuse recent scan results (no limits to fetch for OpenCode) |
| `--cache-seconds N` | Override cache freshness window (default: 20) |

The `omarchy-agent-usage-update` orchestrator only scans `/usr/share/omarchy/bin/`, so user-installed collectors are not auto-discovered. Install to `/usr/share/omarchy/bin/` with `sudo` for full auto-refresh integration, or refresh manually with `~/.local/bin/omarchy-agent-usage-opencode --force`.

## Icon

Two SVG variants are provided so the mark looks good on both dark and light bar themes:

- `assets/opencode.svg` — dark background, white mark (for light bar surfaces)
- `assets/opencode-light.svg` — light background, dark mark (for dark bar surfaces)

Both are based on OpenCode's brand geometry (the interlocking "OC" spiral) and use a transparent background.

## License

MIT — see [LICENSE](./LICENSE).
