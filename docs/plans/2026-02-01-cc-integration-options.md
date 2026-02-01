# Claude Code Integration Options for OpenClaw

**Date:** 2026-02-01
**Status:** Design Exploration

## Context

OpenClaw is a personal AI assistant supporting multiple communication channels (WhatsApp, Telegram, Slack, Discord, iMessage, etc.) and model providers (Anthropic, OpenAI, Google Gemini, Z.ai, and 20+ others).

**Goal:** Integrate Claude Code (via the Anthropic Agent SDK) as a tool that OpenClaw can delegate tasks to—specifically for coding, planning, and complex reasoning—while using a Claude Pro/Max subscription within Anthropic's approved environment.

**TOS Consideration:** This approach is compliant because Claude Code/Agent SDK is used legitimately. The subscription is consumed within Anthropic's approved CLI or SDK environment, not via third-party API spoofing (which is what got OpenCode and similar tools banned).

---

## Architecture Overview

```
┌─────────────────┐     HTTP/WebSocket/MCP    ┌──────────────────────┐
│   OpenClaw      │ ────────────────────────> │  Agent SDK Server    │
│  (primary LLM)  │                            │  (Claude sub-agent)  │
└─────────────────┘                            └──────────────────────┘
                                                           │
                                                           ▼
                                                  ┌─────────────────┐
                                                  │  Claude API     │
                                                  │  (Subscription) │
                                                  └─────────────────┘
```

OpenClaw delegates specialized tasks to Claude Code via one of the integration patterns below. Claude Code uses the subscription legitimately.

---

## Integration Options

### Option 1: MCP Server Integration

**Description:** Use an existing MCP server that wraps Claude Code, connect it to OpenClaw's MCP integration.

