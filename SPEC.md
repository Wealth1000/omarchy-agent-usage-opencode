# Spec: omarchy-agent-usage-opencode

## Problem Statement

Omarchy's taskbar AI agent widget shows usage statistics for Claude Code, Codex, and Fireworks, but the user primarily uses OpenCode as their daily AI coding agent. OpenCode sessions are partially scanned by the Claude collector (for Anthropic-provider messages only) and those stats get attributed to Claude rather than OpenCode. There is no dedicated OpenCode agent record, so the user cannot see their actual OpenCode usage — tokens, model breakdown, daily activity — as a first-class citizen in the agents panel.

## Solution

Create a standalone `omarchy-agent-usage-opencode` collector script that scans the OpenCode SQLite database (`~/.local/share/opencode/opencode.db`) for assistant messages with token usage data, and writes a display-ready JSON record into the agents usage directory. The agents panel auto-discovers new agent records, so no panel or config changes are required. A branded SVG icon (both dark and light variants) is added so the panel shows the OpenCode logo instead of the generic `󱚣` glyph.

## User Stories

1. As an OpenCode user, I want to see my token usage stats in the Omarchy taskbar agents panel, so that I can monitor my daily and weekly usage without opening a terminal.
2. As an OpenCode user, I want to see a per-model token breakdown (input, output, reasoning, cache), so that I can understand which models consume the most tokens.
3. As an OpenCode user, I want to see a 7-day token activity chart, so that I can spot usage patterns over time.
4. As an OpenCode user, I want today's prompt count and session count displayed, so that I can gauge how active I've been.
5. As an OpenCode user, I want the OpenCode agent to appear as a separate tab in the agents panel alongside Claude and Codex, so that I can cycle between agents with middle-click.
6. As an OpenCode user, I want a branded OpenCode icon in the panel hero (instead of the generic AI glyph), so that the panel visually distinguishes OpenCode from other agents.
7. As an OpenCode user on a light theme, I want the icon to adapt to the surface color, so that it remains readable regardless of theme.
8. As an OpenCode user, I want the collector to cache scan results for 20 seconds, so that frequent panel opens don't repeatedly hit the database.
9. As an OpenCode user, I want `--force` to invalidate the cache and do a fresh scan, so that I can refresh after a long idle period.
10. As an OpenCode user, I want `--limits-only` to reuse a recent scan rather than re-scanning the full database, so that opening the panel is fast.
11. As an OpenCode user, I want the agent to appear in the bar only after I've actually used OpenCode (i.e., when there's data), so that empty agents don't clutter the bar.
12. As an OpenCode user, I want the total prompt/session counts to be visible, so that I can see all-time usage at a glance.
13. As an OpenCode user, I want the agent to not show rate limits or balance (since OpenCode has no per-account billing API), so that the limits/balance section is hidden cleanly.
14. As an OpenCode user, I want the collector to be read-only against the database, so that it never interferes with OpenCode's own writes.
15. As an OpenCode user, I want the script to handle missing or malformed database rows gracefully, so that one corrupt entry doesn't abort the entire scan.
16. As an OpenCode user, I want the agent record to use the same JSON schema as the other collectors, so that the panel renders it without any UI changes.
17. As an Omarchy user, I want to be able to disable the OpenCode agent via settings, so that I can hide it if I stop using it.
18. As an OpenCode user who switches between multiple models, I want the model breakdown to show each model ID I actually use (e.g., `agnes-2.5-flash`, `big-pickle`), so that I can see per-model costs.
19. As an OpenCode user on a machine without the database, I want the agent to simply not appear, so that the panel doesn't show an empty stub.
20. As an OpenCode user, I want the active dates to be tracked, so that the "N days active" summary is accurate.

## Implementation Decisions

