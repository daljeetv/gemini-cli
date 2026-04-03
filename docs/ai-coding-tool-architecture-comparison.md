# Architecture Comparison: Opencode vs Claude Code vs Pi

A deep analysis of the design decisions, tradeoffs, and differentiation opportunities across three major open-source AI coding CLI tools — based on cloning and reading their source code.

---

## 1. High-Level Overview

| Dimension | **Opencode** | **Claude Code** | **Pi** |
|---|---|---|---|
| Language | Go 1.24 | TypeScript (single bundled JS) | TypeScript (ESM monorepo) |
| LOC (approx) | ~15K Go | ~7.5K minified (core closed-source) | ~4K TS across 7 packages |
| TUI Framework | Bubbletea (Elm architecture) | Ink/React | Custom differential renderer |
| LLM Providers | 11 (Anthropic, OpenAI, Gemini, Bedrock, etc.) | Anthropic-only (+ Bedrock/Vertex/Foundry variants) | 10+ (Anthropic, OpenAI, Google, xAI, Groq, etc.) |
| Extension Model | MCP + custom commands (.md) | Plugins + Hooks + MCP + Agents + Skills | Extensions (TS) + Skills (.md) + Packages |
| Persistence | SQLite with migrations | Session files (context compaction) | JSONL append-only logs |
| Open-source scope | Fully open | Ecosystem only (core is compiled) | Fully open |
| Stars | ~126K | N/A (vendor-backed) | ~31K |

---

## 2. Core Agent Loop Comparison

All three follow the standard LLM agent loop (prompt → tool calls → execute → loop), but diverge significantly in execution strategy and control flow.

### Opencode: Sequential, Persistence-Heavy

```
User message → Load history from SQLite → Stream LLM response →
Execute tool calls sequentially → Persist to DB → Loop or return
```

- **Tool execution**: Sequential within a single LLM response. Each tool call blocks until complete.
- **Context management**: 95% token threshold triggers auto-summarization via a dedicated summarizer model. Summary replaces old history in the DB.
- **Sub-agents**: A "task agent" with read-only tools (Glob, Grep, LS, View) can be spawned for multi-step search. Creates a child session.
- **Event flow**: Pub/sub broker decouples backend from TUI. All events fan into a single `chan tea.Msg`.

**Key tradeoff**: SQLite persistence gives durability and queryability (you can inspect sessions with standard SQL tools), but adds latency to every tool call and makes the architecture heavier for simple use cases.

### Claude Code: Hook-Driven, Performance-Obsessed

```
User prompt → UserPromptSubmit hooks → Assemble system prompt →
Stream API → PreToolUse hooks (can block/modify/allow) →
Execute tool → PostToolUse hooks → Loop → Stop hooks
```

- **Tool execution**: Parallel-capable with background task support. Sub-agents can run concurrently.
- **Context management**: Auto-compaction with loop detection (stops after 3 consecutive refills). Deferred tool loading keeps initial context lean. Aggressive caching (prompt cache hit optimization).
- **Sub-agents**: Rich multi-agent system — named agents, background execution, permission delegation, model overrides. Supports fan-out/fan-in patterns.
- **Hook control plane**: 10+ hook event types with an exit-code protocol (0=allow, 1=user-error, 2=block). Hooks can modify tool inputs, inject system messages, or force retries.

**Key tradeoff**: The hook system provides extraordinary customizability but the core runtime is closed-source. You can shape behavior from the outside but cannot change the engine. Performance optimizations (startup latency, memory, prompt caching) are best-in-class but opaque.

### Pi: Layered, Extension-First

```
User message → transformContext() prunes/injects →
convertToLlm() translates to provider format →
Stream response → Execute tool calls (parallel by default) →
beforeToolCall/afterToolCall hooks → Loop → Steering/follow-up messages
```

- **Tool execution**: Parallel by default. Hooks can block or override individual tool calls.
- **Context management**: JSONL sessions with compaction via LLM summary. Tracks file operation state across compaction boundaries. Sessions support branching and forking.
- **Sub-agents**: Not built-in (the philosophy is "build it as an extension"). The oh-my-pi fork adds sub-agent support.
- **Extension depth**: Extensions can replace the editor, register tools, intercept all tool calls, add custom renderers, register CLI flags, and control session lifecycle.

