# How MCP Builder Works — Technical Breakdown

> Deep dive into the file structure, prompts, conventions, and internal workflow of the `mcp-builder` skill.

---

## Skill File Structure

```
skills/mcp-builder/
├── SKILL.md                      # The main skill prompt (loaded into Claude's context)
├── LICENSE.txt
├── reference/
│   ├── mcp_best_practices.md     # Universal conventions for all MCP servers
│   ├── node_mcp_server.md        # TypeScript/Node implementation guide + examples
│   ├── python_mcp_server.md      # Python/FastMCP implementation guide + examples
│   └── evaluation.md             # Evaluation creation guidelines + XML format
└── scripts/
    ├── connections.py             # MCP transport helpers for eval harness
    ├── evaluation.py              # Evaluation runner — runs Q&A against Claude + MCP
    ├── example_evaluation.xml    # Sample evaluation file format
    └── requirements.txt          # Python deps for evaluation runner
```

### How The Skill Gets Loaded

When Claude uses the `mcp-builder` skill, `SKILL.md` is loaded into the system prompt. It references the `reference/` files which Claude fetches on-demand via WebFetch during the workflow. The scripts are used _after_ building the server to run evaluations.

---

## The SKILL.md Prompt — What It Tells Claude

`SKILL.md` is essentially a highly-structured **system prompt with instructions**. Key things it establishes:

### 1. Goal / Role
> _"Create MCP servers that enable LLMs to interact with external services through well-designed tools. The quality of an MCP server is measured by how well it enables LLMs to accomplish real-world tasks."_

### 2. The 4-Phase Process (Enforced Order)

Claude is instructed to follow phases strictly in sequence:

**Phase 1: Research & Planning**
- Read the MCP spec from `https://modelcontextprotocol.io/sitemap.xml`
- Load the TypeScript or Python SDK README from GitHub
- Load `reference/mcp_best_practices.md`
- Study the target service's API documentation
- Plan which endpoints/tools to implement (prioritize comprehensive coverage)

**Phase 2: Implementation**
- Set up project structure per the language guide
- Implement API client with auth, error handling, pagination
- Implement each tool with typed schemas (Zod / Pydantic)
- Follow naming conventions strictly

**Phase 3: Review & Test**
- Check for DRY code, consistent errors, full type coverage
- Run `npm run build` (TypeScript) or `python -m py_compile` (Python)
- Test with MCP Inspector: `npx @modelcontextprotocol/inspector`

**Phase 4: Evaluations**
- Use READ-ONLY operations on the live service to explore available data
- Create 10 complex, realistic Q&A pairs
- Output as XML file in `evaluation.xml`

---

## Naming Conventions

These are enforced by `mcp_best_practices.md`:

| Element | Python | TypeScript |
|---------|--------|------------|
| **Server name** | `{service}_mcp` | `{service}-mcp-server` |
| **Tool name** | `service_action_resource` | `service_action_resource` |
| **Example** | `slack_send_message` | `github_create_issue` |

Rules:
- Always include service prefix on tool names (to avoid conflicts when multiple MCPs are loaded)
- Action-oriented verbs: `get`, `list`, `search`, `create`, `update`, `delete`
- snake_case always

---

## TypeScript Project Structure (Generated)

```
{service}-mcp-server/
├── package.json
├── tsconfig.json
├── README.md
├── src/
│   ├── index.ts          # McpServer init + transport setup
│   ├── types.ts          # TypeScript interfaces
│   ├── constants.ts      # API_URL, limits, enums
│   ├── tools/            # Tool implementations (one file per domain)
│   ├── services/         # API client, auth helpers
│   └── schemas/          # Zod validation schemas
└── dist/                 # Compiled output
```

### Key TypeScript Pattern

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";
import { z } from "zod";

const server = new McpServer({ name: "github-mcp-server", version: "1.0.0" });

server.registerTool(
  "github_list_repos",
  {
    title: "List Repositories",
    description: "List repositories for a user or organization",
    inputSchema: {
      owner: z.string().describe("Username or org name"),
      limit: z.number().int().min(1).max(100).default(30)
    },
    annotations: { readOnlyHint: true, destructiveHint: false }
  },
  async ({ owner, limit }) => {
    const repos = await fetchRepos(owner, limit);
    return { content: [{ type: "text", text: JSON.stringify(repos) }] };
  }
);
```

**Important:** Use `server.registerTool()` — NOT the deprecated `server.tool()`.

---

## Python Project Structure (Generated)

```python
from mcp.server.fastmcp import FastMCP
from pydantic import BaseModel, Field

mcp = FastMCP("github_mcp")

class ListReposInput(BaseModel):
    owner: str = Field(..., description="Username or org name")
    limit: int = Field(default=30, ge=1, le=100)

@mcp.tool(
    name="github_list_repos",
    annotations={"readOnlyHint": True, "destructiveHint": False}
)
async def list_repos(params: ListReposInput) -> str:
    repos = await fetch_repos(params.owner, params.limit)
    return json.dumps(repos)
```

---

## Transport Selection Logic

The skill instructs Claude to choose transport based on deployment context:

| Criterion | Use stdio | Use Streamable HTTP |
|-----------|-----------|---------------------|
| Deployment | Local only | Remote/cloud |
| Clients | Single user | Multiple clients |
| Complexity | Simple CLI | Web service |

**Note:** SSE (Server-Sent Events) transport is explicitly deprecated — do not use.

---

## Response Format Design

All tools should support dual output:
- **`json`** — structured data for programmatic use, all fields, metadata
- **`markdown`** (default) — human-readable, timestamps converted, display names shown

Pagination metadata always included:
```json
{ "total": 150, "count": 20, "offset": 0, "has_more": true, "next_offset": 20 }
```

---

## Tool Annotations (Metadata Hints for Clients)

| Annotation | Type | Meaning |
|------------|------|---------|
| `readOnlyHint` | boolean | Tool doesn't modify state |
| `destructiveHint` | boolean | Tool may delete/overwrite data |
| `idempotentHint` | boolean | Safe to call multiple times with same args |
| `openWorldHint` | boolean | Tool reaches out to external systems |

These are hints for client UIs — not security enforcements.

---

## Evaluation System

The skill includes a complete eval harness:

### Evaluation XML Format
```xml
<evaluation>
  <qa_pair>
    <question>Which repository in the org has the most open issues as of today?</question>
    <answer>my-repo-name</answer>
  </qa_pair>
</evaluation>
```

### Evaluation Runner (`evaluation.py`)
- Connects to a running MCP server
- Sends each question to Claude + the MCP tools
- Claude uses the tools to find the answer
- Answers are compared; pass/fail reported
- Also collects structured feedback on tool quality

### Evaluation Quality Criteria
Questions must be:
- Independent (don't depend on other questions)
- Read-only (no writes)
- Complex (multiple tool calls, potentially paging)
- Realistic (real use case humans would have)
- Stable (answer won't change over time)
- **Not** solvable by simple keyword search

---

## Security Patterns

The skill enforces these in the generated code:
- API keys via environment variables only — never hardcoded
- Input validation via Zod/Pydantic on all inputs
- Path sanitization to prevent directory traversal
- Actionable error messages that don't expose internal state
- DNS rebinding protection for local HTTP servers (bind `127.0.0.1`, validate `Origin` header)
- OAuth 2.1 for services that support it
