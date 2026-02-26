# MCP Builder — Use Cases

> Practical applications of the mcp-builder skill. What can we actually build with it?

---

## Overview

The mcp-builder skill works best when you have:
1. A **well-documented API** (REST, GraphQL, etc.)
2. A clear **use case** — what should Claude be able to do with this service?
3. A **target client** (Claude Desktop, OpenClaw, custom agent)

The skill produces a server; the server then becomes a persistent tool extension available to any Claude-powered agent.

---

## General Use Case Patterns

### Pattern 1: Internal Data Sources
Connect Claude to internal databases, internal APIs, or proprietary data systems so agents can query business data without manual data export.

- Read dashboards, filter records, aggregate metrics
- Ideal for: finance tools, operations systems, analytics platforms

### Pattern 2: SaaS Integration
Wrap third-party SaaS APIs (Jira, Salesforce, HubSpot, Slack) so Claude agents can perform multi-step workflows across systems.

- Create tickets, search records, send messages, update statuses
- Ideal for: ops automation, customer support, project management

### Pattern 3: Research & Data Retrieval
Connect Claude to research databases, news APIs, market data feeds, or document repositories for deep research workflows.

- Fetch, filter, paginate, and synthesize large datasets
- Ideal for: financial research, media monitoring, competitive intelligence

### Pattern 4: Workflow Automation
Build MCP servers that combine multiple API calls into workflow-level tools — so Claude can execute a multi-step business process with a single tool call.

- Chain: fetch → transform → write → notify
- Ideal for: reporting automation, data pipelines, approval workflows

---

## Specific Use Cases

### ✅ GitHub / Version Control
Build a GitHub MCP server to let Claude agents:
- Browse repos, issues, PRs, and comments
- Create issues, review diffs, post comments
- Search code across repos
- Generate release notes from commit history

**Complexity:** Medium | **Language:** TypeScript recommended

---

### ✅ Slack / Team Communication
Let Claude:
- Search message history
- Post to channels
- Look up user profiles and channel membership
- Monitor for keywords and surface relevant conversations

**Complexity:** Medium | **Transport:** Streamable HTTP (for push notifications)

---

### ✅ Financial Data APIs (Bloomberg, Refinitiv, etc.)
Connect Claude to market data for research workflows:
- Fetch company financials, ratings, news
- Search by ticker, CUSIP, ISIN
- Paginate through large result sets
- Compare metrics across entities

**Complexity:** High (large APIs, auth complexity)

---

### ✅ Internal Knowledge Bases
Wrap your internal wiki, Notion, or Confluence in an MCP server:
- Search articles
- Retrieve page content
- Surface related documentation

**Complexity:** Low–Medium

---

## 🔬 Planned Exploration: Debtwire

> **Status: TBD / Planned**

Debtwire is a credit and restructuring news/data platform used in leveraged finance and distressed debt analysis. Dom is exploring whether an MCP server for Debtwire would enable Claude to:

- **Search and retrieve distressed debt news articles** by company, industry, or deal type
- **Pull deal data** — restructuring timelines, creditor lists, advisor assignments
- **Monitor credits** — flag when a company coverage changes or new articles are published
- **Cross-reference** against broader market data (e.g., bond prices, ratings)
- **Generate research summaries** from raw Debtwire content, structured for analyst workflows

### Why This Is Interesting
Debtwire content is text-heavy, semi-structured, and requires contextual understanding to extract signal. An LLM with MCP access to Debtwire could dramatically accelerate the workflow of a restructuring or credit analyst — surfacing relevant deals, summarizing creditor dynamics, and flagging key developments without manual searching.

### Open Questions
- Does Debtwire expose a usable API? (vs. web scraping)
- What authentication model do they use?
- What are the content licensing implications of passing Debtwire content to an LLM?
- Is there a "read-only" API tier appropriate for research use?

### Next Steps
- [ ] Contact Debtwire / check if they have a developer/API program
- [ ] Review Debtwire data model (articles, companies, deals, creditors)
- [ ] Prototype a minimal MCP server if API access is granted
- [ ] Define 5 high-value analyst use cases to drive tool design

---

## Evaluation Criteria for New Use Cases

Before building an MCP server for a new service, ask:

1. **Is there a real API?** (Documented, stable, accessible)
2. **What's the read/write balance?** (Read-heavy = safer to build fast)
3. **How complex is auth?** (API key vs. OAuth vs. custom token)
4. **What's the primary LLM workflow?** (Research? Action? Monitoring?)
5. **Can we write good evals?** (If we can't imagine 10 questions, maybe the use case isn't clear)
