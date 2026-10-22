# HANDOFF — Session Summary

## What was done

Created `omarchy-agent-usage-opencode`: a collector script that adds **OpenCode** as a first-class agent in the Omarchy taskbar AI agents panel.

### Files created

| File | Purpose |
|---|---|
| `omarchy-agent-usage-opencode` | Python 3 collector script (install to `~/.local/bin/`) |
| `assets/opencode.svg` | Dark-themed icon for the agents panel hero |
| `assets/opencode-light.svg` | Light-themed icon variant for light bar surfaces |
| `README.md` | Installation & usage docs |
| `LICENSE` | MIT license |
| `SPEC.md` | Feature specification |
| `HANDOFF.md` | This file |

### Files modified on the system

| File | Change |
|---|---|
| `~/.local/bin/omarchy-agent-usage-opencode` | Installed collector (symlink from project) |
| `~/.config/omarchy/plugins/wealth.agents/assets/opencode.svg` | Branded icon |
| `~/.config/omarchy/plugins/wealth.agents/assets/opencode-light.svg` | Light theme icon |
| `~/.config/omarchy/hooks/post-boot.d/opencode-usage` | Runs collector on shell start to keep stats fresh |
| `~/.local/state/omarchy/agents/usage/opencode.json` | Generated usage record (auto-refreshed by hook + panel timer) |

### How it works

1. The collector reads OpenCode's SQLite database (`~/.local/share/opencode/opencode.db`) in read-only mode.
2. It scans `session_message` for rows with token usage data, aggregates by date/model, and prints a JSON record.
3. The record is cached in `~/.cache/omarchy/agent-usage/` for 20 seconds.
4. The agents panel (via `Main.qml`) auto-discovers `opencode.json` in the usage directory — no config changes needed.
5. A post-boot hook refreshes the stats when the shell starts.
6. The panel timer (every 900s by default) re-runs the update orchestrator; since our collector lives in `~/.local/bin/` rather than `/usr/share/omarchy/bin/`, the orchestrator won't auto-discover it — the hook ensures fresh data on login.

### User experience

- Middle-click the AI agent icon in the taskbar to cycle between Claude, Codex, Fireworks, and **OpenCode**.
- Click to open the panel: the OpenCode tab shows today's prompts/sessions, a 7-day token chart, and an all-time per-model breakdown.
- No rate-limit or balance section appears (OpenCode has no billing API).
- The branded OpenCode icon replaces the generic `󱚣` glyph in the panel hero.

### Known limitation

The `omarchy-agent-usage-update` orchestrator only scans `/usr/share/omarchy/bin/` for collectors, so it won't auto-run the OpenCode collector on its 15-minute timer. The post-boot hook covers login-time refresh. For on-demand refresh, run:

```bash
~/.local/bin/omarchy-agent-usage-opencode --force
```

Or open the agents panel and press `r` (the panel triggers a refresh which calls the orchestrator — but the orchestrator won't find our collector, so this only refreshes the other agents. Use the command above for OpenCode-specific refresh).

## Next steps / what to do next

1. **Restart the shell** (already done via `omarchy restart shell`).
2. **Verify** the OpenCode tab appears in the agents panel.
3. **Push to GitHub** — initialize the repo, add a `.gitignore`, commit, and push public.
4. **Optional**: Copy the collector to `/usr/bin/` with `sudo cp` for system-wide availability (so the orchestrator picks it up on its timer).

## Notes

- The script handles SQLite locks gracefully: if opencode is writing when the collector runs, it returns empty stats and relies on the cache. Subsequent runs (after the lock releases) will populate correctly.
- Token counts include reasoning tokens folded into output (matching Claude collector convention).
- Model IDs come from `$.model.id` in the DB (e.g., `agnes-2.5-flash`, `big-pickle`).
