# Getting Started with MCP Builder

> Step-by-step guide to building a custom MCP server using the Anthropic `mcp-builder` skill and Claude.

---

## Prerequisites

Before you start, make sure you have:

- [ ] Node.js 18+ (for TypeScript servers) or Python 3.10+ (for Python servers)
- [ ] npm / npx available
- [ ] Claude access (Claude.ai Pro, API, or OpenClaw with Claude)
- [ ] A target service with a REST/JSON API you want to connect
- [ ] API credentials for that service (API key, OAuth token, etc.)

---

## Step 1: Load the MCP Builder Skill

The skill lives in the Anthropic Skills repo. You have two options:

### Option A: Load SKILL.md directly into Claude
1. Open the skill file: https://github.com/anthropics/skills/blob/main/skills/mcp-builder/SKILL.md
2. Copy the raw content
3. Paste it as a system prompt or at the start of your Claude conversation
4. Tell Claude: _"I want to build an MCP server for [service name]. Here's the API documentation: [paste or link docs]"_

### Option B: Use Claude's built-in skill support (if available)
If your Claude deployment supports Anthropic Skills natively, activate the `mcp-builder` skill from the skills library.

### Option C: Use OpenClaw with a skill config
If running in OpenClaw, add the skill to your project config and reference it in the system prompt.

---

## Step 2: Brief Claude on the Target Service

Once the skill is loaded, give Claude a clear brief:

```
I want to build an MCP server for the [Service Name] API.

Target language: TypeScript (or Python)
Transport: stdio (local) or Streamable HTTP (remote/cloud)

Here's the API documentation: [paste docs or provide URL]

Key capabilities I want to expose:
- [e.g., search articles by keyword]
- [e.g., retrieve company profiles]  
- [e.g., list recent deals]
- [e.g., fetch document content by ID]

Authentication: [API key via header / OAuth2 / etc.]
```

---

## Step 3: Let Claude Run the 4-Phase Workflow

Claude will work through the phases. Your role at each phase:

### Phase 1: Research (Claude does this autonomously)
Claude will:
- Fetch and read the MCP spec
- Load the TypeScript or Python SDK README
- Review the MCP best practices guide
- Study your API docs

**Your job:** Answer any clarifying questions about the API. Provide access to API docs if Claude can't reach them via web.

### Phase 2: Implementation (Claude generates code)
Claude will write:
- Project scaffold (`package.json`, `tsconfig.json`, folder structure)
- API client with auth
- All tool implementations with typed schemas
- Error handling and pagination

**Your job:** Review the generated code structure. Ask Claude to add/remove tools if needed. Make sure auth config matches your actual credentials setup.

### Phase 3: Review & Test
Claude will:
- Self-review for DRY, types, consistent errors
- Ask you to run: `npm run build` or `python -m py_compile`
- Guide you through testing with MCP Inspector

**Your job:**
```bash
# TypeScript
cd my-service-mcp-server
npm install
npm run build
npx @modelcontextprotocol/inspector dist/index.js
```

Share any build errors back to Claude for fixes.

### Phase 4: Evaluations
Claude will use read-only tool calls to explore the live service, then generate an `evaluation.xml` file with 10 hard Q&A pairs.

**Your job:** Review the questions — they should be realistic and non-trivial. If any question requires write access or is too simple, ask Claude to replace it.

---

## Step 4: Run the Evaluation Harness

Clone the eval scripts from the skills repo:

```bash
# Get the evaluation scripts
git clone https://github.com/anthropics/skills.git
cd skills/skills/mcp-builder/scripts
pip install -r requirements.txt
```

Run evaluation against your server:

```bash
python evaluation.py \
  --server-path /path/to/your/server/dist/index.js \
  --eval-file /path/to/evaluation.xml \
  --model claude-opus-4-5
```

The harness will:
1. Start your MCP server
2. Connect Claude to it
3. Run each Q&A pair — Claude uses your tools to find the answer
4. Report pass/fail + tool quality feedback

**Target:** 7+/10 questions answered correctly = good server quality.

---

## Step 5: Connect to Claude Desktop or OpenClaw

### Claude Desktop (`claude_desktop_config.json`)
```json
{
  "mcpServers": {
    "my-service": {
      "command": "node",
      "args": ["/path/to/my-service-mcp-server/dist/index.js"],
      "env": {
        "API_KEY": "your-api-key-here"
      }
    }
  }
}
```

### OpenClaw
Add to your `.openclaw` config or reference the MCP server in your skill/agent config. (Check your OpenClaw version's MCP support docs.)

### Remote Server (Streamable HTTP)
If you built with HTTP transport, deploy to a cloud host (Railway, Fly.io, AWS Lambda) and point your client at the endpoint URL.

---

## Common Issues & Fixes

| Problem | Fix |
|---------|-----|
| Build fails with type errors | Share the full error with Claude, it will fix |
| MCP Inspector can't connect | Check that server starts without errors first |
| Tools not showing up in Claude | Verify server registration — use `registerTool()` not `tool()` |
| Pagination breaks | Make sure `has_more` + `next_offset` are returned |
| Auth failing | Check env var names match exactly between server code and config |
| Eval score < 5/10 | Tool descriptions may be vague — ask Claude to improve them |

---

## Tips for Better Results

1. **Provide complete API docs** — the more Claude knows about the API, the better the tool design
2. **Specify your primary use cases upfront** — Claude will prioritize those tools
3. **Prefer TypeScript** — better SDK support and AI model familiarity
4. **Test eval questions yourself first** — if you can't answer them manually, neither can Claude
5. **Iterate on tool descriptions** — the eval feedback is gold for improving tool discoverability
6. **Start small** — 5-10 core tools first, expand after eval passes

---

## Quick Reference: Key Commands

```bash
# Build (TypeScript)
npm run build

# Test locally
npx @modelcontextprotocol/inspector dist/index.js

# Run evals
python evaluation.py --server-path dist/index.js --eval-file evaluation.xml

# Python syntax check
python -m py_compile server.py

# Python run (FastMCP)
python server.py
```

---

## Resources

- [MCP Builder Skill](https://github.com/anthropics/skills/tree/main/skills/mcp-builder)
- [MCP Protocol Docs](https://modelcontextprotocol.io)
- [TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)
- [Python SDK / FastMCP](https://github.com/modelcontextprotocol/python-sdk)
- [MCP Inspector](https://github.com/modelcontextprotocol/inspector)
- [Anthropic Skills Repo](https://github.com/anthropics/skills)