**Key tradeoff**: Aggressively minimal core (4 default tools: read, write, edit, bash). Maximum extensibility means maximum setup cost for users who want features other tools ship out of the box. The three-layer architecture (pi-ai → pi-agent-core → pi-coding-agent) is the cleanest separation of concerns but adds package management complexity.

---

## 3. Architecture Decision Deep-Dive

### 3.1 Language Choice

| | Opencode (Go) | Claude Code (TypeScript) | Pi (TypeScript) |
|---|---|---|---|
| **Startup time** | Fast (~50ms) | Moderate (~200ms, optimized) | Moderate |
| **Binary distribution** | Single static binary | `npm install` (Node.js required) | `npm install` or compiled Bun binary |
| **Concurrency model** | Goroutines + channels | async/await + Workers | async/await + async iterables |
| **Type safety** | Compile-time (strong) | Compile-time (TypeScript) | Compile-time + runtime (TypeBox) |
| **Extension authoring** | N/A (MCP or .md only) | Markdown + Python hooks | TypeScript extensions |

**Go** (Opencode) gives fast startup and easy cross-platform distribution but limits the extension model to external processes (MCP) or static files. You can't write Go extensions that plug into the runtime.

**TypeScript** (Claude Code, Pi) enables rich in-process extension APIs. Pi takes this furthest — extensions are JIT-compiled TypeScript that runs in the same process with full access to the agent API. Claude Code splits the difference with Markdown-based declarations plus Python hook scripts.

### 3.2 Permission & Safety Model

| | Opencode | Claude Code | Pi |
|---|---|---|---|
| **Sandboxing** | None (blocklist + approval) | Configurable Bash sandbox (network controls, domain allowlists) | None |
| **Permission granularity** | Tool + action + path | Pattern-based rules (`Bash(git:*)`, `Edit(.claude)`) | None (trust-the-user philosophy) |
| **Auto-approve** | Per-session persistent grants | Enterprise-managed policies, auto-mode with classifier | N/A |
| **Enterprise controls** | No | Yes (managed settings, strict marketplaces, allowlists) | No |

Claude Code has the most sophisticated permission model, designed for enterprise deployment. Opencode has a pragmatic middle ground. Pi explicitly rejects permission popups as a design choice — it trusts the user and their environment.

**The tradeoff is fundamental**: more safety = more friction. Claude Code solves this with pattern-based auto-approval and a trained classifier for "auto mode." Opencode uses session-persistent grants (approve once, applies for the session). Pi says: if you don't trust the tool, don't run it.

### 3.3 Context Window Management

| Strategy | Opencode | Claude Code | Pi |
|---|---|---|---|
| **Compaction trigger** | 95% of context window | Approaching token limits | Configurable threshold |
| **Compaction method** | LLM summarization (dedicated model) | Auto-compaction (internal) | LLM summarization |
| **File state tracking** | History service with versions | Deduplicates unchanged re-reads | Tracks reads/edits across compaction |
| **Deferred loading** | No | Yes (ToolSearch for lazy tool schemas) | No |
| **Prompt caching** | No | Yes (aggressive, cache-key stability) | No |

Claude Code's **deferred tool loading** is a standout design: instead of loading all 18+ tool schemas into context upfront, it loads a subset and lets the model discover others via `ToolSearch`. This saves ~thousands of tokens per conversation.

**Prompt caching** is another Claude Code advantage — by keeping tool schemas and system prompts cache-key-stable, it achieves high cache hit rates, reducing latency and cost. Neither Opencode nor Pi implement this.

### 3.4 Extension Architecture

**Opencode**: Two extension mechanisms.
- **MCP servers**: Standards-based but heavyweight (spawn a process, establish protocol, call tools). Each invocation re-establishes the connection.
- **Custom commands**: Markdown files that become prompts. No code execution, no tool registration.

**Claude Code**: Five extension mechanisms layered together.
- **Plugins** (directories with commands, agents, skills, hooks, MCP configs)
- **Hooks** (shell commands or HTTP endpoints triggered on 10+ events)
- **Commands** (Markdown slash commands with frontmatter)
- **Agents** (Markdown sub-agent definitions)
- **Skills** (auto-invoked knowledge documents with path-based activation)

**Pi**: Three extension mechanisms with deep runtime access.
- **Extensions** (TypeScript modules with access to `registerTool()`, `registerCommand()`, `registerShortcut()`, `registerProvider()`, `registerMessageRenderer()`, and 15+ event hooks)
- **Skills** (Markdown with YAML frontmatter, injected into system prompt)
- **Packages** (npm/git bundles of extensions + skills + templates + themes)

