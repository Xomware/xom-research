# Anthropic MCP Builder Skill — Research Overview

> **Status:** Research Complete · February 2026  
> **Source:** https://github.com/anthropics/skills/tree/main/skills/mcp-builder

---

## What Is the MCP Builder Skill?

The `mcp-builder` is one of Anthropic's official **Claude Skills** — a structured system prompt / context pack that instructs Claude on _how_ to build high-quality **Model Context Protocol (MCP) servers**. 

When you load this skill into Claude, it becomes an expert MCP server engineer that follows a battle-tested 4-phase workflow for researching, designing, implementing, and validating MCP servers that enable LLMs to interact with any external service or API.

Think of it as a **blueprint + expert mentor** that lives inside Claude's context window.

---

## What Does It Generate?

Given a target API or service, Claude (guided by the mcp-builder skill) will produce:

- A **complete, production-ready MCP server** in TypeScript or Python
- Proper project structure (`src/`, `tools/`, `services/`, `schemas/`)
- **Typed tool definitions** using Zod (TypeScript) or Pydantic (Python)
- Authentication & error handling built in
- **10 evaluation Q&A pairs** to test the server's effectiveness
- A `README.md` for the generated server

---

## How It Works — The 4-Phase Workflow

| Phase | Name | What Happens |
|-------|------|--------------|
| 1 | **Research & Planning** | Claude reads MCP spec, SDK docs, and the target API docs |
| 2 | **Implementation** | Claude writes the server code following strict conventions |
| 3 | **Review & Test** | Claude audits code quality, runs build, tests with MCP Inspector |
| 4 | **Evaluations** | Claude creates 10 hard Q&A pairs to validate the server's LLM usability |

The skill provides Claude with:
- Reference guides for TypeScript and Python implementation patterns
- MCP best practices document (naming, pagination, transport selection)
- An evaluation harness (Python script) to run the generated evals against Claude

---

## Key Design Principles

1. **Comprehensive API coverage over workflow shortcuts** — prefer many focused tools over a few complex ones
2. **LLM-usable over human-usable** — tool descriptions and schemas are optimized for agent understanding
3. **TypeScript by default** — better SDK support, static typing, broad AI model training coverage
4. **Streamable HTTP for remote, stdio for local** — SSE is deprecated
5. **Evaluations as quality gate** — the real measure of an MCP server is whether an LLM can use it to answer real questions

---

## Files In This Research Folder

| File | Contents |
|------|----------|
| `README.md` | This overview |
| `how-it-works.md` | Technical deep-dive: file structure, prompts, conventions |
| `use-cases.md` | Practical use cases, including a Debtwire placeholder |
| `getting-started.md` | Step-by-step guide to using mcp-builder with Claude |

---

## Quick Links

- [Anthropic Skills Repo](https://github.com/anthropics/skills)
- [MCP Builder Skill](https://github.com/anthropics/skills/tree/main/skills/mcp-builder)
- [MCP Protocol Docs](https://modelcontextprotocol.io)
- [TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)
- [Python SDK (FastMCP)](https://github.com/modelcontextprotocol/python-sdk)
