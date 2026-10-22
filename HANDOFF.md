## HANDOFF — Session Summary

## What was done

Created `omarchy-agent-usage-opencode`: a collector script that adds **OpenCode** as a first-class agent in the Omarchy taskbar AI agents panel.

### Files in the repo

| File | Purpose |
|---|---|
| `omarchy-agent-usage-opencode` | Python 3 collector script |
| `assets/opencode.svg` | Official OpenCode logo (dark bg, white mark) |
| `assets/opencode-light.svg` | Official OpenCode logo (light bg, dark mark) |
| `README.md` | Installation & usage docs |
| `LICENSE` | MIT license |
| `SPEC.md` | Feature specification |
| `HANDOFF.md` | This file |

### Files installed on your machine

| File | Purpose |
|---|---|
| `~/.local/bin/omarchy-agent-usage-opencode` | Collector script (on PATH) |
| `~/.config/omarchy/plugins/wealth.agents/assets/opencode.svg` | Dark icon for light bar surfaces |
| `~/.config/omarchy/plugins/wealth.agents/assets/opencode-light.svg` | Light icon for dark bar surfaces |
| `~/.config/omarchy/hooks/post-boot.d/opencode-usage` | Runs collector on shell start |
| `~/.local/state/omarchy/agents/usage/opencode.json` | Generated usage record |

### How it works

1. Collector reads `~/.local/share/opencode/opencode.db` (read-only) and scans `session_message` for token usage.
2. Aggregates by date and model, prints JSON record to stdout.
3. Results cached in `~/.cache/omarchy/agent-usage/` for 20 seconds.
4. State file written to `~/.local/state/omarchy/agents/usage/opencode.json`.
5. Agents panel auto-discovers the record — no config changes needed.
6. Post-boot hook refreshes stats when the shell starts.

### User experience

- Middle-click the AI agent icon to cycle between Claude, Codex, Fireworks, and **OpenCode**.
- Click to open the panel: OpenCode tab shows today's prompts/sessions, 7-day chart, per-model breakdown.
- No rate-limit or balance section (OpenCode has no billing API).
- Official OpenCode logo used instead of generic glyph.

### Refresh

The orchestrator (`omarchy-agent-usage-update`) only scans `/usr/share/omarchy/bin/`, so it won't auto-run this collector on its 15-min timer. Options:

```bash
# Manual refresh:
~/.local/bin/omarchy-agent-usage-opencode --force > ~/.local/state/omarchy/agents/usage/opencode.json

# System-wide install (for orchestrator discovery):
sudo cp ~/.local/bin/omarchy-agent-usage-opencode /usr/share/omarchy/bin/
sudo chmod +x /usr/share/omarchy/bin/omarchy-agent-usage-opencode
```

### Known limitation

SQLite database lock contention when opencode is actively writing. The collector uses `mode=ro` with a 2-second timeout and returns empty stats on lock. The 20-second cache means subsequent reads are fast. If you see empty stats, wait a moment and refresh.

