# OpenCode Docker Setup

OpenCode is a terminal AI coding assistant. This repo runs the MCP servers in Docker and OpenCode natively via npm.

---

## Upgrading OpenCode

OpenCode is installed via npm (`npm install -g opencode-ai`). To upgrade:

```bash
npm install -g opencode-ai@latest
```

`autoupdate: false` is set in `~/.opencode/opencode.json` to suppress the in-app upgrade prompt.

---

## Important: noexec on /tmp

The host `/tmp` directory has `noexec` set, which causes OpenCode to hang silently at startup (it extracts a render library to `/tmp`). Fix this by pointing `TMPDIR` at a directory without `noexec`. Add to `~/.bashrc`:

```bash
export TMPDIR="/workspace/${USER}/tmp"
mkdir -p "$TMPDIR"
```

---

## First-time setup

### 1. Install OpenCode

```bash
npm install -g opencode-ai
```

Verify:

```bash
opencode --version
```

### 2. Add your keys to `~/.bashrc`

```bash
# LiteLLM proxy token
export ANTHROPIC_AUTH_TOKEN="your-token-here"

# OpenCode Zen paid tier (optional — free models work without this)
export OPENCODE_API_KEY="your-zen-key-here"

# Sentry MCP (optional)
export SENTRY_ACCESS_TOKEN="your-sentry-token-here"
```

Reload: `source ~/.bashrc`

### 3. Create Docker network (once only)

```bash
docker network create opencode-network
```

### 4. Create your OpenCode config at `~/.opencode/opencode.json`

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "litellm": {
      "api": "openai",
      "name": "LiteLLM",
      "options": {
        "apiKey": "your-token-here",
        "baseURL": "https://your-litellm-proxy.example.com"
      },
      "models": {
        "claude-sonnet": {
          "id": "au.anthropic.claude-sonnet-4-6",
          "limit": { "context": 200000, "output": 8096 }
        },
        "claude-haiku": {
          "id": "au.anthropic.claude-haiku-4-5-20251001-v1:0",
          "limit": { "context": 200000, "output": 8096 }
        }
      }
    },
    "opencode": {
      "options": {
        "apiKey": "${OPENCODE_API_KEY}"
      }
    }
  },
  "autoupdate": false,
  "mcp": {
    "playwright": {
      "type": "remote",
      "url": "http://localhost:3002/mcp"
    },
    "semble": {
      "type": "remote",
      "url": "http://localhost:3003/mcp"
    },
    "ansible": {
      "type": "remote",
      "url": "http://localhost:3004/mcp"
    },
    "sentry": {
      "type": "remote",
      "url": "http://localhost:3005/mcp"
    }
  }
}
```

`autoupdate: false` suppresses the upgrade prompt. Upgrade manually with `npm install -g opencode-ai@latest`.

### 5. Start the MCP stack

```bash
docker compose up -d
```

### 6. Open the TUI

```bash
opencode /workspace/<user.name>/<project_dir>/
```

For example:

```bash
opencode /workspace/duncan.lester/ddf-dev/
```

---

## Using OpenCode

The OpenCode terminal interface opens. Type your request and press Enter:

```
> fix all TypeScript errors in src/
> add unit tests for api/routes.ts
> explain how the auth flow works
```

To switch models, use the model selector in the TUI. Available models include:
- `litellm/claude-sonnet` — Claude Sonnet via LiteLLM proxy
- `litellm/claude-haiku` — Claude Haiku via LiteLLM proxy
- `opencode/big-pickle` — OpenCode Zen (free)
- `opencode/*-free` — additional free Zen models

Press `:exit` or `Ctrl+C` to quit.

### Check MCP services are running

```bash
docker compose ps
```

You should see `playwright-mcp`, `semble-mcp`, `ansible-mcp`, and `sentry-mcp` all with status `Up`.

### Stop the MCP stack

```bash
docker compose down
```

### View logs

```bash
docker compose logs -f
```

---

## Non-interactive / scripted use

```bash
opencode run "review this file for bugs" /workspace/duncan.lester/myproject/
```

Example GitHub Actions step:

```yaml
- name: OpenCode review
  run: opencode run "summarise changes and flag any bugs"
  env:
    ANTHROPIC_AUTH_TOKEN: ${{ secrets.ANTHROPIC_AUTH_TOKEN }}
```

---

## MCP Services

MCP servers run in Docker and are exposed on localhost ports. OpenCode connects to them from the host.

| Service | Port | Purpose |
|---|---|---|
| playwright-mcp | 3002 | Headless browser automation |
| semble-mcp | 3003 | Semble healthcare platform API |
| ansible-mcp | 3004 | Ansible playbook execution |
| sentry-mcp | 3005 | Sentry error tracking |

### Playwright MCP

Browser automation — screenshots, form filling, E2E testing.

```
> take a screenshot of https://example.com
> fill the login form at https://staging.myapp.com
> run the checkout flow and report any errors
```

### Semble MCP

Semble healthcare platform API integration.

### Ansible MCP

Ansible playbook execution, inventory management, and ad-hoc commands. Built from `Dockerfile.ansible-mcp` using `bsahane/mcp-ansible`.

```
> run the deploy playbook against the staging inventory
> check which hosts are in the webservers group
> run an ad-hoc command to check disk space on all nodes
```

### Sentry MCP

Error tracking. Requires `SENTRY_ACCESS_TOKEN` in `~/.bashrc`. The container starts without it but will not connect to Sentry.

```
> show recent errors from my Sentry project
> get the stack trace for the latest JavaScript crash
```

---

## Offline mode (no internet, no API key)

Uses a local Ollama model instead of a cloud provider.

```bash
docker volume create ollama_data
docker compose -f docker-compose-offline.yml up -d ollama
docker exec ollama ollama pull mistral
docker compose -f docker-compose-offline.yml up -d
```

Then run:

```bash
opencode
```

---

## Using both OpenCode and Claude Code

Both tools share the same LiteLLM proxy and MCP containers.

**Claude Code** (`~/.claude/settings.json`):
```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://your-litellm-proxy.example.com",
    "ANTHROPIC_AUTH_TOKEN": "your-token-here",
    "ANTHROPIC_MODEL": "au.anthropic.claude-sonnet-4-6"
  }
}
```

**OpenCode** (`~/.opencode/opencode.json`): configured above.

Same proxy, same token — each developer sets their own in `~/.bashrc`, nothing goes in the repo.

---

## Resources

- [OpenCode docs](https://opencode.ai/docs)
- [OpenCode config schema](https://opencode.ai/config.json)
- [Getting started video](https://www.youtube.com/watch?v=e9j2iEwJru0)
- [MCP spec](https://modelcontextprotocol.io)
- [Discord](https://opencode.ai/discord)
