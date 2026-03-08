# Agent Auth by Vigil — Claude Code Plugin

Claude Code plugin for integrating [Vigil (Agent Auth)](https://usevigil.dev) into your website — SDK setup, database schema, callback handling, sign-in flow, and credential verification.

## Install via Plugin Marketplace (Recommended)

Add the marketplace and install the plugin:

```
/plugin marketplace add Agent-Auth-by-Vigil/agent-auth-byvigil-skills
/plugin install vigil-integration@agent-auth-byvigil
```

## Install via Skills CLI

```bash
npx skills add Agent-Auth-by-Vigil/agent-auth-byvigil-skills -a claude-code
```

## What's Included

This plugin teaches Claude Code how to:

- Set up the `auth-agents` SDK (Node.js or Python)
- Build a browser redirect sign-in flow (like "Sign in with Google" but for AI agents)
- Create callback pages that receive and verify agent credentials
- Write backend verification routes (Next.js, Express, FastAPI)
- Design database schemas for storing verified agent identities (SQL, Prisma, Drizzle)
- Handle credential expiration and re-authentication
- Follow security best practices

## Usage

Once installed, Claude Code will automatically use this skill when you ask things like:

- "Add Vigil agent authentication to my app"
- "Set up agent sign-in for my API"
- "Create a callback page for Agent Auth"
- "How do I verify agent credentials?"

## Links

- [Vigil Docs](https://usevigil.dev/docs)
- [Node.js SDK](https://www.npmjs.com/package/auth-agents)
- [Python SDK](https://pypi.org/project/auth-agents/)
- [Demo Site](https://demo.usevigil.dev)
