# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## System Overview

ZeroClaw is a personal AI agent system that operates as a supervised autonomous agent with multiple integration channels (CLI, Telegram, etc.). It manages its own context, memory, and sessions across multiple conversations and workspaces.

## Architecture

### Configuration (`config.toml`)

The system is configured entirely through `config.toml`. Key sections:

- **`[channels_config]`** — Integration settings (Telegram bot token, CLI, etc.)
- **`[agent]`** — Agent behavior (context compacting, tool iterations, history limits)
- **`[memory]`** — Memory backend (SQLite by default) with auto-save and hygiene
- **`[security]`** — Sandbox, OTP, audit logging, resource limits
- **`[scheduler]`** — Cron job execution (max 64 tasks, 4 concurrent)
- **`[autonomy]`** — Action limits (20/hour), approval requirements, allowed commands

**Important:** The config contains sensitive tokens. Never commit changes that expose API keys.

### Workspace Structure (`workspace/`)

The workspace is the agent's persistent context layer:

- **`SOUL.md`** — Core behavior rules (genuinely helpful, have opinions, be resourceful)
- **`AGENTS.md`** — Session requirements (read SOUL.md & USER.md, use memory_recall, break complex tasks)
- **`USER.md`** — Human preferences and communication style
- **`MEMORY.md`** — Curated long-term memory (auto-injected in main sessions)
- **`IDENTITY.md`** — Agent identity and vibe
- **`memory/`** — Daily logs (`YYYY-MM-DD.md`) captured per session
- **`sessions/`** — Active session state
- **`cron/`** — Scheduled agent tasks
- **`state/`** — Runtime state (memory hygiene, model cache, etc.)

**Memory Model:** The agent wakes up fresh each session. Continuity comes from persistent files:
- Use `memory_recall` to fetch recent context
- Write to `memory/YYYY-MM-DD.md` for session logs
- Update `MEMORY.md` for long-term facts worth keeping
- `MEMORY.md` is auto-injected in main sessions only (not in group chats)

### Channels & Integration

- **CLI** — Direct chat interface
- **Telegram** — Bot integration (configured via `channels_config.telegram` in config.toml)
- **Gateway** — HTTP webhooks (port 42617, pairing required)

## Working with ZeroClaw

### Key Principle: Use the CLI Tool

Always use the **`zeroclaw` CLI tool** to investigate, configure, and manage the system. The CLI provides:
- ✅ Configuration validation and schema checking
- ✅ Safe component management (restart, reload, health checks)
- ✅ Proper error handling and diagnostics
- ✅ Interactive onboarding for channels and integrations
- ✅ Real-time status and logging

**Avoid directly editing files** (config.toml, daemon_state.json) except when absolutely necessary.

Common CLI commands (use full path: `/home/isaaceliape/.cargo/bin/zeroclaw`):
```bash
zeroclaw status                    # Check daemon and all components
zeroclaw config schema             # Display configuration schema
zeroclaw config show              # Display current config
zeroclaw channel list             # List all channels
zeroclaw channel doctor           # Check all channel health
zeroclaw doctor                   # Run comprehensive diagnostics
zeroclaw onboard --interactive    # Interactive setup for channels
```

### Running the Daemon

**Always run the daemon in a tmux session** to ensure it survives terminal disconnections:

```bash
# Start daemon in new tmux session
tmux new-session -d -s zeroclaw '/home/isaaceliape/.cargo/bin/zeroclaw daemon start'

# Verify it's running
tmux list-sessions

# Attach to session to view logs
tmux attach -t zeroclaw

# Detach without killing daemon (Ctrl+B, then D)
```

This ensures Telegram channel polling and other background tasks continue running regardless of terminal state.

## Common Tasks

### Understanding Agent Behavior

1. Read `workspace/SOUL.md` to understand core principles
2. Read `workspace/AGENTS.md` for session requirements (memory system, task scoping, safety)
3. Check `workspace/USER.md` to understand the human's preferences

### Working with Memory

