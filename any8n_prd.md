# PRD: any8n — Agent Distribution & Orchestration Layer

## 1. One-liner
Fork n8n's visual workflow engine and turn it into the connective tissue for AI agents: vibe-build agents or import existing ones (yours or third-party), wire them into bigger workflows as nodes, and deploy the result as an API, webhook, schedule, or chat surface.

## 2. Problem
Everyone is building agents. Almost no one is building the layer that lets agents:
- Talk to other agents regardless of framework (LangChain, OpenAI Assistants, Claude Agent SDK, custom HTTP).
- Get composed into larger pipelines without custom glue code.
- Get discovered, reused, and deployed by someone other than their original author.

n8n already solved visual composition + deployment for *apps and APIs*. It hasn't solved it for *autonomous, stateful, LLM-driven agents* as first-class nodes.

## 3. Vision
"any8n" = any agent, any workflow, any deployment target.
- **Vibe-create**: describe a workflow in natural language → get a working agent graph.
- **Compose**: drop in your own agents AND agents published by others as nodes.
- **Deploy**: one click → API endpoint, webhook trigger, cron job, or embeddable chat widget.
- **Govern**: every third-party agent node runs sandboxed, with cost caps, audit logs, and policy gates (reuse StacyVM/Blackbox primitives).

## 4. Target Users
- Indie devs/hackers building multi-agent systems who don't want to write orchestration glue.
- Teams who want to mix internal agents with marketplace agents (e.g., "research agent" + "your CRM agent").
- Agencies/builders who want to package and resell agent workflows.

## 5. Core Concepts

### 5.1 Agent Node
The atomic unit. A wrapper around any agent that exposes:
- `manifest.json`: name, description, capabilities, input schema, output schema, cost/latency profile, auth requirements, trust/version info.
- `invoke()`: standard call interface (sync, async/streaming, or long-running with callback).
- `health()`: liveness/readiness check.

Supported backends (adapters):
- Claude Agent SDK / MCP servers (priority — first-class)
- OpenAI Assistants API
- LangChain/LangGraph agents (HTTP-wrapped)
- Generic HTTP/webhook agents
- Raw LLM calls with tools (fallback "dumb agent" node)

### 5.2 Workflow Canvas
n8n's existing React Flow canvas, extended with:
- New node category: "Agents" (alongside Trigger, Action, Logic).
- Agent nodes show live cost/latency estimates and a trust badge.
- Conditional/branching on agent *reasoning output*, not just structured data (LLM-as-router node).

### 5.3 Vibe-Create
Natural language → workflow graph:
- User prompt → planning agent decomposes into steps → maps steps to existing nodes/agents from registry where possible, generates new agent nodes (via Claude) where not.
- Output: an editable n8n-style graph the user can tweak before deploying.

### 5.4 Agent Registry / Marketplace
- Publish: any user can publish an agent node (manifest + endpoint) to a shared registry.
- Discover: semantic search over manifests (embedding-based, reuse multimodal retrieval engine work).
- Trust: version pinning, usage-based trust score, sandboxed execution by default for non-self agents.
- Monetization (later): per-call pricing, revenue split.

### 5.5 Deploy Targets
Same as n8n today, extended:
- REST API endpoint (auto-generated OpenAPI spec)
- Webhook trigger
- Scheduled/cron execution
- Chat interface (embeddable widget, Slack/Discord bot)
- "Agent-as-a-node": the deployed workflow itself gets a manifest and can be reused as a node in someone else's workflow (recursive composability)

## 6. MVP Scope (v0)

In scope:
1. Fork n8n core; add new node type `AgentNode` with manifest-driven config UI.
2. Build adapters for: Claude Agent SDK/MCP, generic HTTP agent, OpenAI Assistants.
3. Vibe-create: single prompt → generated workflow graph (using Claude to plan + select/generate nodes).
4. Local agent registry (single-tenant, JSON-based) — no marketplace yet, just "save and reuse my own agent nodes."
5. Deploy as: REST API + webhook (reuse n8n's existing deploy infra).
6. Basic sandboxing: each AgentNode execution runs in an isolated container/process with resource + cost limits (StacyVM integration point).

Out of scope for v0:
- Public marketplace / cross-user discovery
- Billing/revenue split
- Trust scoring algorithm
- Chat widget deploy target
- Recursive "workflow-as-agent-node"

## 7. Architecture Sketch

```
┌─────────────────────────────────────────────┐
│              Canvas (React Flow)             │
│  Trigger → Agent Node → Logic → Agent Node   │
└───────────────┬───────────────────────────────┘
                 │
        ┌────────▼─────────┐
        │  Workflow Engine  │  (n8n core, forked)
        └────────┬─────────┘
                 │
     ┌───────────▼────────────┐
     │   AgentNode Executor    │
     │  - loads manifest       │
     │  - picks adapter        │
     │  - sandbox (StacyVM)    │
     └───────────┬────────────┘
                 │
   ┌─────────────┼─────────────────┐
   ▼             ▼                 ▼
Claude Agent   OpenAI Assistant   Generic HTTP
SDK/MCP        API                Agent
```

## 8. Key Risks
- **Adapter sprawl**: every agent framework has different invocation/streaming semantics. Mitigate by defining a strict minimal interface (invoke/health/manifest) and pushing complexity into adapters, not core.
- **Cost runaway**: composed agent workflows can multiply LLM calls. Mitigate with per-node cost caps and workflow-level budget enforcement (Blackbox-style policy gating).
- **Trust for third-party agents**: a malicious/buggy agent node in someone's workflow is a security and reputation risk. v0 sidesteps this by scoping to self-published agents only; sandboxing is still mandatory from day one so the architecture doesn't need rework later.
- **n8n fork maintenance**: diverging too far from upstream n8n makes future merges painful. Keep AgentNode as an additive plugin/node-package where possible rather than core engine changes.

## 9. Success Metrics (v0)
- Time from "describe workflow in English" to "working deployed agent workflow" < 5 minutes.
- At least 3 distinct agent frameworks supported via adapters.
- A user can build a workflow combining 2+ agent nodes + 1 traditional n8n action node (e.g., Slack, Google Sheets) without writing code.

## 10. Roadmap After v0
1. Public registry + semantic agent discovery.
2. Recursive composability (deployed workflow = reusable agent node).
3. Trust/audit layer (OWASP Agentic Top 10 + EU AI Act mapping, per StacyVM Blackbox work).
4. Chat-based deploy target + Slack/Discord bots.
5. Marketplace monetization.