**Existing Solution:** [steipete/claude-code-mcp](https://github.com/steipete/claude-code-mcp)
- Runs Claude Code in "one-shot mode" as an MCP server
- Automatically bypasses permissions
- Creates "an agent in your agent" pattern

**Pros:**
- Existing implementation, no code to write
- OpenClaw has native MCP support
- Minimal configuration required

**Cons:**
- May need to verify OpenClaw MCP compatibility
- Limited control over execution environment
- One-shot mode may not support long-running sessions

**Research Prompt:**
> You are a researcher investigating OpenClaw's MCP integration capabilities. Your task is to determine if OpenClaw can connect to external MCP servers (not just provide them).
>
> Specifically investigate:
> 1. How does OpenClaw configure and connect to external MCP servers?
> 2. Is there a client-side MCP connector in the codebase?
> 3. What configuration format is expected for MCP server connections?
> 4. Are there any existing skills or extensions that connect to external MCP servers?
> 5. Review the steipete/claude-code-mcp server implementation—what endpoints/tools does it expose?
>
> Search in src/, packages/, extensions/, and docs/ for MCP client configuration patterns.
> Return specific file paths, configuration examples, and compatibility assessment.

---

### Option 2: Agent SDK HTTP/WebSocket Service

**Description:** Run the Anthropic Agent SDK as a standalone HTTP/WebSocket server that OpenClaw can call via a custom skill or HTTP tool.

**Existing Solutions:**

| Project | Type | Link | Notes |
|---------|------|------|-------|
| claude-agent-server | WebSocket (Bun/TS) | [GitHub](https://github.com/dzhng/claude-agent-server) | Production-ready, E2B sandbox support |
| claude-multi-agent-api-server | FastAPI (Python) | [GitHub](https://github.com/is0383kk/claude-multi-agent-api-server) | Session management, cost tracking |
| claude-agent-sdk-container | REST API + Docker | [GitHub](https://github.com/receipting/claude-agent-sdk-container) | Docker-ready, multi-agent |

**Pros:**
- Full control over execution environment
- Can run in Docker/container for isolation
- Session persistence and resumption
- Can be deployed anywhere (local, cloud)

**Cons:**
- Need to host/maintain the service
- Additional infrastructure component
- Network latency if deployed remotely

**Research Prompt:**
> You are a researcher designing an OpenClaw skill that delegates tasks to an external HTTP/WebSocket service running the Anthropic Agent SDK.
>
> Your task is to:
> 1. Study the existing HTTP service implementations (dzhng/claude-agent-server, is0383kk/claude-multi-agent-api-server)
> 2. Document their API endpoints, message formats, and authentication methods
> 3. Analyze the OpenClaw skill system—find examples of skills that make HTTP calls to external services
> 4. Identify the best pattern for an OpenClaw skill to:
>    - Send a task to the Agent SDK service
>    - Stream or poll for results
>    - Handle timeouts and errors
> 5. Determine if OpenClaw's existing tools (like sessions_spawn) can be adapted or if a new skill type is needed
>
> Return: Recommended API specification for the Agent SDK service, example skill code structure, and integration steps.

---

### Option 3: Native Agent SDK Skill (TypeScript)

**Description:** Create an OpenClaw skill that directly imports and uses the Anthropic Agent SDK TypeScript package, running Claude Code within the OpenClaw process.

**Existing Solution:** [anthropics/claude-agent-sdk-typescript](https://github.com/anthropics/claude-agent-sdk-typescript)

```typescript
import { query } from '@anthropic-ai/claude-agent-sdk'

async function delegateToClaudeCode(prompt: string) {
  for await (const message of query({ prompt })) {
    // Process results
  }
}
```

**Pros:**
- No external service dependency
- Lowest latency (in-process)
- Full access to Agent SDK features
- Can share OpenClaw's workspace and context

**Cons:**
- May compete for resources with primary OpenClaw agent
- More complex installation (Agent SDK dependency)
- Potential isolation concerns

**Research Prompt:**
> You are a researcher investigating how to create an OpenClaw skill that directly uses the Anthropic Agent SDK TypeScript package.
>
> Your task is to:
> 1. Study the OpenClaw skill system—find example skills in extensions/ and skills/
> 2. Document the skill interface, lifecycle, and how skills receive/return data
> 3. Investigate how skills are packaged and distributed
> 4. Review the claude-agent-sdk-typescript package—determine dependencies and runtime requirements
> 5. Find examples of skills that make heavy use of external npm packages
> 6. Identify any potential conflicts between the Agent SDK and OpenClaw's Pi SDK
>
> Return: A recommended skill architecture, file structure, example skill skeleton code, and installation approach.

---

### Option 4: TMUX/CLI Wrapper

**Description:** Run Claude Code CLI in a tmux session, control it programmatically via shell commands from an OpenClaw skill.

**Existing Pattern:** Community practice for long-running autonomous sessions with Claude Code.

**Pros:**
- Uses official Claude Code CLI unchanged
- Subscription used exactly as intended
- Can attach/detach from sessions
- Visual debugging possible

**Cons:**
- Fragile (screen scraping, shell state)
- Harder to extract structured output
- Tmux dependency
- May not work well on Windows

**Research Prompt:**
> You are a researcher investigating the viability of controlling Claude Code CLI via tmux from an OpenClaw skill.
>
> Your task is to:
> 1. Search for existing tools that wrap Claude Code via tmux or similar terminal control
> 2. Document how Claude Code CLI's non-interactive mode works (flags, input/output formats)
> 3. Investigate OpenClaw's shell/terminal capabilities—can it spawn and control tmux sessions?
> 4. Find examples of OpenClaw skills that use shell commands or terminal control
> 5. Assess Windows compatibility (WSL2, Git Bash, etc.)
> 6. Research alternative terminal control approaches (node-pty, xterm.js, etc.)
>
> Return: Feasibility assessment, recommended approach, and example implementation pattern.

---

## Comparison Summary

| Option | Complexity | Maintenance | Windows Support | TOS Safe |
|--------|------------|-------------|-----------------|----------|
| MCP Server | Low | Low | TBD | ✅ |
| HTTP/WebSocket Service | Medium | Medium | ✅ | ✅ |
| Native SDK Skill | Medium | Low | ✅ | ✅ |
| TMUX/CLI Wrapper | High | High | ⚠️ WSL only | ✅ |

---

## Recommended Next Steps

1. **Research Phase:** Run the research prompts above in parallel using specialized agents
2. **Proof of Concept:** Build a simple integration for the most promising option
3. **Documentation:** Document the integration pattern for the community
4. **Optional:** Contribute back to OpenClaw as a first-party integration

---

## Sources

- [OpenClaw Documentation](https://docs.openclaw.ai)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [steipete/claude-code-mcp](https://github.com/steipete/claude-code-mcp)
- [dzhng/claude-agent-server](https://github.com/dzhng/claude-agent-server)
- [is0383kk/claude-multi-agent-api-server](https://github.com/is0383kk/claude-multi-agent-api-server)
- [receipting/claude-agent-sdk-container](https://github.com/receipting/claude-agent-sdk-container)
- [anthropics/claude-agent-sdk-typescript](https://github.com/anthropics/claude-agent-sdk-typescript)
- [anthropics/claude-agent-sdk-python](https://github.com/anthropics/claude-agent-sdk-python)
- [Hosting the Agent SDK - Claude Docs](https://platform.claude.com/docs/en/agent-sdk/hosting)
- [Agent SDK MCP Integration - Claude Docs](https://platform.claude.com/docs/en/agent-sdk/mcp)
