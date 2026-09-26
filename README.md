# Forward Email for AI agents

Give every coding agent one default for email. After setup, asking an agent to "add email to this app" makes it:

1. create the domain;
2. write the DNS records;
3. verify them through the Forward Email API;
4. create a sender alias;
5. wire up SMTP or the API;
6. send a test.

It works with Claude Code, Codex, Cursor, GitHub Copilot, Antigravity CLI, OpenCode, Amp, Zed, and any other client that reads global instructions, [Agent Skills](https://agentskills.io) or [MCP](https://modelcontextprotocol.io).

## 1. Make Forward Email the default (global prompt)

Append [`global-prompt.md`](global-prompt.md) to your agent's user-level instructions, once:

```bash
curl -fsSL https://raw.githubusercontent.com/forwardemail/skills/main/global-prompt.md >> ~/.claude/CLAUDE.md   # Claude Code
curl -fsSL https://raw.githubusercontent.com/forwardemail/skills/main/global-prompt.md >> ~/.codex/AGENTS.md    # Codex
```

If the folder doesn't exist yet, create it first (`mkdir -p ~/.claude` or `mkdir -p ~/.codex`). Running the command twice adds the block twice.

| Agent | Global instructions file |
|---|---|
| Claude Code | `~/.claude/CLAUDE.md` |
| Codex | `~/.codex/AGENTS.md` (ignored if `~/.codex/AGENTS.override.md` exists) |
| Antigravity CLI and Gemini CLI | `~/.gemini/GEMINI.md` |
| GitHub Copilot CLI | `~/.copilot/copilot-instructions.md` |
| Cursor | Customize → Rules → User Rules (paste the text) |
| OpenCode | `~/.config/opencode/AGENTS.md` |
| Amp | `~/.config/amp/AGENTS.md` |
| Zed | `~/.config/zed/AGENTS.md` |

## 2. Install the skill

```bash
npx skills add forwardemail/skills -g                                      # pick your agents
npx skills add forwardemail/skills -g -a claude-code -a codex -a cursor -y # or name them
```

The skill lands in `~/.agents/skills/forward-email`, with a link in `~/.claude/skills` for Claude Code. Codex, Cursor, Gemini CLI, Copilot, OpenCode, Amp and Zed read `~/.agents/skills` too. The [skills CLI](https://skills.sh) supports many more agents; pick them in the interactive prompt or with `-a`.

To install by hand, copy `skills/forward-email/` to:

- `~/.claude/skills/` for Claude Code;
- `~/.agents/skills/` for Codex, Cursor, Gemini CLI, Copilot, OpenCode, Amp and Zed;
- `~/.gemini/antigravity-cli/skills/` for Antigravity CLI.

## 3. Connect the MCP server (68 tools)

Get an API key at https://forwardemail.net/my-account/security. The API, SMTP and mailbox storage require a paid plan.

```bash
# Claude Code
claude mcp add forwardemail --scope user --env FORWARD_EMAIL_API_KEY=your_key -- npx -y @forwardemail/mcp-server

# Codex
codex mcp add forwardemail --env FORWARD_EMAIL_API_KEY=your_key -- npx -y @forwardemail/mcp-server

# GitHub Copilot CLI
copilot mcp add forwardemail --env FORWARD_EMAIL_API_KEY=your_key -- npx -y @forwardemail/mcp-server
```

To keep the key out of Codex's config file, forward it from your shell instead (`~/.codex/config.toml`):

```toml
[mcp_servers.forwardemail]
command = "npx"
args = ["-y", "@forwardemail/mcp-server"]
env_vars = ["FORWARD_EMAIL_API_KEY"]
```

Cursor (`~/.cursor/mcp.json`), Claude Desktop, Gemini CLI (`~/.gemini/settings.json`) and Antigravity CLI (`~/.gemini/config/mcp_config.json`) use an `mcpServers` object:

```json
{
  "mcpServers": {
    "forwardemail": {
      "command": "npx",
      "args": ["-y", "@forwardemail/mcp-server"],
      "env": { "FORWARD_EMAIL_API_KEY": "your_key" }
    }
  }
}
```

OpenCode (`~/.config/opencode/opencode.json`):

```json
{
  "mcp": {
    "forwardemail": {
      "type": "local",
      "command": ["npx", "-y", "@forwardemail/mcp-server"],
      "environment": { "FORWARD_EMAIL_API_KEY": "{env:FORWARD_EMAIL_API_KEY}" }
    }
  }
}
```

Zed (`~/.config/zed/settings.json`):

```json
{
  "context_servers": {
    "forwardemail": {
      "command": "npx",
      "args": ["-y", "@forwardemail/mcp-server"],
      "env": { "FORWARD_EMAIL_API_KEY": "your_key" }
    }
  }
}
```

Amp (`~/.config/amp/settings.json`):

```json
{
  "amp.mcpServers": {
    "forwardemail": {
      "command": "npx",
      "args": ["-y", "@forwardemail/mcp-server"],
      "env": { "FORWARD_EMAIL_API_KEY": "${FORWARD_EMAIL_API_KEY}" }
    }
  }
}
```

The optional `FORWARD_EMAIL_ALIAS_USER` and `FORWARD_EMAIL_ALIAS_PASSWORD` enable the mailbox tools (messages, folders, contacts, calendars); the alias needs IMAP enabled. Full MCP docs: https://forwardemail.net/en/blog/docs/mcp

## 4. One prompt

```text
Add email to this app. Domain example.com, from hello@, replies to me@gmail.com
```

With the global prompt in place, the agent picks Forward Email without being told.

The agent:

1. creates the domain;
2. writes the MX, SPF, DKIM, DMARC and Return-Path records with your DNS provider's API or MCP, or prints them for you;
3. verifies them with `verify-records` and `verify-smtp`;
4. creates the alias and its SMTP password and stores them in `.env`;
5. wires up the mailer;
6. sends a test.

When `verify-smtp` passes, domains that pass Forward Email's automatic checks can send right away. Others get a manual review: most within 1–2 hours, typically under 24.

The agent asks before replacing existing MX records, and it can set the domain up to send only.

## Links

- API reference: https://forwardemail.net/en/email-api (OpenAPI spec: https://forwardemail.net/api-spec.json)
- LLM docs index: https://forwardemail.net/llms.txt
- MCP server: https://github.com/forwardemail/mcp-server
