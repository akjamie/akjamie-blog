---
layout: post
title: "OpenClaw: A Self-Hosted AI Agent That Runs on Your Own Hardware"
date: "2026-04-24"
updated: "2026-04-26"
author: "Jamie Zhang"
tags: ["Agentic AI", "DevOps", "Open Source", "OpenClaw"]
categories: ["AI"]
description: "A deep dive into OpenClaw: its hub-and-spoke architecture, setup experience with MiniMax models, Feishu integration, and how to handle network restrictions when corporate firewalls block external AI services."
image: "/img/ai-01.jpg"
keywords: ["openclaw", "ai agent", "self-hosted", "local llm", "devops", "agentic ai", "feishu", "minimax"]
---

# OpenClaw: A Self-Hosted AI Agent That Runs on Your Own Hardware

## The Problem with Cloud AI in Corporate Environments

If you've tried to use ChatGPT, Claude, or any cloud-based AI assistant at work, you know the friction: your company blocks external AI services, the legal team flags data privacy concerns, IT won't approve another SaaS tool, and even if they did, sending code and documents to a third-party API feels like handing over the keys to your intellectual property.

This isn't unique to AI. Every time a new category of developer tool emerges, enterprise IT moves slowly, and the people who could benefit most — the engineers on the ground — are left waiting.

**OpenClaw** is the answer to that gap. It lets you run a capable AI agent entirely on your own hardware, connecting to whatever LLM you choose (cloud or local), and it works with messaging platforms you already use. Nothing leaves your machine unless you explicitly configure it to.

## What OpenClaw Actually Is

OpenClaw is not a chatbot wrapper around an API. It's an **operating system for AI agents** — a persistent, long-running service that treats AI as infrastructure: sessions, memory, tool sandboxing, access control, and orchestration are all first-class concerns.

The project was started by Peter Steinberger (founder of PSPDFKit) as a weekend project in late 2025 called "Warelay" — a WhatsApp relay that passes your message to an AI and sends the reply back. It went viral in January 2026 after trademark issues forced two rebrands in four days, generating fresh waves of tech press coverage each time. The project is MIT-licensed and completely open source.

At its core, OpenClaw connects LLMs (Anthropic, OpenAI, local models via Ollama, MiniMax, Google Gemini, and others) to your local machine and messaging platforms. The key architectural pieces are the **Gateway** and the **Agent Runtime**.

## Architecture: Hub-and-Spoke Around a Gateway

OpenClaw follows a hub-and-spoke architecture centered on a single long-running Node.js process called the **Gateway**:

```
[WhatsApp] [Telegram] [Slack] [Discord] [Feishu] [CLI] [Web UI]
                        ↓
                    Gateway (:18789)
                        ↓
                 Agent Runtime
                        ↓
             LLM Adapter (Anthropic/OpenAI/Ollama/MiniMax/etc.)
                        ↓
                  Tools + Memory
```

The **Gateway** is the control plane. It:
- Receives and normalizes input from any channel (WhatsApp, Telegram, Slack, Discord, iMessage, Feishu, web UI, CLI)
- Manages session state and context
- Routes everything to the **Agent Runtime**

The **Agent Runtime** executes the core agentic loop: assemble context from session history and memory → invoke the LLM → execute tool calls → loop until complete → return response.

This separation between interface layer (where messages come from) and intelligence layer (where execution lives) is the key design insight. You get one persistent assistant accessible through any messaging app, with conversation state and tool access managed centrally on your hardware.

## Key Components

### Gateway (Control Plane)

A WebSocket server that runs as a system service. It handles:
- Channel adapters (WhatsApp via Baileys, Telegram via grammY, Slack, Discord, Feishu via WebSocket, etc.)
- The Control UI for web-based interaction
- RPC interface for CLI tools
- Outbound delivery queue (disk-backed)

Default port is `18789`. By default it binds to `loopback` only — meaning it won't accept connections from other machines. For remote access, you use SSH tunnels or Tailscale Serve.

### Agent Runtime