**The spectrum**: Opencode is the most restricted (can't change tool behavior). Claude Code provides extensive control via hooks but from outside the process. Pi gives the deepest in-process access but requires TypeScript knowledge.

### 3.5 TUI Architecture

| | Opencode | Claude Code | Pi |
|---|---|---|---|
| **Framework** | Bubbletea (Elm) | Ink (React) | Custom differential renderer |
| **Rendering model** | Full re-render per update | React reconciliation | Line-level diffing |
| **Component model** | Model/Update/View | JSX components | render() → string[] |
| **Overlay system** | Boolean flags per dialog | React component tree | Overlay stack with focus routing |
| **Image support** | No | Yes (via sharp) | Yes (Kitty/iTerm2 protocols) |

Opencode's Bubbletea approach is well-proven but the flat boolean-flag dialog management leads to 900+ line Update/View methods. Claude Code's Ink/React approach is familiar to web developers but carries the React runtime overhead. Pi's custom renderer is the most performant (differential line updates) but the least accessible to contributors.

---

## 4. Comparative Strengths

### Opencode Excels At:
- **Provider breadth**: 11 providers with auto-detection and priority ordering. If you have a GitHub Copilot subscription, it works automatically.
- **Cross-tool interoperability**: Reads `.cursorrules`, `CLAUDE.md`, `.github/copilot-instructions.md` — works with everyone's config.
- **LSP integration**: Real-time diagnostics after edits. Catches type errors, lint issues immediately.
- **Data durability**: SQLite storage means sessions survive crashes, can be queried externally, and support proper undo with file version history.
- **Sourcegraph integration**: Can search public codebases for patterns/examples — unique among the three.

### Claude Code Excels At:
- **Multi-agent orchestration**: Named agents, background execution, fan-out/fan-in, permission delegation. The most sophisticated agent-spawning system.
- **Enterprise readiness**: Managed settings hierarchy, strict marketplace controls, permission allowlists, sandbox configuration.
- **Performance**: Prompt caching, deferred tool loading, startup optimizations, memory management. Every token and millisecond is counted.
- **Hook ecosystem**: 10+ event types with a simple exit-code protocol. Can enforce organizational policies, run formatters, validate commands.
- **Distribution**: `npm install -g @anthropic-ai/claude-code` and it works. Single bundled file with vendored dependencies.

### Pi Excels At:
- **Architectural clarity**: Three cleanly separated layers (LLM API → Agent runtime → Coding agent). Each package has a single responsibility.
- **Extension depth**: In-process TypeScript extensions can replace core components (editor, message renderers, providers).
- **Session model**: JSONL with branching and forking. You can navigate conversation history as a tree, fork from any point.
- **Provider abstraction**: The `pi-ai` package is the cleanest multi-provider LLM abstraction — a standalone library usable outside the coding agent.
- **Composability**: The monorepo packages can be used independently (the Slack bot and web UI demonstrate this).

---

## 5. Opportunities for Differentiated OSS Projects

Based on analyzing the gaps and tradeoffs across all three tools, here are concrete opportunities for new projects that differentiate on dimensions none of them fully exploit:

### 5.1 **Formal Verification & Proof-Carrying Edits**
**Gap**: All three tools apply edits optimistically and rely on post-hoc validation (LSP diagnostics, test runs). None provide formal guarantees about edit correctness.

**Project idea**: A coding agent that uses tree-sitter AST parsing + type-system integration to generate *proof-carrying edits* — each edit comes with a machine-checkable certificate that it preserves type safety, doesn't break imports, and maintains API contracts. Could integrate with tools like Stainless (Scala), LiquidHaskell, or TypeScript's compiler API.

**Differentiator**: Correctness guarantees, not just "it compiles."

### 5.2 **Collaborative Multi-User Agent**
**Gap**: All three are single-user tools. There's no shared agent session where multiple developers can observe, steer, or contribute to the same agent conversation in real-time.

**Project idea**: A coding agent built on CRDTs (like Yjs or Automerge) where multiple developers share a session. Each user can see what the agent is doing, approve/deny tool calls, inject context, or take over editing. Think "Google Docs for AI coding sessions."

**Differentiator**: Team-native AI coding, not bolted-on sharing.

### 5.3 **Repository-Scale Understanding via Code Graphs**
**Gap**: All three tools search code via text patterns (grep/ripgrep). None build or maintain a semantic code graph (call graphs, dependency trees, type hierarchies) that persists across sessions.

**Project idea**: A coding agent that maintains a persistent code graph database (using something like Tree-sitter + LSP + a graph DB like Neo4j or DuckDB). The agent can answer "what calls this function?", "what would break if I change this type?", "show me the data flow from user input to database" without grep-scanning every time.

**Differentiator**: Semantic understanding, not text search.

### 5.4 **Cost-Optimized / Local-First Agent**
**Gap**: Opencode and Pi support local models but don't optimize for them. Claude Code doesn't support local models at all. None implement intelligent model routing (use a cheap/fast model for simple tasks, expensive model for complex ones).

**Project idea**: A coding agent built around intelligent model routing. Simple tasks (file reads, grep, small edits) use a local 7B model or a fast API model. Complex tasks (architecture decisions, multi-file refactors) escalate to frontier models. The router learns from user feedback which tasks need which tier.

**Differentiator**: 10x cost reduction with minimal quality loss.

### 5.5 **Deterministic Replay & Audit Trail**
**Gap**: None of the tools support deterministic replay of agent sessions. You can't take a session transcript and replay it to verify what happened, or use it for compliance auditing.

**Project idea**: A coding agent where every session is a deterministic, replayable log. All inputs (user messages, file states, tool outputs, LLM responses) are captured in a content-addressed store. Sessions can be replayed, diffed, branched, and audited. Critical for regulated industries (finance, healthcare) where AI-generated code changes need audit trails.

**Differentiator**: Compliance and reproducibility.

### 5.6 **Language-Server-Native Agent**
**Gap**: Opencode has basic LSP integration (diagnostics after edits). Claude Code and Pi have none. No tool uses LSP as a *primary* navigation and editing mechanism.

**Project idea**: A coding agent that communicates primarily through LSP. Instead of grep for search, it uses `textDocument/references`, `textDocument/definition`, `workspace/symbol`. Instead of text replacement for edits, it uses LSP code actions, rename refactoring, and extract-method operations. The agent thinks in terms of semantic operations, not text transformations.

**Differentiator**: Refactoring that actually works across the codebase.

### 5.7 **Offline-First with Sync**
**Gap**: All three require constant internet connectivity (even Opencode with local models needs the model server running). None support true offline operation with later sync.

**Project idea**: A coding agent that works fully offline using local models (Ollama/llama.cpp), queues actions that need cloud models, and syncs when connectivity returns. Sessions, context, and model artifacts are stored locally in a git-like content-addressed store. When online, it can optionally "upgrade" local model responses by re-running them through a frontier model.

**Differentiator**: Works on airplanes, in secure environments, and in developing regions.

### 5.8 **Visual-First / Multimodal Agent**
**Gap**: All three are text-primary with bolt-on image support. None can reason about UI screenshots, design mockups, or visual diffs as a core workflow.

**Project idea**: A coding agent built around multimodal understanding. It can screenshot the running app, compare it to a Figma mockup, identify visual discrepancies, and generate CSS/layout fixes. It maintains a visual regression test suite and can catch UI bugs that text-based tools miss entirely.

**Differentiator**: "Make it look like this design" as a first-class workflow.

---

## 6. Summary: How to Think About These Tradeoffs

The three tools represent three distinct philosophies:

| | Philosophy | Optimizes For | Sacrifices |
|---|---|---|---|
| **Opencode** | "Kitchen sink, open to all" | Provider choice, interoperability, data durability | Performance, extension depth |
| **Claude Code** | "Polished product, controlled ecosystem" | Performance, enterprise, multi-agent | Transparency, provider choice |
| **Pi** | "Minimal core, build it yourself" | Extensibility, architectural clarity | Out-of-box features, approachability |

The most impactful new OSS project would likely combine:
- **Pi's architectural clarity** (clean layered packages, provider-agnostic)
- **Claude Code's multi-agent orchestration** (parallel agents, background tasks, permission delegation)
- **Opencode's data durability** (SQLite, file versioning, LSP integration)
- Plus a novel differentiator from section 5 (semantic code graphs, cost-optimized routing, or collaborative sessions)

The space is maturing from "can an AI write code?" to "how do we build reliable, auditable, team-scale AI coding infrastructure?" — and that's where the real differentiation opportunities lie.
