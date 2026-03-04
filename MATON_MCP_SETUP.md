# Maton AI MCP Configuration for ZeroClaw

## Overview

This document explains how to configure the Maton AI MCP server with ZeroClaw for accessing 100+ business integrations including HubSpot, Salesforce, Google Workspace, Slack, Stripe, Shopify, Jira, and more.

## Prerequisites

1. **Node.js/npm** installed (for running npx)
2. **Maton API Key** from https://maton.ai/api-keys

## Step 1: Get Your Maton API Key

1. Visit https://maton.ai/api-keys
2. Sign in or create a Maton account
3. Generate a new API key
4. Copy the key (you'll need it in the next step)

## Step 2: Set Environment Variable

Before starting ZeroClaw, set your Maton API key:

```bash
export MATON_API_KEY="your-maton-api-key-here"
```

Or add it to your shell profile (~/.bashrc, ~/.zshrc, etc.):

```bash
echo 'export MATON_API_KEY="your-maton-api-key-here"' >> ~/.bashrc
source ~/.bashrc
```

## Step 3: Verify Configuration

Check that the Maton MCP server is configured in config.toml:

```toml
[mcp_servers.maton]
command = "npx"
args = ["-y", "@maton/mcp", "hubspot", "--actions=all", "--api-key=${MATON_API_KEY}"]
```

This configuration:
- **command**: Uses `npx` to run the Maton MCP package
- **args**: Runs Maton MCP for HubSpot with all available actions
- **api-key**: Uses environment variable interpolation via `${MATON_API_KEY}`

## Step 4: Restart ZeroClaw Daemon

After setting the environment variable, restart the daemon:

```bash
tmux kill-session -t zeroclaw
tmux new-session -d -s zeroclaw "/home/isaaceliape/.cargo/bin/zeroclaw daemon"
```

## Available Actions

The Maton MCP server provides access to 100+ API actions across multiple platforms:

**CRM:**
- HubSpot (contacts, deals, companies, pipelines)
- Salesforce (contacts, opportunities, accounts)

**Communication:**
- Slack (messages, channels, workflows)
- Google Workspace (Gmail, Calendar, Docs)

**Payments & Commerce:**
- Stripe (charges, customers, subscriptions)
- Shopify (products, orders, customers)

**Project Management:**
- Jira (issues, projects, workflows)

And many more integrations...

## Customizing Actions

To use specific actions instead of all available ones, modify the config:

```toml
[mcp_servers.maton]
command = "npx"
args = ["-y", "@maton/mcp", "hubspot", "--actions=create-contact,list-contacts,update-contact", "--api-key=${MATON_API_KEY}"]
```

## Available Integrations

You can switch between integrations by changing the service name in the args:

```toml
# For Salesforce
args = ["-y", "@maton/mcp", "salesforce", "--actions=all", "--api-key=${MATON_API_KEY}"]

# For Stripe
args = ["-y", "@maton/mcp", "stripe", "--actions=all", "--api-key=${MATON_API_KEY}"]

# For Jira
args = ["-y", "@maton/mcp", "jira", "--actions=all", "--api-key=${MATON_API_KEY}"]
```

## Testing the Integration

Once configured, ZeroClaw will have access to Maton's tools. You can test by:

1. Sending a message to your Telegram bot or CLI
2. Ask it to perform a Maton action, e.g., "List my HubSpot contacts"
3. The agent will use the Maton MCP tools to fetch the data

## Troubleshooting

**Issue:** "Unknown config key" warnings

**Solution:** These are normal if you're using environment variable interpolation. The config will validate once the daemon starts.

**Issue:** MCP server fails to connect

**Solution:**
1. Verify MATON_API_KEY is set: `echo $MATON_API_KEY`
2. Check internet connectivity
3. Verify the API key is valid at https://maton.ai/api-keys
4. Check daemon logs: `tmux attach -t zeroclaw`

**Issue:** Actions not appearing in agent

**Solution:**
1. Verify `npx` can run: `npx -y @maton/mcp hubspot --help`
2. Check your API key has permission for those actions
3. Restart the daemon after configuration changes

## References

- [Maton AI Agent Toolkit](https://github.com/maton-ai/agent-toolkit)
- [Maton MCP npm Package](https://www.npmjs.com/package/@maton/mcp)
- [ZeroClaw MCP Integration](https://github.com/zeroclaw-labs/zeroclaw/blob/main/docs/config-reference.md)

## Environment Variables

ZeroClaw supports environment variable interpolation using `${VAR_NAME}` syntax:

```toml
# This will be replaced with the value of MATON_API_KEY
api-key = "${MATON_API_KEY}"
```

You can set variables in:
1. Shell profile (~/.bashrc, ~/.zshrc)
2. `.env` file in the workspace directory
3. Directly in the terminal before starting the daemon