Owns the LLM tool-use loop: call LLM → execute tool → repeat until done. It:
- Streams events back via an in-process pub/sub (`AgentEventStream`)
- Manages session transcript and context compaction
- Supports Anthropic, OpenAI, Ollama, MiniMax, Google Gemini, and any provider with a standard API

### Skills System

Skills extend what the agent can do. They package:
- A `SKILL.md` prompt file that gets injected into the agent's system prompt
- Optional scripts, references, and assets

Skills live under `~/.openclaw/workspace/skills/<skill-name>/`. OpenClaw ships some built-in skills (GitHub, Notion, Tavily, etc.) and you can install more from ClawHub or build your own.

### Memory

OpenClaw maintains memory across sessions:
- **Daily notes** (`memory/YYYY-MM-DD.md`) — raw logs of what happened
- **Long-term memory** (`MEMORY.md`) — curated by the agent after reviewing daily notes
- **Workspace files** — AGENTS.md, SOUL.md, USER.md define the agent's persona and context

## Setup Experience: What I Learned

### Installation

```bash
npm install -g openclaw
openclaw gateway start
openclaw status
```

Node.js 22+ is required. On macOS, the installer sets up a LaunchDaemon for auto-start on boot. On Linux, you manage the service yourself via systemd.

### Connecting to MiniMax M2.7

The primary model I chose is **MiniMax M2.7**, a reasoning-focused model from Chinese AI startup MiniMax with a 204,800 token context window and competitive pricing.

**Configuration in `openclaw.json`:**
```json
{
  "models": {
    "providers": {
      "minimax": {
        "baseUrl": "https://api.minimaxi.com/anthropic",
        "api": "anthropic-messages",
        "authHeader": true
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "minimax/MiniMax-M2.7",
        "fallbacks": ["minimax/MiniMax-M2.5-highspeed"]
      }
    }
  }
}
```

### MiniMax Token Plans

MiniMax offers a **Token Plan** subscription model that bundles requests and model access:

| Plan | Price | M2.7-highspeed |
|------|-------|----------------|
| Basic | $10/month | 4,500 requests/5hrs |
| Pro | $20/month | 15,000 requests/5hrs |
| Enterprise | $50/month | 30,000 requests/5hrs |

On the open pricing model (not subscribed), M2.7 costs:
- **$0.30/M input tokens**
- **$1.20/M output tokens**
- Cache read: $0.06/M

The Token Plan is worth it if you're a heavy user of coding agents like Claude Code, Cline, or Roo Code that integrate with MiniMax's plan. For occasional use, pay-per-token is more economical.

### Models Available

The setup I run includes:

- **MiniMax M2.7** — primary reasoning model (context: 204K, max output: 131K)
- **MiniMax M2.5-highspeed** — faster, cheaper fallback
- **MiniMax Image-01** — image generation model
- **MiniMax Music-2.6** — music generation
- **Ollama (local)** — running qwen3.5:4b and ministral-3:3b on a local machine for completely offline inference

### TTS Setup

OpenClaw supports TTS via MiniMax's `speech-2.8-hd` model with voice `English_expressive_narrator`. Configuration:
```json
{
  "messages": {
    "tts": {
      "provider": "minimax",
      "providers": {
        "minimax": {
          "model": "speech-2.8-hd",
          "voiceId": "English_expressive_narrator"
        }
      }
    }
  }
}
```

### Image & Music Generation

Built-in support via the model config:
```json
{
  "agents": {
    "defaults": {
      "imageModel": { "primary": "minimax/image-01" },
      "musicGenerationModel": { "primary": "minimax/music-2.6" }
    }
  }
}
```

Both work through the standard tool interface — ask for an image or music generation and the agent dispatches to the appropriate model.

### Feishu (Lark) Integration

OpenClaw supports Feishu (also known as Lark), which is popular in China. The configuration:

```json
{
  "channels": {
    "feishu": {
      "enabled": true,
      "appId": "your-app-id",
      "appSecret": "your-app-secret",
      "connectionMode": "websocket",
      "domain": "feishu",
      "dmPolicy": "pairing",
      "allowFrom": ["ou_xxxxxx"],
      "groupPolicy": "open",
      "requireMention": true
    }
  }
}
```