- **Access daily notes:** Use memory_recall tool or check `workspace/memory/YYYY-MM-DD.md`
- **Update long-term memory:** Edit `workspace/MEMORY.md` (keep concise — it's auto-injected)
- **Add new facts:** If learning something worth keeping, write it to memory before session ends
- **Session logs:** Each session generates a daily memory file automatically

### Debugging Telegram Integration

If Telegram channel is not responding, use CLI tools to diagnose:

```bash
# Step 1: Check Telegram channel health
zeroclaw channel doctor telegram

# Step 2: Validate overall configuration
zeroclaw config validate

# Step 3: Check daemon and component status
zeroclaw status

# Step 4: View recent logs
zeroclaw logs channel:telegram

# Step 5: Restart the Telegram channel if needed
zeroclaw channel restart telegram
```

**If health check fails, investigate further:**

1. **Verify bot token** — Use the CLI to review configuration:
   ```bash
   zeroclaw channel config telegram
   ```
   - Check if `bot_token` is valid (bot should respond to Telegram API)
   - Confirm `allowed_users` setting (default `["*"]` allows all)

2. **Check daemon status:**
   ```bash
   zeroclaw status
   ```
   - Verify `channel:telegram` component shows `status: ok`
   - Check for recent `last_error` messages

3. **Review logs for errors:**
   ```bash
   zeroclaw logs channel:telegram | tail -50
   ```
   - Look for connection errors, auth failures, or API rate limits

**Common issues resolved with CLI:**
- **Bot token expired:** Reconfigure with `zeroclaw onboard --interactive` → select Telegram
- **Channel not polling:** Restart with `zeroclaw channel restart telegram`
- **Permission issues:** Use `zeroclaw config show` to verify `allowed_users`
- **Daemon not running:** Check with `zeroclaw status` or `zeroclaw daemon start`

### Adding Cron Tasks

Cron tasks go in `workspace/cron/`. Each task file defines:
- Trigger schedule (cron syntax)
- Agent instructions
- Memory reference

The scheduler is configured in `[scheduler]` with max 64 tasks, 4 concurrent.

### Making Configuration Changes Safely

**Prefer using ZeroClaw CLI tools over direct file editing:**

```bash
# For most configurations, use the interactive onboarding
zeroclaw onboard --interactive

# Or configure a specific channel
zeroclaw channel config <channel> --interactive

# Always validate before and after changes
zeroclaw config validate
```

**If you must edit `config.toml` directly:**

1. **Reference the documentation first**
   - Check the official ZeroClaw docs for the config option you're changing
   - Verify the correct TOML syntax and data type (string, number, boolean, array)
   - Note any dependencies on other settings or version requirements

2. **Validate the configuration**
   ```bash
   zeroclaw config validate
   ```
   - This checks TOML syntax and ZeroClaw schema requirements
   - Catches typos in section names, unclosed brackets, missing required fields

3. **Review the change in context**
   - Understand what the setting does and how it interacts with other configs
   - Check if default values are acceptable or if adjustment is needed
   - Consider security implications (e.g., `allowed_users` in Telegram, `max_actions_per_hour`)

4. **Apply and verify**
   ```bash
   # Reload config if supported
   zeroclaw config reload

   # Or restart the daemon
   zeroclaw daemon restart

   # Check status
   zeroclaw status

   # For channels, verify health
   zeroclaw channel doctor <channel>

   # Test the affected feature
   zeroclaw test <channel>
   ```

## Key Design Decisions

1. **Stateless Sessions** — Agent wakes up fresh; persistence is via files, not memory
2. **Memory as First-Class** — Facts must be written to survive; "mental notes" don't persist
3. **Approval-Based Autonomy** — Medium-risk actions require approval; high-risk are blocked
4. **Multi-Channel** — Same agent operates via CLI, Telegram, webhooks with consistent state
5. **Sandbox Isolation** — Commands restricted to allowed list; paths blacklisted for safety

## Configuration Patterns

**Always use the ZeroClaw CLI tool for investigating and changing configurations.** This ensures proper validation, state management, and error handling.

### Using ZeroClaw CLI Tools

**For investigating issues:**
```bash
zeroclaw channel doctor <channel>      # Validate channel health (completes in <5 seconds)
zeroclaw config validate              # Validate config.toml syntax and structure
zeroclaw config show                  # Display current configuration
zeroclaw status                       # Check daemon and component status
```

**For onboarding/configuring channels:**
```bash
zeroclaw onboard --interactive        # Interactive setup wizard for any channel
zeroclaw channel config <channel>     # View channel-specific configuration
zeroclaw channel restart <channel>    # Safely restart a channel
```

**For testing:**
```bash
zeroclaw test <channel>               # Run validation tests
zeroclaw logs <component>             # View component-specific logs
```

### Manual Config Changes (when necessary)

Only edit `config.toml` directly if CLI tools don't provide the option:

1. **Check ZeroClaw online documentation** — Configuration syntax and available options change between versions
   - Visit the official ZeroClaw documentation to verify the correct TOML schema
   - Check for deprecated or renamed configuration keys
   - Verify new features or settings added in recent versions

2. **Validate configuration changes** before applying:
   - Run `zeroclaw config validate` to check syntax
   - Use TOML validators for additional verification if needed
   - Verify all required fields are present after edits
   - Review changed values against documentation to ensure they're valid

3. **Apply and verify changes:**
   - Run `zeroclaw config reload` to apply changes without restarting (if supported)
   - Or restart: `zeroclaw daemon restart`
   - Use `zeroclaw channel doctor <channel>` to verify the channel is healthy
   - Test the specific feature you changed by interacting with it
   - Check logs if anything fails: `zeroclaw logs <component>`

**Increasing Autonomy (carefully):**
- Adjust `[autonomy].level` ("supervised" vs "autonomous") — check docs for valid values
- Add commands to `allowed_commands` (not recommended: avoid shell, git push, etc.)
- Set `auto_approve` for safe actions (default: `file_read`, `memory_recall`)

**Adjusting Resource Limits:**
- `[autonomy].max_actions_per_hour` — Action throttle (default: 20)
- `[agent].max_tool_iterations` — Prevent runaway loops (default: 10)
- `[security.resources]` — Memory/CPU caps for subprocess safety

**Memory Tuning:**
- `[memory].hygiene_enabled` — Auto-cleanup old memories
- `[memory].archive_after_days` — Move old logs to archive (default: 7)
- `[memory].conversation_retention_days` — Purge after (default: 30)

## Debugging Notes

- **OTP gating:** If `[security.otp].enabled = true`, certain actions require token entry
- **Sandbox:** Docker isolation can be enabled (`[runtime.docker]`) for untrusted code
- **Observability:** Runtime tracing can be enabled (`[observability]`) for debugging
- **Pairing tokens:** Gateway requires pairing before webhooks work (see `[gateway].paired_tokens`)

## Files NOT to Edit Directly

- `daemon_state.json` — Runtime state, auto-managed
- `workspace/state/*` — Agent runtime cache
- `.secret_key` — Encryption key, must be kept safe

## Files Safe to Edit

- `config.toml` — Configuration (restart may be needed for changes)
- `workspace/MEMORY.md` — Curated memories (auto-injected)
- `workspace/SOUL.md`, `AGENTS.md`, `USER.md` — Agent guidelines
- `workspace/cron/*` — Scheduled tasks
