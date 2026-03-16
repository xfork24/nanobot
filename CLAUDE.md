# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

nanobot is an ultra-lightweight (~4,000 lines) personal AI assistant framework. It connects to multiple chat platforms (Telegram, Discord, Feishu, Slack, WhatsApp, QQ, DingTalk, Email, Matrix, Mochat) and provides an agent that can execute tools, manage memory, and handle scheduled tasks.

## Development Commands

```bash
# Install in development mode
uv pip install -e .

# Run tests
uv run pytest

# Run a single test
uv run pytest tests/test_filename.py::test_function_name

# Lint code
uv run ruff check nanobot/
```

## Architecture

The codebase follows a layered architecture:

```
Channels → Message Bus → Agent Loop → Tools + Providers + Memory
                                    ↓
                              Skills (loaded dynamically)
```

### Core Components

- **agent/loop.py**: The `AgentLoop` class is the heart of the system — it receives messages, builds context, calls the LLM, executes tool calls, and sends responses.
- **bus/**: Message routing. `InboundMessage` and `OutboundMessage` events flow through a `MessageBus`.
- **channels/**: Chat platform integrations. Each channel (telegram.py, discord.py, etc.) implements `BaseChannel` and forwards messages to the bus.
- **providers/**: LLM integrations. The registry (`providers/registry.py`) defines all supported providers. Adding a new provider requires adding a `ProviderSpec` entry and a config field — no if-elif chains.
- **agent/tools/**: Built-in tools (shell, file read/write/edit, web search/fetch, MCP, cron, message, spawn). Tools are registered in `ToolRegistry`.
- **agent/memory.py**: Two-layer memory — recent history (Session) and long-term memory (MemoryStore with consolidation).
- **session/**: Per-user conversation sessions with history management.
- **skills/**: Bundled skills (GitHub, weather, tmux, etc.) that extend agent capabilities.
- **cron/**: Scheduled task execution.
- **heartbeat/**: Periodic wake-up (every 30 min) to check for proactive tasks.
- **config/**: Pydantic-based configuration schema and loading.

### Data Flow

1. User sends message to any channel (Telegram, Discord, etc.)
2. Channel converts to `InboundMessage` and publishes to `MessageBus`
3. `AgentLoop` subscribes to inbound messages
4. For each message: build context → call LLM → execute tool calls → send response
5. Response published as `OutboundMessage` → channel sends to user

### Key Patterns

- **Async throughout**: All I/O is async (`async def`, `await`, `asyncio`)
- **Dependency injection**: Components receive their dependencies (bus, provider, config) in constructors
- **Provider auto-detection**: The system auto-detects which provider to use based on API key prefix or model name keywords
- **Tool validation**: Tools validate their inputs via Pydantic models before execution

## Configuration

Config file: `~/.nanobot/config.json`

- `providers`: LLM provider settings (API keys, bases)
- `channels`: Chat platform credentials
- `agents.defaults`: Default model, provider, temperature
- `tools`: Security settings (restrictToWorkspace, exec path)
- `mcpServers`: MCP tool servers

## Adding New Providers

Edit `nanobot/providers/registry.py` — add a `ProviderSpec` entry. Then add a config field in `nanobot/config/schema.py`. That's it.

## Adding New Channels

1. Create `nanobot/channels/<name>.py` implementing `BaseChannel`
2. Add config validation in `nanobot/config/schema.py`
3. Add channel setup in `nanobot/channels/manager.py`