The Feishu integration uses a WebSocket connection mode, which is more efficient than polling. You'll need to create a Feishu app in the [Feishu Open Platform](https://open.feishu.cn/) and configure the bot permissions.

### Ollama Local Models

For completely offline inference, OpenClaw integrates with Ollama:

```json
{
  "models": {
    "providers": {
      "ollama": {
        "baseUrl": "http://192.168.179.1:11434",
        "api": "ollama",
        "apiKey": "OLLAMA_API_KEY",
        "auth": "token",
        "authHeader": true,
        "models": [
          {
            "id": "qwen3.5:4b",
            "name": "qwen3.5:4b",
            "input": ["text", "image"],
            "contextWindow": 262144,
            "maxTokens": 8192
          },
          {
            "id": "ministral-3:3b",
            "name": "ministral-3:3b",
            "input": ["text", "image"],
            "contextWindow": 32768,
            "maxTokens": 8192
          }
        ]
      }
    }
  }
}
```

The Ollama models run locally on your network (in this case at `192.168.179.1:11434`), providing full privacy for whatever you send to them.

## Handling Network Restrictions

### The Corporate Firewall Problem

One of the key lessons learned from my setup: **not all LLM providers are accessible from all networks**.

During setup, I attempted to configure Google Gemini as a fallback model. Testing revealed that `generativelanguage.googleapis.com` is unreachable from my network — curl returns exit code 56 ("failure in receiving network data"), indicating the connection is blocked at the network or ISP level.

This is a common scenario in corporate environments:
- Google domains may be blocked by enterprise firewalls
- Some AI APIs may be restricted by IT policy
- Your proxy (if you have one) may not route certain domains correctly

### What I Tried: Proxy Configuration

OpenClaw supports HTTP proxy configuration via environment variables. I attempted to route Gemini traffic through a proxy at `192.168.179.1:7890`:

```bash
# In ~/.openclaw/.env
HTTP_PROXY=http://192.168.179.1:7890
HTTPS_PROXY=http://192.168.179.1:7890
NO_PROXY=api.minimaxi.com,api.minimax.chat
```

However, this approach had two problems:

1. **The proxy itself couldn't reach Google** — if your proxy's configuration doesn't include Google domains in its routing rules, the connection still fails
2. **Collateral damage** — other channels like WeChat (Weixin) can be impacted by the proxy, causing connection failures

### Lessons Learned

- **Always test connectivity before blaming the application** — use `curl` to verify if a domain is reachable at all
- **Proxy configuration is global** — OpenClaw doesn't yet support per-provider proxy settings; all traffic goes through the same proxy
- **Monitor for collateral damage** — changing proxy settings can affect channels you didn't intend to modify
- **Fallback to local models** — when cloud APIs are blocked, Ollama models running on your local network provide a working alternative

### Current Status

My current working setup:
- ✅ **MiniMax** — primary model, working (API accessible)
- ✅ **Ollama local models** — fallback, working (completely offline)
- ✅ **Feishu** — channel integration, working
- ✅ **WeChat** — channel integration, working
- ❌ **Google Gemini** — blocked at network level (Google APIs unreachable)
- ⚠️ **Tailscale** — set to `off` mode (not needed for my use case)

## Security Considerations

### Gateway Bind Configuration

Out of the box, OpenClaw defaults to `bind: "0.0.0.0:18789"` which exposes the API to all network interfaces. This is fine on a home machine but a real security concern on a shared corporate laptop.

**Always set `gateway.bind: "loopback"` and access remotely via SSH tunnels** unless you're on a trusted network:

```json
{
  "gateway": {
    "bind": "loopback",
    "port": 18789
  }
}
```

### Credential Storage

Credentials are stored in plaintext files under `~/.openclaw/`. This is a known concern — if your machine is compromised, everything is readable. Keep your OpenClaw config directory protected with appropriate file permissions.

### Config Auditing

OpenClaw keeps an audit log of config changes at `~/.openclaw/logs/config-audit.jsonl`. Review this periodically to track what changed and when.

## What You Can Actually Do With It

OpenClaw's real value shows up in the workflows people have built:

- **Morning briefings** — agent checks email, calendar, notifications, summarizes into a single message
- **Automated triage** — email or Slack messages get categorized and routed
- **Coding sessions from a phone** — start a code review or debugging session via WhatsApp
- **Heartbeat monitoring** — scheduled checks that alert you when something needs attention
- **Fully offline** — if you run Ollama locally, the entire stack works without internet
- **Cross-platform access** — Feishu, WeChat, Telegram, Discord — use whatever your team already uses

## The Corporate Network Problem: Why This Matters

Here's the scenario I keep hitting: I'm at work, on a machine that can't reach external AI services. Corporate proxy, firewall, data classification — the usual. I want to use AI to help me review code, write documentation, or debug an issue, but I can't ship our proprietary code to a third-party API, and I can't get a cloud AI tool approved in the time I have.

Before OpenClaw, I'd either:
1. Do without AI assistance (slow, frustrating)
2. Use a local model that barely works for complex tasks (waste of time)
3. Screenshot code and use my phone for AI (ridiculous workaround)

**With OpenClaw running locally on my machine:**

1. I can connect to an LLM that IT *has* approved (or a local Ollama instance)
2. Nothing leaves my machine unless I explicitly push it
3. The agent runs 24/7 in the background, proactively checking things without me prompting
4. It's accessible from any channel I configure — Slack, WhatsApp, Feishu, Telegram, or just a web UI

The key insight: **OpenClaw is infrastructure, not just a chatbot**. The fact that it's a persistent background service means it can be proactive. It can check your email every 30 minutes, monitor a CI pipeline, alert you when something breaks — all without you opening a separate app and asking.

Even in a restricted corporate environment, you can:
- Run Ollama locally for completely offline AI
- Configure the Gateway to bind to loopback only (security-safe)
- Access it via SSH tunnel from your work machine to your home machine running OpenClaw
- Use it as a secure bridge to a cloud LLM you trust for sensitive work
- Use Feishu or WeChat if your team is already on those platforms

## Practical Takeaways

If you're an engineer evaluating OpenClaw, here's what I'd tell you:

1. **Start with the CLI** — `openclaw help` and `openclaw status` give you a clear picture of what's running
2. **Pick your model budget** — MiniMax's Token Plan is excellent value for heavy coding workloads; for occasional use, pay-per-token works fine
3. **Lock down the Gateway config** — set `bind: "loopback"` and use SSH tunnels for remote access; this isn't optional in any multi-user environment
4. **Think in workflows, not chat** — the power comes from heartbeat-driven proactive automation, not from typing questions
5. **Skills are how it grows** — the built-in skills are just the start; when you need something specific to your stack, write a skill for it
6. **Test your network first** — verify that your LLM provider domains are reachable before assuming it's an application problem
7. **Keep a local fallback** — Ollama models ensure you always have *something* working, even when cloud APIs are blocked

## What's Missing

OpenClaw is young (late 2025 start) and moves fast. Rough edges:

- **Documentation is still catching up** — some features aren't well documented, you end up reading source or Discord threads
- **Security model needs care** — plaintext credentials, permissive defaults, no built-in audit logging; treat it like any other privileged service
- **No multi-user support by default** — it's designed as a personal assistant; multi-tenant requires explicit workspace separation
- **Updates can be breaking** — the project is still in rapid evolution; lock versions if you need stability
- **No per-provider proxy configuration** — proxy settings are global; this can be limiting in complex network environments

## The Insight That Matters

OpenClaw's real value proposition isn't "another AI chatbot." It's **local-first AI infrastructure** — the idea that AI can run persistently on your hardware, connected to the tools and channels you already use, without your data leaving your control.

For engineers in corporate environments, this is a genuine unlock. You're not waiting for IT to approve a SaaS tool. You're running AI on hardware you already have, connected to LLMs that may already be approved for your use case.

The tool is not a toy. It's infrastructure. Treat it that way, understand what it can do, and you'll get real value from it.

---

*OpenClaw is MIT-licensed and available on GitHub. It's built in TypeScript (Node.js) and the codebase is a good reference for understanding how to build agentic systems with tool use, session management, and channel adapters.*