- **Collector script**: A standalone Python 3 script at `omarchy-agent-usage-opencode` (to be installed to `/usr/bin/` on the target system). It follows the same CLI interface as the existing collectors: `--force`, `--limits-only`, `--cache-seconds`.
- **Data source**: Reads from `~/.local/share/opencode/opencode.db` (configurable via `XDG_DATA_HOME/opencode/opencode.db`). Queries the `session_message` table for assistant messages with non-zero token usage.
- **Schema compatibility**: The JSON output uses the exact same schema as the Claude/Codex/Fireworks collectors (`schemaVersion: 1`, with fields: `id`, `name`, `updatedAt`, `ready`, `hasLocalStats`, `tierLabel`, `usageStatusText`, `authHelpText`, `limits`, `todayPrompts`, `todaySessions`, `todayTotalTokens`, `todayTokensByModel`, `recentDays`, `totalPrompts`, `totalSessions`, `activeDays`, `activeDates`, `modelUsage`).
- **No rate limits**: Since OpenCode has no billing API endpoint, `limits` is always an empty array and `tierLabel`/`usageStatusText`/`authHelpText` are empty strings. This causes the limits and balance sections to be hidden automatically by the panel.
- **Token extraction**: Parses `$.tokens.{input, output, reasoning, cache.{read, write}}` from each assistant message's JSON data. Reasoning tokens are counted as output (matching the Claude collector's convention).
- **Model identification**: Uses `$.model.id` as the model key. Falls back to `"opencode"` if absent.
- **Date handling**: Uses `$.time.created` (Unix ms) to determine the date of each message. Falls back to today's date on parse failure.
- **Caching**: Scans are cached in `~/.cache/omarchy/agent-usage/opencode-<sha1-of-db-path>.json` with a 20-second freshness window (same as other collectors).
- **File lock**: Uses `fcntl.flock` to prevent concurrent collector runs from corrupting the cache.
- **Atomic writes**: Cache and state files are written via temp-file-then-rename, matching the pattern used by existing collectors.
- **Icon assets**: Two SVG files — `opencode.svg` (dark-themed, for dark surfaces) and `opencode-light.svg` (light-themed, for light surfaces) — placed in the plugin's `assets/` directory. Both use transparent backgrounds and the OpenCode brand color palette.
- **Plugin location**: The collector and icons are delivered as part of a cloned `omarchy.agents` plugin under `~/.config/omarchy/plugins/omarchy.agents/`, which is the standard Omarchy customization pattern. This means updates to the packaged plugin won't overwrite the custom collector or icon.
- **No schema changes**: The panel's auto-discovery already handles new agents; no QML or manifest changes are needed.

## Testing Decisions

- **Integration test**: Run the collector script directly and verify it outputs valid JSON with the expected top-level fields.
- **Data accuracy test**: Cross-check token totals from the collector against a direct SQLite query on the user's database.
- **Cache test**: Run the collector twice with default cache seconds; verify the second run reads from cache (no disk I/O on the DB).
- **Force test**: Run with `--force`; verify the cache is invalidated and the DB is re-scanned.
- **Limits-only test**: Run with `--limits-only`; verify a recent cache hit is reused instead of re-scanning.
- **Missing DB test**: Run the collector with `XDG_DATA_HOME` pointing to a directory without an opencode DB; verify it outputs a valid (but empty) JSON record with `ready: false`.
- **Malformed row test**: Verify the collector skips rows with invalid JSON or missing token fields without crashing.
- **Icon render test**: Load both SVG files and verify they render without errors (check viewBox, no missing attributes).
- **Panel integration test**: After installation, open the agents panel and verify the OpenCode tab appears with correct data.
- **Backwards compatibility test**: Verify that adding the new collector does not affect the Claude or Codex collectors' output.

## Out of Scope

- Rate limit reporting (OpenCode has no public rate limit API).
- Balance/credit reporting (OpenCode has no prepaid billing for the CLI).
- Sync/aggregation across devices (out of scope for v1; the record schema supports it if added later).
- Modifying the existing Claude collector's opencode scan (those stats will continue to feed Claude; the new collector reads the same DB independently).
- Adding a keyboard shortcut or bar button customization (the generic `󱚣` fallback is replaced by the branded icon automatically).
- Supporting OpenCode configurations beyond the default `~/.local/share/opencode/opencode.db` path.

## Further Notes

- The OpenCode DB schema uses `session_message` table with columns `id`, `session_id`, `type`, `seq`, `time_created`, `time_updated`, `data` (JSON). The `data` column contains the full message object including `providerID`, `model.id`, `tokens`, and `time.created`.
- The existing Claude collector already scans this same DB for `providerID == "anthropic"` messages. The new collector will read the same table but filter for all provider types (since OpenCode is itself the provider framework).
- The opencode brand favicon (`favicon-v3.svg` from the upstream repo) uses a dark `#131010` background which is unsuitable for a panel icon on light themes. We will extract the logo mark (the white geometric "OC" spiral) and render it on transparent background with appropriate foreground colors for both variants.
- The panel's `iconCandidatesForProvider` function checks luminance of the surface color and tries the `-light.svg` variant first on light surfaces, then falls back to the default `.svg`. Both variants must be provided for optimal appearance.
