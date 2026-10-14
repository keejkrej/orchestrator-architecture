# 02 — Harness-Internal Subagents & Multi-Agent Machinery

> **Research Scope & Snapshot Date:** October 2026  
> **Focus:** Single-session and intra-harness multi-agent architectures (orchestrator/subagent trees, forked vs. isolated contexts, sibling messaging, fast/slow model routing, compaction, and cross-provider delegation) across 12 leading AI coding harnesses.

---

## Executive Overview

Modern coding-agent harnesses have converged on a fundamental architectural insight: **a single monolithic context window is the wrong abstraction for long-horizon software engineering.** File exploration, web documentation lookups, terminal test loops, and architectural verification generate massive amounts of intermediate noise (grep dumps, failed stack traces, DOM trees) that degrade frontier model attention and bust prompt caches.

To solve this, harnesses embed internal subagent and multi-agent machinery directly into the runtime. Across the 12 harnesses analyzed in this report, four distinct architectural paradigms emerge:

1. **Synchronous & Asynchronous Tool-Call Trees (Depth-Bounded):** The parent agent invokes a tool (`Agent`, `subagent`, `spawn_agent`, `delegate`, `task`) that spins up an isolated child session with its own system prompt, filtered toolset, and model tier, returning only a distilled final report or schema-validated payload to the parent (e.g., **Claude Code**, **OpenCode**, **Gemini CLI**, **Goose**, **Crush**, **Cursor**, **Factory Droids**, **Amp**).
2. **Prefix-Preserving Forked Subagents & Hierarchical Actor Trees:** Instead of starting from a blank slate or passing a lossy handoff summary, the child clones a curated projection of the parent's rollout history—preserving the exact token prefix to get a near-100% prompt-cache hit—and communicates via canonical hierarchical paths (`/root/worker_1`) and asynchronous mailboxes with residency eviction (e.g., **OpenAI Codex CLI V2**, **Claude Code Fork Subagents**).
3. **Sequential Stack / "Boomerang" Delegation:** Only one task is active in the UI at a time; delegating suspends and flushes the parent session to disk, hands the active window to a mode-specialized child, and reloads the parent with the child's completion summary once `attempt_completion` fires (e.g., **Roo Code**).
4. **Contract-Driven Multi-Model Orchestrators & Peer Teams:** A specialized conductor or team lead coordinates named, model-specialized workers (e.g., fast searchers, frontier planners, adversarial plan/gate reviewers, and resident sidecar memory agents) across shared task boards, JSONL ledgers, and peer-to-peer mailboxes (e.g., **oh-my-opencode / OmO**, **Claude Code Agent Teams**, **Cline SDK Teams**).

---

## 1. OpenAI Codex CLI (`openai/codex`)

### 1.1 What It Is / OSS Status / Repo URL
* **What it is:** OpenAI's flagship local coding agent CLI, TUI, and app-server daemon, written in Rust (`codex-rs`).
* **OSS Status:** Open Source (Apache-2.0 License).
* **Repo URL:** `https://github.com/openai/codex` (inspected locally at `repos/codex/codex-rs`).

### 1.2 Topology & Hierarchy
Codex implements **two generations** of multi-agent orchestration inside `repos/codex/codex-rs/core/src/tools/handlers/` and `repos/codex/codex-rs/core/src/agent/`:

* **Multi-Agent V1 (`Feature::Collab` / `multi_agent`, default-enabled):**
  * **Tool Namespace:** `multi_agent_v1` (`core/src/tools/handlers/multi_agents_spec.rs:28`).
  * **Tools:** `spawn_agent`, `send_input` (supports `interrupt: bool`), `resume_agent`, `wait_agent` (waits on multiple agent IDs with timeout `MIN_WAIT_TIMEOUT_MS = 10_000`, `DEFAULT_WAIT_TIMEOUT_MS = 30_000`, `MAX_WAIT_TIMEOUT_MS = 3_600_000` in `core/src/tools/handlers/multi_agents_common.rs`), and `close_agent`.
  * **Depth & Concurrency Limits:** `DEFAULT_AGENT_MAX_DEPTH: i32 = 1` (`core/src/config/mod.rs:265`), enforced via `exceeds_thread_spawn_depth_limit(depth, max_depth)` (`core/src/agent/registry.rs:85`). Root threads start at depth `0`; direct children run at depth `1` and cannot spawn further children by default. Maximum concurrent threads defaults to `DEFAULT_AGENT_MAX_THREADS: Option<usize> = Some(6)` (`core/src/config/mod.rs:255`).
* **Multi-Agent V2 (`Feature::MultiAgentV2` / `multi_agent_v2`):**
  * **Tool Namespace:** `collaboration` (`core/src/tools/handlers/multi_agents_spec.rs:29`).
  * **Hierarchical Path Addressing:** Every agent in the session tree has a canonical filesystem-like `AgentPath` rooted at `/root` (e.g., `/root/task1`, `/root/task1/task_3`), resolved via `resolve_Target_path` in `core/src/tools/handlers/multi_agents_v2/target.rs`.
  * **Tools:** `spawn_agent` (requires `task_name`, `message`, optional `agent_type`, `fork_turns`), `send_message` (queues mail without triggering a turn), `followup_task` (queues mail and triggers a new turn on a non-root agent, optionally with `interrupt: true`), `wait_agent` (parameterless; waits until any background agent finishes or sends mail), `list_agents` (filters by `path_prefix`), and `interrupt_agent`.
  * **Concurrency & Residency Slots:** `DEFAULT_MULTI_AGENT_V2_MAX_CONCURRENT_THREADS_PER_SESSION: usize = 4` (`core/src/config/mod.rs:256`); `effective_agent_max_threads` reserves `max - 1 = 3` child slots so the root thread always keeps a slot.
  * **Persisted Topology Graph:** Parent-child spawn edges are persisted in `repos/codex/codex-rs/agent-graph-store/src/lib.rs` (`LocalAgentGraphStore`, `ThreadSpawnEdgeStatus::{Open, Closed}`).

### 1.3 Information Passing
* **V1 Context Handoff:** `spawn_agent` accepts `fork_context: bool` (defaults to `false`, passing only the initial prompt/`items`).
* **V2 Context Handoff (`fork_turns`):** `spawn_agent` (`core/src/tools/handlers/multi_agents_v2/spawn.rs`) accepts `fork_turns: "none" | "all" | "<positive integer>"`, **defaulting to `"all"`** (`ForkTurns::All`).
* **Curated Rollout Forking (`prepare_forked_history`):** Defined in `core/src/agent/control/spawn.rs:295-372`. When forking, Codex flushes the parent rollout (`ensure_rollout_materialized`), strips parent role/collaboration developer instructions (`retain_forked_developer_message`), and filters rollout items via `keep_forked_rollout_item` (`spawn.rs:226-267`):
  * **Retained:** `UserMessage`, `SystemMessage`, `DeveloperMessage`, `PartialAnswer`, and `FinalAnswer` assistant messages, plus `<turn_aborted>` markers.
  * **Stripped:** Intermediate `Reasoning` traces, `FunctionCall` / `LocalShellCall` / `CustomToolCall` invocations and outputs, `WebSearchCall`, `InterAgentCommunication`, and `ContextCompaction` events. This keeps the shared prefix lean while preserving conversational continuity.
* **Structured Inter-Agent Envelope (`InterAgentCommunication`):** Mailboxes (`core/src/agent/control/mailbox.rs`) exchange `InterAgentCommunication` structs (`author: AgentPath`, `recipient: AgentPath`, `other_recipients: Vec<AgentPath>`, `content: String`, `trigger_turn: bool`).
* **Terminal Turn Completion:** When a child turn finishes, `notify_parent_of_terminal_turn` (`core/src/agent/control/completion.rs:94-169`) wraps the child's final text (or error truncated to `ERROR_MAX_TOKENS = 1000`) via `format_inter_agent_completion_message` (`core/src/session_prefix.rs`) and delivers it to the parent with `trigger_turn: false`.
* **Shared Agent Message Board (`Feature::AgentMessageBoard`):** Implemented in `repos/codex/codex-rs/ext/agent-message-board/src/lib.rs`, exposing `create_channel`, `list_channels`, `create_thread`, `post`, `read_thread`, `subscribe`, and `unsubscribe` for pub/sub coordination across local or remote boards (`agent-message-board-client`).

### 1.4 Agent Lifespan & Context Window Management
* **Stateful Multi-Turn Lifespan:** Spawned agents stay alive across turns (`AgentStatus::PendingInit | Running | Completed(Option<String>) | Errored(String) | Shutdown`).
* **V2 Residency Eviction (`core/src/agent/control/residency.rs`):** When a child becomes idle and a new child needs a residency slot, `evict_idle_for_spawn` (`residency.rs:51-99`) picks the oldest idle leaf session that has no live children, calls `unload_local_session_preserving_mail`, drains any unread messages from the session queue back into `Mailboxes`, and unloads the in-memory `CodexThread`. When a `followup_task` later targets that agent, `get_or_reload_local_session` (`residency.rs:146-214`) transparently reloads its rollout from disk and replays queued mail.
* **Compaction:** Both parent and child threads run automatic and manual context compaction via `core/src/compact.rs` (`run_inline_auto_compact_task`, `SUMMARIZATION_PROMPT` in `prompts/templates/compact/prompt.md`), replacing older rollout items with a summary prefixed by `prompts/templates/compact/summary_prefix.md`.

### 1.5 Fast vs. Slow / Model Mixing & Cross-Provider Support
* **Built-In Roles (`core/src/agent/role.rs` & `core/assets/agent/builtins/*.toml`):**
  * `default`: General-purpose fallback role.
  * `explorer` (`explorer.toml`): Read-only codebase exploration; prompted to prefer fast search (`rg --files`, `rg`) and avoid spawning further subagents.
  * `worker`: Execution-focused implementation agent.
  * `awaiter` (`awaiter.toml`): Specialized role for long-running external monitors (builds, deploys, CI, cloud training), configured with `model_reasoning_effort = "low"` and `background_terminal_max_timeout = 3600000` (1 hour).
* **Custom Roles:** Loaded from `config.agents` (`AgentRoleConfig`) or `.codex/agents/*.toml`, each specifying `model`, `model_reasoning_effort`, `model_reasoning_summary`, `sandbox_mode`, `developer_instructions`, and `nickname_candidates`.
* **Cross-Provider Support:** Supports OpenAI Responses API, Azure, Ollama (`codex-rs/ollama`), LM Studio (`codex-rs/lmstudio`), and custom OpenAI-compatible model providers (`codex-rs/model-provider`).

### 1.6 Human-in-the-Loop & Quality Gates
* **Approval & Sandbox Inheritance:** `build_agent_shared_config` (`core/src/agent/control/spawn.rs:190-224`) propagates the parent's `AskForApproval` policy, `SandboxPolicy`, `ExecPolicy`, and writable roots to children unless restricted by the child's role (`sandbox_mode = "read-only"`).
* **Guardian Subagent Review (`codex-rs/guardian-context`):** Codex includes a dedicated Guardian subagent review context (`codex-rs/core/src/guardian/`) that evaluates high-risk actions against security policy before execution.

### 1.7 Persistence, Memory, Learning Across Sessions
* **Rollout Persistence & Thread Store:** Every thread (parent and child) is recorded as a JSONL rollout in `~/.codex/sessions/` and indexed in SQLite (`codex-rs/state`, `codex-rs/thread-store`).
* **Memories Subsystem (`codex-rs/memories`):** Two-phase background memory pipeline (`core/src/memories/`): Phase 1 extracts per-rollout summaries from completed sessions, and Phase 2 consolidates durable cross-session memories into `MEMORY.md` and `memory_summary.md` under `~/.codex/memories/`.

### 1.8 Notable Design Insights
* **Prefix-Filtered Forking (`keep_forked_rollout_item`) vs. Full Transcript Cloning:** Cloning every historical tool output into a forked subagent wastes context tokens; stripping intermediate tool calls while keeping user/developer prompts and final assistant answers preserves narrative coherence at a fraction of the token cost.
* **Residency Eviction with Durable Mailboxes:** Decoupling an agent's logical existence (`AgentPath` + `Mailboxes` + disk rollout) from its in-memory residency (`ResidencySlot`) allows a session tree to contain dozens of logical subagents while strictly bounding active RAM and concurrent LLM turns to 3–4 slots.

---

## 2. OpenCode (`anomalyco/opencode` / `sst/opencode`)

### 2.1 What It Is / OSS Status / Repo URL
* **What it is:** A terminal-native and client-server coding agent harness built with TypeScript, Bun, and Effect (`effect`), originally created by SST / Anomaly.
* **OSS Status:** Open Source (MIT License).
* **Repo URL:** `https://github.com/anomalyco/opencode` (also `https://github.com/sst/opencode`; inspected locally at `repos/opencode`).

### 2.2 Topology & Hierarchy
* **Subagent Tool:** `packages/core/src/tool/plugin/subagent.ts` (`name: "subagent"`).
* **Agent Modes (`packages/core/src/plugin/agent.ts`):** Agents have `mode: "primary" | "subagent" | "all"`. Primary agents (`Build`, `Plan`, `Compaction`) cannot be invoked by the `subagent` tool (`subagent.ts:132-136`). Built-in subagents (`mode: "subagent"`) are:
  * `General`: Full tool access (except `question` and `subagent` are denied).
  * `Explore`: Fast read-only codebase explorer (`PROMPT_EXPLORE` in `packages/core/src/plugin/explore.txt`; allowed only `shell`, `grep`, `glob`, `webfetch`, `websearch`, `read`; all other tools and `subagent` denied).
* **Depth Limit:** Default max depth is **1** (`limit = (await Config.latest(context, "experimental"))?.subagent_depth ?? 1`). Before spawning, `subagent.ts:110-124` walks `current.parentID` via `sessions.get` to compute `depth` and throws `"Maximum subagent depth (${limit}) reached"` if `depth >= limit`. Furthermore, built-in subagents explicitly set `subagent: "deny"` in their permission rulesets (`agent.ts:116, 134`).

### 2.3 Information Passing
* **Prompt-Only Fresh Context:** When spawning a new child session (`sessions.create({ parentID: context.sessionID, title: input.description, agent: agent.name, model })`), the parent's conversation history is **not** copied. Instead, the child receives only its agent system prompt plus:
  ```text
  You are a subagent spawned by another session.
  <input.prompt>
  ```
* **Foreground Return Payload:** `formatSubagentResult` (`subagent.ts:48-61`) wraps the child's final assistant message in an XML envelope:
  ```xml
  <subagent sessionID="ses_..." state="completed">
  ...child final response...
  </subagent>
  ```
* **Background Delivery via Durable SQLite Inbox:** When `background: true` is passed, `SubagentJob` (`packages/core/src/session/subagent-job.ts`) runs the child fiber asynchronously. Upon completion, `SubagentCompletion.deliver` (`packages/core/src/session/subagent-completion.ts:71-141`) checks if the parent session is currently busy:
  * If busy, it inserts a row into `SessionInboxTable` (`packages/core/src/session/inbox.ts`, backed by Drizzle SQLite `session_inbox`), which is drained between turns and injected as a synthetic user turn (`sessions.synthetic`).
  * If idle, it immediately calls `sessions.synthetic` with `<subagent sessionID="..." state="completed|failed|interrupted" description="...">...</subagent>` and triggers a new parent turn (`jobs.prompt(parentID)`).

### 2.4 Agent Lifespan & Context Window Management
* **Resumable Sessions:** Every subagent call creates or reuses a full persistent `Session` in SQLite (`parentID = context.sessionID`) and returns its `sessionID` in the `<subagent sessionID="...">` tag. Passing `sessionID` on a subsequent `subagent` call resumes that exact child session (`subagent.ts:154-175`), even allowing the parent to switch the child's `agent` or `model` mid-session.
* **Parent-Cascaded Cancellation:** For foreground subagents, `Fiber.onInterrupt` (`subagent.ts:207-209`) automatically calls `jobs.cancel(sessionID)` on the child if the parent turn is interrupted.
* **Compaction (`packages/core/src/session/compaction.ts`):** When a session overflows, OpenCode runs the hidden `Compaction` agent while **retaining the most recent `15,000` tokens verbatim** (`keep: 15_000`, `compaction.ts:32`), prepending a structured `<conversation-checkpoint><summary>...</summary><recent-context>...</recent-context></conversation-checkpoint>` message (`compaction.ts:40-48`).

### 2.5 Fast vs. Slow / Model Mixing & Cross-Provider Support
* **Per-Call & Per-Agent Model Overrides:** Each configured agent in `opencode.json` can specify `model: "providerID/modelID#variant"`, `temperature`, and `topP` (`agent.ts:27-35`). Additionally, the `subagent` tool accepts a dynamic `model` parameter on every call (`subagent.ts:73-77`), parsed via `Provider.parseModel`.
* **Cross-Provider Support:** Built on Vercel AI SDK (`ai`) + OpenCode's multi-provider registry, allowing a single session to mix Anthropic, OpenAI, Google Gemini, Bedrock, Vertex, OpenRouter, and local models across parent and child agents.

### 2.6 Human-in-the-Loop & Quality Gates
* **Permission Rulesets (`packages/core/src/plugin/agent.ts:81-141`):** Every agent has a fine-grained permission table (`allow | ask | deny` with glob patterns). Subagents default to `question: "deny"` so they cannot block on interactive multiple-choice user modals unless explicitly enabled in user config.
* **Plan Mode Gate (`packages/core/src/plugin/plan.ts`):** The primary `Plan` agent denies all file edits except to its designated plan file (`planPath(session)`).

### 2.7 Persistence, Memory, Learning Across Sessions
* **SQLite Session & Inbox Persistence:** All parent and child sessions, messages, parts (`SubagentPart` with `{ sessionID, agent, model, description, prompt, background }`), and pending background completions (`SessionInboxTable`) are persisted in SQLite via Drizzle ORM (`packages/core/src/session/session.sql.ts`). If the process crashes while a background subagent completion is waiting to be delivered, `SessionInbox` guarantees at-least-once delivery on restart.

### 2.8 Notable Design Insights
* **Crash-Safe Asynchronous Completion Inbox (`SessionInbox`):** Most harnesses lose background subagent notifications if the parent is mid-turn or the CLI restarts. OpenCode's transactional `SessionInbox` + `SubagentCompletion` fiber guarantees that background subagents wake an idle parent immediately or queue cleanly for the next turn boundary without race conditions.
* **Hybrid Verbatim + Checkpoint Compaction:** Retaining the last 15,000 tokens of raw tool/assistant messages after a `<conversation-checkpoint>` summary avoids the "post-compaction amnesia" where an agent immediately re-reads files it just edited on the previous turn.

---

## 3. Claude Code (`anthropics/claude-code` & `claude-agent-sdk-python`)

### 3.1 What It Is / OSS Status / Repo URL
* **What it is:** Anthropic's agentic coding CLI, IDE extension, and programmable Agent SDK.
* **OSS Status:** Proprietary CLI runtime (distributed via npm/native installer), with open-source TypeScript/Python SDKs and plugin ecosystems.
* **Repo URLs:** `https://github.com/anthropics/claude-code` and `https://github.com/anthropics/claude-agent-sdk-python` (inspected locally at `repos/claude-code/mods/types/claude-code.d.ts`, `repos/claude-agent-sdk-python/src/claude_agent_sdk/types.py`, and official documentation snapshots).

### 3.2 Topology & Hierarchy
Claude Code supports **two distinct multi-agent topologies**:

1. **Hierarchical Subagents (`Agent` tool, renamed from `Task` in v2.1.63):**
   * **Nesting Depth Limit:** By default, subagents can nest up to **3 layers deep** below the main conversation (`CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`, default `3` since v2.1.219; setting `1` restores the legacy one-layer cap). At the configured depth limit, the `Agent` tool is automatically withheld from the child's toolset.
   * **Concurrency Limit:** Up to **20 concurrent subagents** (`CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS=20`).
   * **Allowlist Restriction:** A parent running as a custom agent (`claude --agent`) can restrict which subagents it is allowed to spawn using `Agent(worker, researcher)` in its `tools` frontmatter.
2. **Experimental Agent Teams (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`):**
   * **Peer Mesh + Lead Topology:** Consists of a **Team Lead** (main session), independent **Teammates** (separate Claude Code sessions running either `in-process` or in `tmux`/`iTerm2` split panes), a **Shared Task List** (`~/.claude/tasks/{team-name}/`), and **Peer-to-Peer Mailboxes** (`~/.claude/teams/{team-name}/inboxes/{agent-name}.json`).
   * **No Nested Teams:** Teammates cannot spawn their own teams or sub-teammates; only the lead manages team lifecycle (`TeamCreate`, `TeamDelete`).

### 3.3 Information Passing
* **Standard Subagents (Isolated Context):** Receive a fresh context window containing their own system prompt (markdown body), the `Agent` tool prompt, `CLAUDE.md` project instructions (unless `omitClaudeMd: true`), git status snapshot, preloaded `skills` (up to 32), and a **one-line sibling roster** listing other active subagents' names and task descriptions (`SendMessage` addresses).
* **Fork Subagents (`CLAUDE_CODE_FORK_SUBAGENT`, default ON in interactive sessions since v2.1.232; also `/subtask` or `subagent_type: "fork"`):**
  * Instead of starting fresh, a fork subagent **inherits the parent's full conversation history, system prompt, and tool definitions**, allowing it to reuse the parent's Anthropic API prompt cache prefix while executing an isolated side-investigation or multi-file edit.
* **Prompt-Injection Output Sanitization (v2.1.210+):** Before the parent reads a subagent's final report, the harness scans the subagent output for instruction-shaped patterns (e.g., `<system-reminder>`, `Human:`/`Assistant:` turn markers, prompt-injection phrases). If matched, the harness escapes the tags with backslashes, prepends `[harness: subagent output matched instruction-shaped pattern(s): ...]`, and wraps the report under a header stating that subagent claims carry no user authority.
* **Agent Teams Shared Task List & Mailboxes:**
  * Tasks have states `pending`, `in_progress`, and `completed`, with explicit dependency edges (completing a blocking task automatically unblocks dependents) and **file-system locking** (`~/.claude/tasks/{team-name}/`) to prevent simultaneous task claiming races.
  * Teammates communicate directly with each other via `SendMessage(to: "<teammate-name>")` or broadcast (`to: "*"`) without routing through the lead.

### 3.4 Agent Lifespan & Context Window Management
* **Resumable Named Subagents:** Subagents run in foreground (blocking) or background (`background: true` or `Ctrl+B`). Completed subagents (except one-shot `Explore` and `Plan`) return an `agentId` and remain resumable via `SendMessage(to: agentId_or_name)`, preserving their full conversation history across multiple follow-up turns.
* **Per-Subagent Compaction Threshold (`autoCompactWindow`):** Custom subagents can configure `autoCompactWindow` (clamped between `100,000` and `1,000,000` tokens, or set via `CLAUDE_CODE_AUTO_COMPACT_WINDOW`) so research subagents compact earlier and return crisp summaries.
* **Transcripts & Retention:** Subagent transcripts are saved at `~/.claude/projects/{project}/{sessionId}/subagents/agent-{agentId}.jsonl` alongside `.meta.json` and survive main-session compaction (cleaned up after `cleanupPeriodDays`, default 30 days).

### 3.5 Fast vs. Slow / Model Mixing & Cross-Provider Support
* **Built-In Subagent Model Routing:**
  * `Explore`: Read-only codebase discovery (`omitClaudeMd: true`, omits git status snapshot to save input tokens; supports `quick`, `medium`, `very thorough` levels; configurable via `CLAUDE_CODE_SUBAGENT_MODEL=haiku`, defaults to main/Opus in interactive sessions).
  * `Plan`: Read-only research agent invoked during Plan Mode (`omitClaudeMd: true`).
  * `general-purpose`: Inherits main model and all tools for complex multi-step research + edits.
  * `statusline-setup` (`sonnet`) and `claude-code-guide` (`haiku`).
* **Custom Subagent Frontmatter (`AgentDefinition` in `repos/claude-agent-sdk-python/src/claude_agent_sdk/types.py:217-227`):** Supports `model: "sonnet" | "opus" | "haiku" | "inherit"` (or full model ID), `effort: "low" | "medium" | "high" | "xhigh" | "max"`, and `experimental.cacheTtl: "5m" | "1h"`.
* **Cross-Provider Support:** Primarily Anthropic Claude (1P API, AWS Bedrock, Google Vertex AI, Azure Foundry), but routinely spawns external model CLIs (such as OpenAI Codex or Gemini) via **MCP servers** (`mcpServers` in subagent frontmatter) or Bash skills.

### 3.6 Human-in-the-Loop & Quality Gates
* **Permission Modes (`permissionMode`):** Subagents inherit parent permissions or override with `default`, `acceptEdits`, `auto`, `dontAsk`, `plan`, or `bypassPermissions`. (If the parent uses `bypassPermissions` or `auto`, it takes precedence and cannot be overridden by a child).
* **Lifecycle & Quality Gate Hooks:**
  * `SubagentStart` and `SubagentStop` hooks run on subagent lifecycle transitions.
  * For Agent Teams, `TeammateIdle`, `TaskCreated`, and `TaskCompleted` hooks act as deterministic quality gates: if a hook script exits with **code `2`**, the action is blocked (e.g., preventing task completion when tests fail) and stderr is fed back to the teammate to continue working.
* **Worktree Isolation (`isolation: worktree`):** Runs the subagent inside a temporary `git worktree` that is automatically cleaned up if the subagent makes no changes.

### 3.7 Persistence, Memory, Learning Across Sessions
* **Scoped Agent Memory (`memory: user | project | local`):** Gives a subagent a dedicated persistent directory (`~/.claude/agent-memory/<name>/`, `.claude/agent-memory/<name>/`, or `.claude/agent-memory-local/<name>/`), automatically injects the first **200 lines / 25 KB** of its `MEMORY.md` into the subagent's system prompt, and enables `Read`/`Write`/`Edit` tools so the subagent curates its own domain knowledge across sessions.

### 3.8 Notable Design Insights
* **Security Boundary Between Subagent and Parent:** Treating subagent outputs as untrusted data (scanning and escaping `<system-reminder>` and role-turn markers before parent ingestion) mitigates indirect prompt injection when a subagent reads malicious web pages or third-party repo files.
* **Token-Stripped Fast Explorers (`omitClaudeMd`):** Stripping `CLAUDE.md` and git status snapshots from read-only `Explore` and `Plan` subagents saves thousands of fixed prompt tokens on every quick file-search call.

---

## 4. Gemini CLI (`google-gemini/gemini-cli`)

### 4.1 What It Is / OSS Status / Repo URL
* **What it is:** Google's open-source terminal coding agent for Gemini models, written in TypeScript/Node.js.
* **OSS Status:** Open Source (Apache-2.0 License).
* **Repo URL:** `https://github.com/google-gemini/gemini-cli` (inspected locally at `repos/gemini-cli`).

### 4.2 Topology & Hierarchy
* **Subagents-as-Tools Architecture (`packages/core/src/agents/`):** Registered subagents are wrapped into standard `DeclarativeTool` instances via `SubagentToolWrapper` / `AgentTool` (`packages/core/src/agents/agent-tool.ts`). The main agent invokes a subagent exactly like any other tool call.
* **Local vs. Remote (A2A) Subagents (`packages/core/src/agents/types.ts:192-218`):**
  * `LocalAgentDefinition` (`kind: 'local'`): Executed in-process by `LocalAgentExecutor` (`packages/core/src/agents/local-executor.ts`).
  * `RemoteAgentDefinition` (`kind: 'remote'`): Executed over HTTP/streaming via the **Agent2Agent (A2A) protocol** (`@a2a-js/sdk`) using `agent_card_url` in `RemoteAgentInvocation` (`packages/core/src/agents/remote-invocation.ts`).
* **Strict Depth Limit = 1 (`local-executor.ts:204, 392-396`):** When `LocalAgentExecutor.create` builds a subagent's isolated `ToolRegistry`, it explicitly filters out all tools with `tool.kind === Kind.Agent` (`if (tool.kind === Kind.Agent) return;`). Subagents are structurally prevented from spawning other subagents.

### 4.3 Information Passing
* **Isolated Context (`GeminiChat`):** Each local subagent starts with a fresh `GeminiChat` instance (`local-executor.ts:154-174`) seeded only with its compiled system prompt (`buildChatSystemPrompt`), tool declarations, and the Zod-validated `AgentInputs` parameters passed by the parent.
* **Mandatory `complete_task` Termination Protocol (`local-executor.ts:261-319`):**
  * Every local subagent is injected with a synthetic tool named `complete_task` (`COMPLETE_TASK_TOOL_NAME = 'complete_task'`).
  * If the subagent defines an `outputConfig` (`OutputConfig<T extends z.ZodTypeAny>`), the `complete_task` tool's input parameter schema is dynamically generated from that Zod schema (`z.toJSONSchema(outputConfig.schema)`) and validated on invocation.
  * **Protocol Enforcement & Grace Turn (`local-executor.ts:373-459`):** A subagent **cannot** finish simply by emitting text and stopping tool calls. If a subagent stops calling tools without invoking `complete_task`, the executor marks `ERROR_NO_COMPLETE_TASK_CALL` and grants a **1-turn grace period** (`GARACE_PERIOD_MS = 60_000`) with a recovery prompt (`getTimeoutPrompt`) demanding an immediate `complete_task` call.
* **Structured Return Value:** `CodebaseInvestigatorAgent` (`packages/core/src/agents/codebase-investigator.ts:32-55`) enforces `CodebaseInvestigationReportSchema`:
  ```typescript
  {
    SummaryOfFindings: string,
    ExplorationTrace: string[],
    RelevantLocations: Array<{ FilePath: string, Reasoning: string, KeySymbols: string[] }>
  }
  ```

### 4.4 Agent Lifespan & Context Window Management
* **Bounded Ephemeral Local Runs & Stateful Remote A2A Sessions:**
  * Local subagents are ephemeral per tool call, bounded by `RunConfig` (`maxTimeMinutes` default `10`, `maxTurns` default `50` in `codebase-investigator.ts:89-93`).
  * Remote A2A subagents (`remote-invocation.ts`) persist `contextId` and `taskId` state via `A2AClientManager` across invocations within the session.
* **Subagent Context Compression:** Every `LocalAgentExecutor` instantiates its own `ChatCompressionService` (`local-executor.ts:947`), allowing long-running subagents to compress their internal history mid-task without failing on context limits.

### 4.5 Fast vs. Slow / Model Mixing & Cross-Provider Support
* **Built-In Specialized Subagents (`packages/core/src/agents/`):**
  * `codebase_investigator` (`codebase-investigator.ts`): Uses `PREVIEW_GEMINI_FLASH_MODEL` (`gemini-3-flash-preview`) with `ThinkingLevel.HIGH` and `temperature: 0.1` (`codebase-investigator.ts:80-87`) plus read-only tools (`ls`, `read_file`, `glob`, `grep`).
  * `generalist` (`generalist-agent.ts`): Uses `model: 'inherit'` and inherits all parent tools (`allToolNames`) for high-volume batch implementation.
  * `cli_help` (`cli-help-agent.ts`): Documentation specialist using `get_internal_docs`.
  * `browser_agent` (`browser/browser-agent-definition.ts`): Launches a dedicated `BrowserManager` + `chrome-devtools-mcp` client and optionally pairs a `visualModel` for computer-use screenshot analysis.
  * `skill_extraction_agent` (`skill-extraction-agent.ts`): Background memory/skill distillation agent.
* **Custom Subagents (`.gemini/agents/*.md` and `~/.gemini/agents/*.md`):** Loaded by `AgentLoader` (`packages/core/src/agents/agent-loader.ts`), supporting YAML frontmatter with `kind: local | remote`, `model`, `temperature`, `max_turns`, `timeout_mins`, `tools`, and `mcp_servers`.

### 4.6 Human-in-the-Loop & Quality Gates
* **Parent MessageBus Confirmation Forwarding (`agent-tool.ts:122-145`):** Local subagents share a child `MessageBus` connected to the parent's policy engine. If a subagent attempts a tool call requiring user confirmation (e.g., shell execution in a custom agent), confirmation requests bubble up to the main CLI UI while streaming real-time subagent activity (`SubagentActivityEvent`: `TOOL_CALL_START`, `TOOL_CALL_END`, `THOUGHT_CHUNK`, `ERROR`).

### 4.7 Persistence, Memory, Learning Across Sessions
* **Automatic Skill Extraction (`packages/core/src/agents/skill-extraction-agent.ts`):** Gemini CLI includes a background `SkillExtractionAgent` that inspects completed session transcripts (`session_transcript`) alongside existing skills, extracts reusable non-obvious multi-step workflows into `${memoryDir}/.inbox/<skill-name>/SKILL.md`, and is restricted by filesystem write guards to only modify `${memoryDir}/.inbox` (`skill-extraction-agent.ts:144-169`).

### 4.8 Notable Design Insights
* **Schema-Validated `complete_task` Contract with Grace Turn:** Instead of hoping a subagent formats its final markdown response properly, Gemini CLI turns task completion into a schema-validated tool call (`complete_task`) and uses a 60-second recovery turn if the subagent forgets to call it or hits `MAX_TURNS` / `TIMEOUT`.
* **First-Class A2A Protocol Integration:** Treating remote A2A agents (`agent_card_url`) as peer `AgentDefinition` entries alongside local markdown agents enables seamless delegation to external enterprise microservices.

---

## 5. Goose (`block/goose`)

### 5.1 What It Is / OSS Status / Repo URL
* **What it is:** Block's extensible open-source AI agent CLI and desktop engine, written in Rust with first-class Model Context Protocol (MCP) architecture.
* **OSS Status:** Open Source (Apache-2.0 License).
* **Repo URL:** `https://github.com/block/goose` (inspected locally at `repos/goose`).

### 5.2 Topology & Hierarchy
* **Unified `summon` Platform Extension (`crates/goose/src/agents/platform_extensions/summon.rs`):** In 2026, Goose consolidated its legacy `subagent`, `skills`, `subrecipes`, and `dynamic_task` tools into a single built-in platform extension called **`summon`** (`EXTENSION_NAME = "summon"`), exposing two tools:
  * `load`: Injects knowledge, skills, recipes, or agent definitions directly into the *current* session's context (or discovers available sources when called with no arguments).
  * `delegate`: Spawns an isolated subagent session (`SessionType::SubAgent`) to execute a task and return a summary.
* **Strict Depth Limit = 1 (`summon.rs:851-853`):**
  ```rust
  if session.session_type == SessionType::SubAgent {
      return Err("Delegated tasks cannot spawn further delegations".to_string());
  }
  ```
  Subagents are explicitly blocked from calling `delegate` recursively.

### 5.3 Information Passing
* **Three Delegation Modes (`summon.rs:89-99`):**
  1. *Ad-hoc:* `delegate(instructions: "...")`
  2. *Source-based:* `delegate(source: "code-reviewer", parameters: {...})` (loads a Recipe YAML, Skill, or `.claude/agents/*.md` / `.goose/agents/*.md` file)
  3. *Combined:* `delegate(source: "rust-expert", instructions: "review auth.rs", context: "...")`
* **Explicit `context` Injection (`build_instructions_with_context`, `summon.rs:943-956`):** The parent does **not** pass its full transcript. Instead, `delegate` provides a dedicated `context: Option<String>` parameter ("Supporting material from the conversation that the delegate needs — e.g. code snippets, file contents, error output, or prior decisions. Unlike instructions (what to do), context is background data the delegate cannot discover on its own"), which is formatted under a `## Context` section in the child's prompt.
* **Structured Final Output Extraction (`summon.rs:897-904, 151-162`):** Every ad-hoc subagent recipe attaches a JSON `Response` schema requiring `{"summary": {"type": "string"}}`. When the child finishes, `readable_output` parses the JSON output and returns only the `summary` string to the parent tool call.

### 5.4 Agent Lifespan & Context Window Management
* **Synchronous Foreground Execution (`ForegroundSubagentRunner`):** Implemented in `crates/goose/src/agents/state_machine/ops_subagent.rs`. Notably, Goose **removed** its earlier `async: true` background task pool in favor of deterministic foreground execution (`DELEGATED_META_KEY = "foreground_subagent"`), though the LLM can still invoke multiple `delegate` tool calls in parallel within a single turn (`run_subagent_batch` in `ops_subagent.rs:344-421`).
* **Turn Limits:** Bounded by `DEFAULT_SUBAGENT_MAX_TURNS: usize = 25` (`summon.rs:41`), overridable via `DelegateParams.max_turns`, recipe `settings.max_turns`, or `GOOSE_SUBAGENT_MAX_TURNS`.

### 5.5 Fast vs. Slow / Model Mixing & Cross-Provider Support
* **Per-Delegation Provider & Model Overrides (`DelegateParams` in `summon.rs:60-87`):** Every `delegate` call can specify `provider: Option<String>`, `model: Option<String>`, `temperature: Option<f32>`, `extensions: Option<Vec<String>>` (passing `[]` disables all extensions for fast pure-LLM tasks), and `working_dir: Option<PathBuf>`.
* **Cross-Ecosystem Agent Discovery (`summon.rs:369-410`):** Goose's `discover_local_sources` scans not only `.goose/recipes/` and `.goose/skills/`, but also **`.agents/agents` and `.claude/agents/*.md`**, parsing Claude Code agent frontmatter directly into Goose `SourceKind::Agent` definitions.

### 5.6 Human-in-the-Loop & Quality Gates
* **Extension Downscoping:** By passing `extensions: ["developer"]` or `extensions: []` in `delegate`, the parent restricts the subagent's MCP surface area. Cancellation tokens (`CancellationToken`) propagate from parent to child immediately upon user interrupt.

### 5.7 Persistence, Memory, Learning Across Sessions
* **Session Storage (`SessionManager`):** Every delegated run creates a persistent `SessionType::SubAgent` record in Goose's SQLite session store (`context.session_manager.create_session`), linked to the working directory and inspectable in session logs.

### 5.8 Notable Design Insights
* **The `load` vs. `delegate` Dichotomy (`summon`):** Treating *in-context skill loading* (`load`) and *isolated-context subagent execution* (`delegate`) as two modes of the same `summon` extension over the same unified catalog of Recipes, Skills, and Agents eliminates redundant configuration formats.
* **Separating `instructions` from `context` in the Tool Schema:** Explicitly splitting `instructions` ("what to do") and `context` ("background data from parent conversation") in `DelegateParams` prompts the parent LLM to include necessary snippets rather than issuing vague one-line delegations.

---

## 6. Roo Code / Kilo Code / Cline (`Roo-Code`, `kilocode`, `cline`)

### 6.1 What They Are / OSS Status / Repo URLs
This family represents the evolution of open-source VS Code / CLI multi-agent harnesses originating from Cline:
* **Roo Code (`RooCodeInc/Roo-Code`):** Open Source (Apache-2.0), `https://github.com/RooCodeInc/Roo-Code` (`repos/Roo-Code`). Pioneered the **Orchestrator / "Boomerang Task"** architecture in VS Code.
* **Kilo Code (`Kilo-Org/kilocode`):** Open Source (Apache-2.0), `https://github.com/Kilo-Org/kilocode` (`repos/kilocode`). Originally forked from Roo Code (retaining the VS Code Agent Manager UI), its 2026 core CLI/runtime (`packages/opencode/`) is built on an Effect-TS engine forked from OpenCode with custom `Scout` and `TaskTool` subagents.
* **Cline (`cline/cline`):** Open Source (Apache-2.0), `https://github.com/cline/cline` (`repos/cline`). In its 2026 SDK (`sdk/packages/core/src/extensions/tools/team/`), Cline implements both parallel `spawn_agent` delegation and a full stateful `AgentTeamsRuntime`.

### 6.2 Topology & Hierarchy
* **Roo Code (Sequential Stack / "Boomerang" Tree):**
  * Uses the `new_task` tool (`repos/Roo-Code/src/core/tools/NewTaskTool.ts`).
  * Any mode (typically `orchestrator` mode) can spawn a child task in a target mode (`code`, `architect`, `ask`, `debug`, or custom modes).
  * **Single Active Task Invariant (`ClineProvider.delegateParentAndOpenChild` in `src/core/webview/ClineProvider.ts:2783-2911`):** Roo Code maintains a task stack (`clineStack`) where **only one task runs at a time**. Spawning a child suspends and closes the parent task in memory (`removeClineFromStack`), updates `HistoryItem.status = "delegated"`, and opens the child task in the webview. Arbitrary nesting depth is supported (`parentTaskId`, `rootTaskId`).
* **Kilo Code (Bounded Session Tree):**
  * Uses `TaskTool` (`repos/kilocode/packages/opencode/src/tool/task.ts`).
  * Subagents (`explore`, `general`, `scout`) have `task: "deny"` in their permission ruleset (`src/agent/agent.ts:103, 128, 161` and `src/agent/subagent-permissions.ts`), enforcing **depth = 1** unless a custom subagent explicitly sets `permission.task: "allow"`.
* **Cline SDK (Parallel Subagents + Peer Team Runtime):**
  * **Tier 1 (`spawn_agent` & `subagent_<name>`):** `spawn-agent-tool.ts` (`executionMode: "parallel"`, default timeout 5 minutes) and `configured-agent-tool.ts` run isolated `createDelegatedAgent` instances (`delegated-agent.ts`) that cannot recursively spawn unless explicitly handed the `spawn_agent` tool.
  * **Tier 2 (`AgentTeamsRuntime` in `repos/cline/sdk/packages/core/src/extensions/tools/team/`):** Full team orchestration (`team-tools.ts`, `multi-agent.ts`, `team-child-session-manager.ts`) with persistent teammates (`team_add_member`, `team_run_task`, `team_await_runs`), mailboxes (`team_send_message`, `team_broadcast`, `team_read_mailbox`), and a structured **Outcome Assembly pipeline** (`team_create_outcome`, `team_attach_outcome_fragment`, `team_review_outcome_fragment`, `team_finalize_outcome`).

### 6.3 Information Passing
* **Roo Code Boomerang Handoff:**
  * **Downward:** `new_task` passes `mode`, `message` (note: escaped `\\@` file mentions are unescaped one level per hop in `NewTaskTool.ts:74`: `message.replace(/\\\\@/g, "\\@")`), and an initial `todos` markdown checklist (`parseMarkdownChecklist`).
  * **Upward (`reopenParentFromDelegation` in `ClineProvider.ts:2918-3029`):** When the child calls `attempt_completion(result)`, `ClineProvider` reads the suspended parent's `apiConversationHistory` from disk, replaces the pending `new_task` tool result with:
    ```text
    Subtask <childTaskId> completed.

    Result:
    <completionResultSummary>
    ```
    and re-instantiates the parent `Task` to continue its loop.
* **Kilo Code XML Envelope (`task.ts:169-179`):** Returns both a tool-call summary list (`<summary>`) and the final assistant text (`<task_result>`) wrapped in `<task id="${session.id}" state="completed">...</task>`.
* **Cline Outcome Fragments (`team-tools.ts:118-174`):** Beyond raw text returns (`SpawnAgentOutput: { text, iterations, finishReason, usage }`), Cline Teams allow teammates to attach typed `TeamOutcomeFragment` artifacts to a shared `TeamOutcome`, which a lead or reviewer explicitly approves/rejects via `team_review_outcome_fragment` before calling `team_finalize_outcome`.

### 6.4 Agent Lifespan & Context Window Management
* **Roo Code:** Parent tasks are completely unloaded from memory while a child runs, eliminating idle memory overhead. Each task manages its own sliding window and auto-condense threshold.
* **Kilo Code:** Supports synchronous subagents by default and asynchronous background subagents via `SubagentRunRegistry` when `Flag.KILO_EXPERIMENTAL_BACKGROUND_SUBAGENTS` is enabled (`task.ts:127-166`), with session resumption via `task_id`.
* **Cline:** `TeamChildSessionManager` (`repos/cline/sdk/packages/core/src/session/team/team-child-session-manager.ts`) manages concurrent teammate sessions (`SessionRuntime`) and incremental state persistence via `AgentTeamsRuntime.drainStateDelta()` (`multi-agent.ts:44-59`).

### 6.5 Fast vs. Slow / Model Mixing & Cross-Provider Support
* **Roo Code Per-Mode API Profiles:** Every Roo mode (`orchestrator`, `architect`, `code`, `debug`, `ask`) can be bound to a distinct API configuration profile (e.g., Claude Opus / o3 for `orchestrator` and `architect`, Claude Sonnet / DeepSeek / Gemini Flash for `code`).
* **Kilo Code `Scout` vs. `General`:** Adds a dedicated `scout` subagent (`PROMPT_SCOUT` in `src/agent/prompt/scout.txt`) alongside `explore` and `general`, with per-agent model overrides.
* **Cline Dynamic Subagent Config:** `spawn_agent` (`spawn-agent-tool.ts:27-35`) allows the parent LLM to dynamically pass a custom `systemPrompt` and `maxIterations` on the fly, while `DelegatedAgentConfigProvider` allows switching provider/model for subagents.

### 6.6 Human-in-the-Loop & Quality Gates
* **Roo Code:** `new_task` requires user approval unless "Auto-approve mode switch / subtask creation" is enabled. Because the child becomes the foreground UI task, the user can watch, steer, or abort the child directly in the chat pane before it boomerangs back to the parent.
* **Kilo Code (`subagent-permissions.ts`):** `deriveSubagentSessionPermission` strips parent `todowrite` and `task` permissions from children while preserving user-level ask/deny safety guardrails.
* **Cline:** Supports `onSubAgentStart` and `onSubAgentEnd` lifecycle hooks (`spawn-agent-tool.ts:39-57`) and fragment review gates (`team_review_outcome_fragment`).

### 6.7 Persistence, Memory, Learning Across Sessions
* **Roo Code:** Persists the full delegation metadata (`delegatedToId`, `awaitingChildId`, `childIds`, `completedByChildId`) in `GlobalFileNames.apiConversationHistory` and `uiMessages` JSON files on disk (`ClineProvider.ts:2835-2890`).
* **Cline:** Persists team tasks, mailbox messages, mission logs, runs, outcomes, and a SQLite `subagent_spawn_queue` (`repos/cline/sdk/packages/core/src/storage/sqlite/schema.ts:163`).

### 6.8 Notable Design Insights
* **Roo Code's "Boomerang" Memory Model:** By suspending and serializing the parent task to disk while the child task takes over the single active UI slot, Roo Code achieves arbitrary delegation depth with $O(1)$ active session memory and zero UI clutter in a narrow IDE sidebar.
* **Cline's Structured Outcome Fragments (`TeamOutcomeFragment`):** Instead of forcing a team lead to stitch together 5 teammates' final chat messages inside its own context window, teammates attach modular fragments to a shared `TeamOutcome` object that can be reviewed and finalized out-of-band.

---

## 7. Amp (Sourcegraph)

### 7.1 What It Is / OSS Status / Repo URL
* **What it is:** Sourcegraph's frontier coding agent for the terminal and VS Code/Cursor.
* **OSS Status:** Closed Source / Proprietary commercial product (`ampcode.com`).
* **Repo / Docs URL:** `https://ampcode.com/manual`

### 7.2 Topology & Hierarchy
* **Depth-1 Specialized Subagent Architecture:** The main Amp agent orchestrates work and delegates to three classes of subagents (depth = 1; subagents cannot recursively spawn further subagents):
  1. **`Task` (General Execution Subagent):** Spawns an independent Amp worker instance with its own context window and full file/shell tools for parallelizable coding or multi-file refactors.
  2. **`Oracle` (Slow-Reasoning Advisor Subagent):** A read-only consulting subagent backed by a high-reasoning model (e.g., OpenAI `o3` / `gpt-5` reasoning tier) invoked when the main agent needs architectural critique, deep debugging analysis, or complex implementation planning.
  3. **`Librarian` (Cross-Repository Discovery Subagent):** A specialized research subagent connected to Sourcegraph's code intelligence and GitHub search across both public open-source repositories and private organizational repos.

### 7.3 Information Passing
* **Zero-History Prompt + Context Handoff:** Subagents do **not** inherit the parent thread's conversation history. The main agent must explicitly synthesize the problem statement, relevant file paths, constraints, and context into the tool call arguments (`Task`, `oracle`, or `librarian`).
* **Return Value:** Subagents stream their tool progress in the UI and return a single synthesized markdown report to the parent thread.

### 7.4 Agent Lifespan & Context Window Management
* **Ephemeral Per-Invocation:** Each `Task`, `oracle`, or `librarian` call runs in a fresh, ephemeral context window and terminates once its final answer is returned.
* **Thread Handoff / Forking for Main Threads:** Rather than relying solely on lossy in-place compaction when a main thread gets long, Amp emphasizes **Thread Handoff**—extracting relevant context from the current thread into a clean new thread to keep context windows sharp.

### 7.5 Fast vs. Slow / Model Mixing & Cross-Provider Support
* **Explicit Multi-Provider Fast/Slow Specialization:** Amp was the pioneer of pairing an **Anthropic Claude** primary execution loop (optimized for fast, reliable tool use and code editing) with an **OpenAI reasoning model (`o3` / `gpt-5` series)** as the **`Oracle`** subagent, and fast search-optimized models for the **`Librarian`** and **Fast / Smart capability modes**.
* **Fast vs. Smart Modes:** Users and workflows can toggle between **Smart Mode** (unconstrained frontier models for complex engineering) and **Fast / Rush Mode** (faster, lower-latency models for quick edits).

### 7.6 Human-in-the-Loop & Quality Gates
* **Read-Only Enforcement on Oracle & Librarian:** Neither the Oracle nor the Librarian can mutate workspace files or run arbitrary destructive shell commands; all code changes flow back through the main agent's tool permission system.

### 7.7 Persistence, Memory, Learning Across Sessions
* **Shared Thread Links & `AGENTS.md`:** Amp persists all threads (including nested subagent call traces) to the user/workspace account on `ampcode.com` (enabling team-wide URL sharing of agent transcripts) and loads hierarchical `AGENTS.md` files for persistent project memory.

### 7.8 Notable Design Insights
* **The "Oracle Pattern" (Cross-Family Reasoning Escalation):** Keeping the primary coding loop on a fast, high-throughput tool-calling model (like Claude Sonnet/Opus) while escalating only hard algorithmic or architectural bottlenecks to a slow thinking model (`Oracle` on OpenAI o-series/GPT-5) avoids paying the latency and token tax of deep thinking on every routine `read_file` or `edit_file` step.

---

## 8. Factory Droids (Factory.ai)

### 8.1 What It Is / OSS Status / Repo URL
* **What it is:** Factory's enterprise software engineering agent platform (`droid` CLI and cloud platform).
* **OSS Status:** Closed Source / Proprietary commercial product (`factory.ai`), with open plugin/skill repositories (`Factory-AI/factory-plugins`).
* **Repo / Docs URL:** `https://docs.factory.ai`

### 8.2 Topology & Hierarchy
* **Primary Droid + Custom Subagent Droids (`Task` tool):** The primary Droid orchestrates work and dispatches scoped subtasks to **Custom Droids** via the `Task` tool.
* **Depth-1 Star Topology:** Custom Droids run as isolated child sessions under the primary Droid and return their synthesized results upward; recursive multi-layer sub-droid spawning is prevented to keep execution deterministic and auditable.

### 8.3 Information Passing
* **Scoped Prompt Dispatch:** When invoking the `Task` tool, the primary Droid selects a registered subagent (`subagent_type` matching a Custom Droid's `name`) and passes a self-contained task prompt. The child Droid boots with a clean context window containing only its markdown system prompt, configured MCP/tool permissions, and the delegated task prompt.
* **Return Payload:** Returns the sub-droid's final completion report to the primary Droid's tool result.

### 8.4 Agent Lifespan & Context Window Management
* **Foreground & Background Execution:** Sub-droids can run synchronously in the foreground (blocking until completion) or asynchronously in the background for parallel verification/research streams. Each invocation gets an isolated context window that is discarded after its summary is returned to the parent.

### 8.5 Fast vs. Slow / Model Mixing & Cross-Provider Support
* **Per-Droid Model Configuration (`.factory/droids/<name>.md`):** Custom Droids are defined in `.factory/droids/` (project) or `~/.factory/droids/` (personal) or bundled inside `.factory-plugin/` packages using YAML frontmatter:
  * `name`, `description`, `model` (e.g., pinning a fast model for a log-triage droid and a frontier reasoning model for a security-auditor droid), `reasoningEffort`, and `tools` (explicit allowlists or category groups like `read-only`, `edit`, `execute`).
* **Cross-Provider Support:** Supports Anthropic Claude, OpenAI GPT/o-series, and Google Gemini models across different Droids within the same workflow.

### 8.6 Human-in-the-Loop & Quality Gates
* **Least-Privilege Tool Scoping & Hooks:** Each Custom Droid declares its exact tool allowlist in frontmatter (e.g., a `security-reviewer` droid is restricted to read-only file and static analysis tools). Factory also supports pre/post lifecycle **Hooks** and enterprise autonomy levels that gate shell execution and external network calls.

### 8.7 Persistence, Memory, Learning Across Sessions
* **Shareable `.factory-plugin/` Bundles & `AGENTS.md`:** Engineering organizations package Custom Droids alongside Skills, Hooks, Slash Commands, and MCP server configs into versioned plugins so specialized subagents are shared across all repositories.

### 8.8 Notable Design Insights
* **Packaging Subagents + Hooks + MCPs as Atomic Plugins:** Instead of treating subagents as standalone prompt files, Factory's `.factory-plugin` unit bundles the subagent (`droids/*.md`), its domain knowledge (`skills/`), its external tools (`mcp.json`), and its deterministic guardrails (`hooks/`) into a single distributable artifact.

---

## 9. Cursor (Agent & Subagents)

### 9.1 What It Is / OSS Status / Repo URL
* **What it is:** Anysphere's AI-first IDE and CLI (`cursor-agent`) with native foreground/background subagents and cloud background agents.
* **OSS Status:** Closed Source / Proprietary commercial product (`cursor.com`).
* **Repo / Docs URL:** `https://cursor.com/docs/context/subagents`

### 9.2 Topology & Hierarchy
* **Orchestrator + Single-Layer Subagents (Depth = 1):** Cursor's main Agent can dynamically spin up built-in or custom subagents (`Task` / subagent tool calls) in parallel or sequentially.
* **Built-In Subagents:** Cursor ships built-in subagents for context-heavy operations—specifically **Explore** (codebase search), **Bash** (noisy shell command execution and log filtering), and **Browser** (Playwright/DOM interaction)—so raw grep results, terminal logs, and DOM snapshots stay out of the main orchestrator's context window.
* **No Recursive Subagent Nesting:** Subagents cannot spawn nested subagents; all delegation is flat from the top-level orchestrator.

### 9.3 Information Passing
* **Isolated Context Handoff:** Subagents start with a fresh context window containing their custom markdown system prompt, workspace rules, and the task prompt crafted by the parent agent.
* **Cross-Tool Discovery Compatibility:** Cursor discovers custom subagent definitions not only from **`.cursor/agents/*.md`** and **`~/.cursor/agents/*.md`**, but also natively reads **`.claude/agents/*.md`** and **`.codex/agents/`** for cross-harness repository compatibility.

### 9.4 Agent Lifespan & Context Window Management
* **Foreground vs. Background + Resumable Subagent IDs:**
  * **Foreground:** Blocks the parent turn until the subagent finishes and returns its final report.
  * **Background:** Runs asynchronously while the main agent continues working or spawns sibling subagents in parallel.
  * **Resumability:** Each subagent execution produces an agent ID that can be resumed with follow-up prompts while preserving that subagent's internal conversation transcript.

### 9.5 Fast vs. Slow / Model Mixing & Cross-Provider Support
* **Frontmatter Model Selection (`.cursor/agents/<name>.md`):**
  ```yaml
  ---
  name: verifier
  description: Validates completed work by running tests and checking edge cases.
  model: fast # or 'inherit', or a specific model ID (Claude, GPT, Gemini)
  readonly: true
  is_background: false
  ---
  ```
* Built-in `Explore` and `Bash` subagents use fast, low-cost models (including Cursor's own fast models / Haiku / Flash tiers), while custom planner/verifier subagents can pin frontier models across Anthropic, OpenAI, or Google.

### 9.6 Human-in-the-Loop & Quality Gates
* **`readonly: true` Enforcement & Independent Verifier Pattern:** Setting `readonly: true` in frontmatter strips file-modification tools from the subagent. Cursor specifically promotes the **Verifier subagent pattern**—an independent subagent spawned after implementation finishes to run the test suite and audit git diffs with skeptical instructions before reporting completion to the user.

### 9.7 Persistence, Memory, Learning Across Sessions
* **Project Rules (`.cursor/rules/*.mdc`), `AGENTS.md`, and Cloud Background Agents:** Subagent transcripts are persisted in local Cursor workspace storage, while long-running asynchronous tasks can be dispatched to remote **Cursor Background Agents** running in isolated cloud VMs on Git branches.

### 9.8 Notable Design Insights
* **Built-In `Bash` and `Browser` Subagents as "Noise Filters":** Most harnesses only provide an `Explore` subagent for file search. Cursor additionally routes **shell execution (`Bash`)** and **DOM automation (`Browser`)** through dedicated subagents because a single verbose `npm test` output or accessibility tree dump can consume 20k–50k tokens.

---

## 10. oh-my-opencode / OmO (`code-yeongyu/oh-my-opencode`)

### 10.1 What It Is / OSS Status / Repo URL
* **What it is:** A multi-model, multi-agent orchestration harness ("Sisyphus / Ultrawork") originally built as an OpenCode plugin and now evolved into a native TypeScript + Rust distribution (`omo` binary built on `senpi`, a fork of `pi-mono`).
* **OSS Status:** Source-available / Open Source under SUL-1.0 License.
* **Repo URL:** `https://github.com/code-yeongyu/oh-my-opencode` (inspected locally at `repos/oh-my-opencode`).

### 10.2 Topology & Hierarchy
OmO implements the most elaborate role- and model-specialized multi-agent hierarchy among open-source coding harnesses, spanning two architectures in the repository:

1. **Classic Named-Pantheon Architecture (OpenCode Plugin Edition, `docs/guide/overview.md`):**
   * **Primary Orchestrators:** `Sisyphus` (main ultrawork orchestrator), `Hephaestus` ("The Legitimate Craftsman", autonomous deep worker inspired by Amp), `Prometheus` (interview-driven strategic planner), and `Atlas` (master todo-list conductor).
   * **Specialized Subagents:** `Oracle` (read-only high-reasoning debugging/architecture advisor), `Librarian` (multi-repo/docs/web search), `Explore` (fast read-only codebase grep/AST search), `Metis` (pre-planning gap consultant), `Momus` (adversarial plan reviewer), `Multimodal-Looker` (PDF/image analyzer), and `Sisyphus-Junior` (category-spawned execution worker).
2. **Senpi Native Edition (`packages/senpi-task/`, `docs/reference/features.md`):**
   * **Orchestrator-Only Rule (`/ulw-execute`):** The main agent is **strictly forbidden from writing product code directly**; it acts purely as an orchestrator that reads state from `.omo/boulder.json` and `.omo/ulw-execute/ledger.jsonl` and delegates all implementation and verification via `task`.
   * **Depth Limit = 1 (`packages/senpi-task/src/index.ts:36-38`):** Child sessions created by `task` are initialized with `excludeTaskTool = true` (`buildChildToolNames(getToolNames(), true)`), which removes both `task` and `task_Manage` from children so subagents cannot recursively spawn subagents.
   * **Workflow DAGs (`mass-ulw`) & Peer Team Mode (`packages/team-core`, `packages/senpi-task/src/team/`):** `workflow` executes multi-phase DAGs (`plan -> approve -> execute -> verify -> complete`), while `team_mode` exposes 12 `team_*` tools (`max_parallel_members: 4`, `max_members: 8`).

### 10.3 Information Passing
* **Task Delegation (`packages/senpi-task/src/types.ts`):** The `task` tool requires:
  * `description` (3–5 words), `prompt`, `run_in_background: boolean`, `load_skills: string[]` (mandatory parameter), and **either** `category` (task intent) **or** `subagent_type` (specific agent role), plus optional `session_id` to resume an existing child session.
* **Contract-Driven Plan & Gate Ledgers (`.omo/`):**
  * During `/ulw-plan`, the planner writes `.omo/plans/<slug>.md` with explicit `Must Have`, `Must NOT Have`, and `Acceptance Criteria`, then runs up to **5 adversarial review rounds** with `plan-reviewer` until it emits `VERDICT: PASS`.
  * During `/ulw-execute`, waves of tasks record status transitions in `.omo/ulw-execute/ledger.jsonl` and verify completion through a 3-stage pipeline: `omo-native-code-reviewer` $\rightarrow$ `omo-native-qa-executor` (runs real terminal/Playwright commands) $\rightarrow$ `omo-native-gate-reviewer` (checks diffs against the plan contract).
* **Kibitzer Sidecar Memory Injection (`packages/memory-core`):** A resident background sidecar agent (`Kibitzer`) ingests a bounded, redacted stream of main-session tool events (`session_start`, `user_prompt_submit`, `post_tool_use`, `stop`) and injects a concise one-line `recalled memory: <hint>` nudge when the main agent is about to repeat a known mistake.

### 10.4 Agent Lifespan & Context Window Management
* **Background/Foreground Subagent Manager (`packages/senpi-task/src/subagent-manager.ts`):** Supports synchronous (`run_in_background: false`) and background (`run_in_background: true`) execution, managed via `task_Manage(action: "list" | "status" | "result" | "cancel")` and resumable via `session_id`.
* **Session Compaction & Boulder Continuity:** Long-running `/ulw-execute` sessions persist execution state in `.omo/boulder.json` so if the orchestrator compacts or restarts, it resumes from the exact incomplete wave in `.omo/ulw-execute/ledger.jsonl`.

### 10.5 Fast vs. Slow / Model Mixing & Cross-Provider Support
OmO's defining feature is **intent-based multi-provider model routing with automatic fallback chains** (`docs/reference/features.md:97-130`):

| Subagent / Category | Primary Model Tier (Oct 2026 Config) | Fallback Chain |
|---|---|---|
| `explore` / `librarian` | `kimi-for-coding-highspeed` | `gpt-6-luna-fast` $\rightarrow$ `claude-haiku-5-5` $\rightarrow$ `deepseek-flash` |
| `plan-consultant` / `architect` | `claude-fable-5-1` (high) | `gpt-6-astra` (high) $\rightarrow$ `glm-6-prime` |
| `plan-reviewer` / `ultrabrain` / `deep-high` | `gpt-6-astra` (high / xhigh) | `claude-fable-5-1` (high) $\rightarrow$ `gemini-3.5-pro` |
| `visual-engineering` / `artistry` | `claude-fable-5-1` (high / max) | `gemini-3.5-pro` $\rightarrow$ `gpt-6-astra` |
| `deep-low` | `gpt-6.1-sol` (medium) | `claude-sonnet-5-0` $\rightarrow$ `gemini-3.5-flash` |
| `quick` | `gpt-6-luna-fast` | `claude-haiku-5-5` $\rightarrow$ `gemini-3.5-flash` |
| `writing` | `claude-sonnet-5-0` | `gemini-3.5-flash` $\rightarrow$ `gpt-6.1-sol` |

### 10.6 Human-in-the-Loop & Quality Gates
* **Multi-Agent Verification Triad:** Instead of trusting a worker subagent's self-reported success, `/ulw-execute` enforces separation of duties:
  1. Worker subagent implements the change and runs unit tests.
  2. `omo-native-code-reviewer` performs static diff review.
  3. `omo-native-qa-executor` executes end-to-end CLI/browser checks.
  4. `omo-native-gate-reviewer` audits all evidence against `.omo/plans/<slug>.md` before marking the ledger entry passed.

### 10.7 Persistence, Memory, Learning Across Sessions
* **Three-Tier Memory (`packages/memory-core` & `.omo/`):**
  1. **Episodic / Ledger State:** `.omo/boulder.json`, `.omo/plans/<slug>.md`, `.omo/ulw-execute/ledger.jsonl`, and `.omo/notepads/<plan-name>/{learnings,decisions,issues,problems}.md`.
  2. **Resident Sidecar (`Kibitzer`):** Real-time memory retrieval and synthesis across sessions.
  3. **Team State (`packages/team-core`):** Persisted in `.omo/teams/<team-name>/`.

### 10.8 Notable Design Insights
* **Decoupling Task Intent (`category`) from Hardcoded Model Names:** Letting the orchestrator call `task(category="ultrabrain", ...)` or `task(category="visual-engineering", ...)` instead of hardcoding model IDs allows the harness to route each subtask to the best provider/model family with automatic fallback chains.
* **The Resident "Kibitzer" Memory Sidecar:** Running a cheap, low-latency model in parallel as a read-only observer of the main agent's tool stream solves the problem that primary coding agents rarely remember to proactively call a `memory_search` tool before making a mistake.

---

## 11. Crush (`charmbracelet/crush`)

### 11.1 What It Is / OSS Status / Repo URL
* **What it is:** Charmbracelet's terminal-based coding agent (the successor/evolution in the Charm ecosystem), written in Go on top of the `charm.land/fantasy` agent library.
* **OSS Status:** Open Source (FSL-1.1-MIT / Charmbracelet open source).
* **Repo URL:** `https://github.com/charmbracelet/crush` (inspected locally at `repos/crush`).

### 11.2 Topology & Hierarchy
* **Coordinator + Two Built-In Subagent Tools (`internal/agent/`):**
  1. **`agent` tool (`internal/agent/agent_tool.go`):** Spawns the configured `AgentTask` subagent (`config.AgentTask` in `internal/config/config.go:1129-1143`) via `fantasy.NewParallelAgentTool`.
  2. **`agentic_fetch` tool (`internal/agent/agentic_fetch_tool.go`):** Spawns a specialized web-research subagent (`fantasy.NewParallelAgentTool`) in a temporary working directory (`os.MkdirTemp("", "crush-fetch-*")`) when the parent invokes `agentic_fetch(prompt, url?)`.
* **Strict Depth Limit = 1 (`internal/config/config.go:1103-1143`):** `resolveReadOnlyTools` explicitly excludes `agent` and `agentic_fetch` from `AgentTask`'s `AllowedTools` (`glob`, `grep`, `ls`, `lsp_call_hierarchy`, `lsp_definition`, `lsp_symbols`, `sourcegraph`, `view`). Subagents cannot spawn nested subagents.

### 11.3 Information Passing
* **Prompt-Only Fresh Child Session (`runSubAgent` in `internal/agent/coordinator.go:639-725`):**
  * When `agent` or `agentic_fetch` runs, `runSubAgent` creates a fresh child session in SQLite using a hierarchical ID (`c.sessions.CreateAgentToolSessionID(agentMessageID, call.ID)`), passes only `params.Prompt`, and returns `result.Response.Content.Text()` as the tool response (` fantasy.NewTextResponse`).
* **Automatic Cost Rollup (`updateParentSessionCost` in `coordinator.go:728-763`):** Upon child completion, `runSubAgent` fetches the child session's accumulated token cost (`childSession.Cost`) and atomically adds it to the parent session (`parentSession.Cost += childCost`), traversing up the parent chain if necessary.

### 11.4 Agent Lifespan & Context Window Management
* **Ephemeral Parallel Execution:** Both `agent` and `agentic_fetch` are registered with `fantasy.NewParallelAgentTool`, allowing the Go runtime to execute multiple subagents concurrently in parallel goroutines during a single parent turn. Each subagent call is single-turn/stateless from the parent's perspective (no `session_id` resume parameter exposed in `AgentParams`).

### 11.5 Fast vs. Slow / Model Mixing & Cross-Provider Support
* **Dual-Tier Model Architecture (`LargeModel` vs. `SmallModel` in `internal/agent/coordinator.go:810-900`):**
  * Crush configures two model slots per session: `ModelTypeLarge` (for complex reasoning and coding) and `ModelTypeSmall` (for fast/cheap operations like summarization and web fetching).
  * `agent` (`AgentTask`) runs using the configured `AgentTask.Model` (`ModelTypeLarge` by default, configurable in `crush.json`).
  * `agentic_fetch` explicitly swaps both slots to the small model (`agent.SetModels(small, small)` in `agentic_fetch_tool.go:142`) so reading large documentation pages or scraping search results uses a cheap model (`SmallModel`) before returning a distilled markdown answer to the `LargeModel`.

### 11.6 Human-in-the-Loop & Quality Gates
* **Non-Interactive Subagent Permissions & Hook Bypass:**
  * `AgentTask` is strictly read-only (`AllowedTools` restricted to file/LSP/Sourcegraph readers; `AllowedMCP` empty by default).
  * `agentic_fetch` auto-approves permissions for its isolated child session (`c.permissions.AutoApproveSession(sessionID)`, `agentic_fetch_tool.go:150`) and confines file writes/reads to its ephemeral `crush-fetch-*` temp directory (`os.RemoveAll(tmpDir)` on exit).
  * In `buildTools` (`coordinator.go:797`), `wrapToolsWithHooks` receives `isSubAgent = true`, bypassing interactive user hooks for internal subagents.

### 11.7 Persistence, Memory, Learning Across Sessions
* **SQLite Task Sessions (`internal/session/`):** Even though subagents are ephemeral in the prompt loop, `c.sessions.CreateTaskSession` persists every child session's full message history, tool calls, and token usage in Crush's local SQLite database for TUI inspection and cost accounting.

### 11.8 Notable Design Insights
* **`agentic_fetch` as an Isolated Small-Model Sandbox:** When a coding agent fetches a large documentation page, raw HTML-to-markdown conversion can dump 30,000+ tokens into the main frontier model's context. Crush's `agentic_fetch` writes large payloads exceeding `LargeContentThreshold` to a temporary file inside `/tmp/crush-fetch-*`, spins up the `SmallModel` with `grep`/`view` tools to extract the exact answer, deletes the temp directory, and returns only the concise answer to the `LargeModel`.

---

## 12. Aider (`Aider-AI/aider`)

### 12.1 What It Is / OSS Status / Repo URL
* **What it is:** The classic open-source terminal pair-programming CLI that pioneered repository maps (`RepoMap`), pluggable edit formats (`diff`, `udiff`, `whole`, `editor-diff`), and the **Architect / Editor** dual-model pattern.
* **OSS Status:** Open Source (Apache-2.0 License).
* **Repo URL:** `https://github.com/Aider-AI/aider` (inspected locally at `repos/aider`).

### 12.2 Topology & Hierarchy
* **Fixed Two-Stage Pipeline (`ArchitectCoder` $\rightarrow$ `EditorCoder`, Depth = 1):**
  * Unlike tool-calling subagent trees, Aider's `--architect` mode (`repos/aider/aider/coders/architect_coder.py`) implements a deterministic **two-stage sequential handoff** inside `ArchitectCoder(AskCoder)`.
  * **Stage 1 (Architect):** `ArchitectCoder` runs `self.main_model` using `ArchitectPrompts` (`aider/coders/architect_prompts.py`) with **no file-editing format**—its sole job is to analyze the user request, repository map, and added files, and output a clear natural-language specification of the required code changes.
  * **Stage 2 (Editor):** Inside `ArchitectCoder.reply_completed()` (`architect_coder.py:21-68`), Aider instantiates a transient child `Coder` (`editor_coder = Coder.create(**kwargs)`) configured with `main_model = self.main_model.editor_model or self.main_model` and `edit_format = self.main_model.editor_edit_format` (`editblock-editor` or `whole-editor`).

### 12.3 Information Passing
* **Curated Context Reset (`architect_coder.py:45-53`):**
  * When `editor_coder` is cloned from `self` (`from_coder=self`), it inherits the active set of chat files (`abs_fnames`), but **explicitly wipes the conversation history and repository map**:
    ```python
    editor_coder.cur_messages = []
    editor_coder.done_messages = []
    ```
    with `map_tokens=0` and `suggest_shell_commands=False`.
  * The Architect's natural-language response (`content = self.partial_response_content`) is passed as the **sole user prompt** to `editor_coder.run(with_message=content, preproc=False)`.
  * Once `editor_coder` finishes applying edits, `ArchitectCoder` copies `self.total_cost = editor_coder.total_cost` and `self.aider_commit_hashes = editor_coder.aider_commit_hashes` back to the parent (`architect_coder.py:57-58`).

### 12.4 Agent Lifespan & Context Window Management
* **Single-Turn Ephemeral Editor:** The `ArchitectCoder` persists across the entire user session (`cur_messages` / `done_messages`, summarized on overflow via `ChatSummary` in `aider/history.py`), whereas `editor_coder` is instantiated, executed for a single editing turn (with self-reflection retries if lint/test or diff application fails), and garbage-collected at the end of `reply_completed()`.

### 12.5 Fast vs. Slow / Model Mixing & Cross-Provider Support
* **The Canonical Slow-Reasoning + Fast-Syntactic Split:**
  * **Architect (`--model`):** Frontier reasoning model (e.g., `o1`/`o3`/`gpt-5`, `deepseek-r1`, `claude-opus`) that excels at algorithms and system design but may be slow or inconsistent at emitting strict search/replace diff blocks.
  * **Editor (`--editor-model`):** Fast, syntax-accurate coding model (e.g., `claude-sonnet`, `gpt-4.1` / `gpt-5-mini`, `deepseek-chat`) tuned for Aider's `editor-diff` (`SEARCH`/`REPLACE`) or `editor-whole` formats.
* **Cross-Provider Support:** Powered by `litellm`, allowing any provider combination (e.g., DeepSeek R1 or OpenAI o3 as Architect + Anthropic Sonnet as Editor).

### 12.6 Human-in-the-Loop & Quality Gates
* **Interactive Handoff Gate (`architect_coder.py:25-26`):** Before spawning `editor_coder`, if `auto_accept_architect` is false, Aider pauses after printing the Architect's plan and asks `self.io.confirm_ask("Edit the files?")`. After `editor_coder` applies edits, Aider's standard `--auto-lint` and `--auto-test` gates run and can feed lint/test errors back into the coder loop.

### 12.7 Persistence, Memory, Learning Across Sessions
* **Git-Native Persistence & `.aider.chat.history.md`:** Every successful `editor_coder` pass auto-commits changes to Git with an AI-generated commit message (`aider_commit_hashes`) and logs the full Architect + Editor exchange to `.aider.chat.history.md`.

### 12.8 Notable Design Insights
* **Zero-History, Zero-RepoMap Editor Context (`map_tokens=0`):** By stripping the entire chat history and the repository map from the Editor model—giving it *only* the target files and the Architect's instructions—Aider drastically reduces Editor input tokens and prevents the Editor from getting confused by earlier rejected ideas in the conversation history.

---

## Cross-Product Comparison Matrix

| Product | License / Language | Max Subagent Depth | Context Handoff (Parent $\rightarrow$ Child) | Child Return Payload | Child Lifespan & Resume | Sibling / Peer Messaging | Fast / Slow Model Mixing |
|---|---|---|---|---|---|---|---|
| **OpenAI Codex CLI** (`openai/codex`) | Apache-2.0 (Rust) | **V1:** Default `1` (`agent_max_depth` configurable)<br>**V2:** Configurable tree (`/root/...`) | **V1:** Prompt-only (or `fork_context: true`)<br>**V2:** `fork_turns: "all"` (curated rollout filter preserving prompt cache) | `InterAgentCommunication` completion message (`trigger_turn: false`) | Stateful multi-turn threads; V2 evicts idle threads to disk (`ResidencySlot`) while keeping `Mailboxes` | **Yes:** V2 `send_message` / `followup_task` by `AgentPath` + `agent-message-board` pub/sub | Built-in `explorer`, `worker`, `awaiter` (`low` effort) + `.codex/agents/*.toml` |
| **OpenCode** (`anomalyco/opencode`) | MIT (TypeScript / Effect) | Default `1` (`experimental.subagent_depth`) | Prompt-only fresh session (`"You are a subagent..."`) | `<subagent sessionID="..." state="...">` XML envelope (sync or via SQLite `SessionInbox`) | Stateful SQLite session; resumable via `sessionID` param | No (parent-mediated star/tree) | Per-agent config + dynamic per-call `model` param in `subagent` tool |
| **Claude Code** (`anthropics/claude-code`) | Proprietary CLI + OSS SDKs | **Subagents:** Default `3` (`CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`)<br>**Teams:** `1` (flat teammates) | **Standard:** Prompt + `CLAUDE.md` (omitted on `Explore`/`Plan`)<br>**Fork (`subtask`):** Full parent history + cache reuse | Sanitized final report (escaped `<system-reminder>` / role tags) | Stateful JSONL transcript; resumable via `SendMessage(to: agentId)` | **Yes:** Sibling roster in subagents (`SendMessage`) + full P2P `inboxes/*.json` & shared tasks in Agent Teams | `Explore`/`claude-code-guide` on Haiku/Opus; custom frontmatter `model` + `effort` |
| **Gemini CLI** (`google-gemini/gemini-cli`) | Apache-2.0 (TypeScript) | Strict `1` (`Kind.Agent` tools stripped from children) | Prompt / typed Zod `AgentInputs` (isolated `GeminiChat`) | Mandatory `complete_task` tool call validated against Zod `outputConfig.schema` | Ephemeral local runs (`maxTurns: 50`, `maxTimeMinutes: 10`); stateful remote A2A `contextId` | No (parent-mediated; remote A2A over HTTP) | `codebase_investigator` on `gemini-3-flash-preview` (`HIGH` thinking); `generalist` inherits |
| **Goose** (`block/goose`) | Apache-2.0 (Rust) | Strict `1` (`SessionType::SubAgent` blocked from `delegate`) | `instructions` + explicit `context` section (`DelegateParams`) | JSON schema `{ "summary": string }` extracted by `readable_output` | Synchronous foreground run (`max_turns = 25`), persisted in SQLite | No | Per-call `provider`, `model`, `temperature`, and `extensions` filter on `delegate` |
| **Roo Code** (`RooCodeInc/Roo-Code`) | Apache-2.0 (TypeScript) | Arbitrary stack depth (`clineStack`) | `new_task(mode, message, todos)` (suspends parent to disk) | `attempt_completion(result)` reloads parent from disk (`reopenParentFromDelegation`) | Single active task ("Boomerang"); parent unloaded while child runs | No (strict call stack) | Per-mode API profile binding (`orchestrator`, `architect`, `code`, `debug`) |
| **Kilo Code** (`Kilo-Org/kilocode`) | Apache-2.0 (TypeScript) | Default `1` (`task: "deny"` in `subagent-permissions.ts`) | Prompt-only fresh session (`TaskTool`) | `<task id="..." state="..."><summary>...<task_result>...</task>` | Stateful session; resumable via `task_id`; optional background registry | No | Built-in `explore`, `scout`, `general` with per-agent model overrides |
| **Cline** (`cline/cline`) | Apache-2.0 (TypeScript) | `1` for `spawn_agent` & `AgentTeamsRuntime` | Prompt + dynamic `systemPrompt` (`spawn_agent`) or Team task brief | `{ text, iterations, finishReason, usage }` or `TeamOutcomeFragment` | Ephemeral `spawn_agent` (5m timeout) or stateful `TeamChildSessionManager` | **Yes:** `team_send_message`, `team_broadcast`, `team_read_mailbox`, `TeamOutcome` | `DelegatedAgentConfigProvider` + per-teammate `AgentConfig` |
| **Amp** (Sourcegraph) | Proprietary | Strict `1` | Prompt-only handoff (`Task`, `oracle`, `librarian`) | Synthesized markdown report | Ephemeral per tool call | No | Primary loop (Claude) + `Oracle` (OpenAI `o3`/`gpt-5`) + `Librarian` + Fast/Smart modes |
| **Factory Droids** (Factory.ai) | Proprietary | Strict `1` (`Task` tool $\rightarrow$ Custom Droids) | Prompt-only + `.factory/droids/<name>.md` system prompt | Synthesized markdown report | Ephemeral foreground or background run | No | Per-droid `model` and `reasoningEffort` across Anthropic, OpenAI, Gemini |
| **Cursor** (Anysphere) | Proprietary | Strict `1` (flat orchestrator $\rightarrow$ subagents) | Prompt-only + `.cursor/agents/<name>.md` | Synthesized markdown report | Foreground or background; resumable via subagent ID | No | `Explore`/`Bash`/`Browser` on fast models; custom frontmatter `model: fast | inherit | <id>` |
| **oh-my-opencode (OmO)** (`oh-my-opencode`) | SUL-1.0 (TS & Rust) | Strict `1` (`excludeTaskTool = true` on child sessions) | `task(category | subagent_type, prompt, load_skills)` + `.omo/plans/<slug>.md` | Text result + `.omo/ulw-execute/ledger.jsonl` + review verdicts | Foreground/background (`task_Manage`), resumable via `session_id`; resident `Kibitzer` sidecar | **Yes:** `team_mode` (12 `team_*` tools + mailboxes) | Multi-provider category fallback chains (Claude, GPT-6, Gemini, Kimi, GLM, DeepSeek) |
| **Crush** (`charmbracelet/crush`) | FSL-1.1-MIT (Go) | Strict `1` (`resolveReadOnlyTools` strips `agent`) | Prompt-only fresh SQLite task session | Final text response + automatic cost rollup to parent | Ephemeral parallel goroutines (`NewParallelAgentTool`) | No | `LargeModel` for `AgentCoder`/`AgentTask`; `SmallModel` for `agentic_fetch` |
| **Aider** (`Aider-AI/aider`) | Apache-2.0 (Python) | Fixed `1` (`ArchitectCoder` $\rightarrow$ `EditorCoder`) | Wipes `cur_messages`/`done_messages` & `map_tokens=0`; passes only files + Architect plan | Direct file edits + Git commit (`aider_commit_hashes`) | Single-turn ephemeral `editor_coder` per Architect reply | No | Slow reasoning `--model` (Architect) + fast syntactic `--editor-model` (Editor) |

---

## Key Architectural Patterns & Synthesis

### 1. Tree Depth Limits: Why Depth = 1 Dominates (and When Depth > 1 Works)
Across the 12 harnesses, **9 enforce a default maximum subagent depth of 1** (`OpenCode`, `Gemini CLI`, `Goose`, `Kilo Code`, `Amp`, `Factory`, `Cursor`, `OmO`, `Crush`), blocking subagents from spawning sub-subagents by stripping the delegation tool (`Kind.Agent`, `excludeTaskTool = true`, `subagent: "deny"`) or checking `SessionType::SubAgent`.
* **Why Depth = 1 is the default:** Unbounded recursive spawning creates multiplicative token amplification, deadlocks on concurrency semaphores, and "telephone-game" context drift where a leaf subagent optimizes for a distorted restatement of the user's goal.
* **How Depth > 1 is safely engineered (`Claude Code` depth=3, `Codex CLI V2`, `Roo Code`):**
  * **Hard Global Concurrency Caps + Residency Eviction:** Codex V2 allows deep trees (`/root/task1/sub2`) only because `effective_agent_max_threads` caps active concurrent child turns to `3` and `evict_idle_for_spawn` unloads waiting parent nodes from RAM while children run.
  * **Single-Active-Node Call Stack:** Roo Code supports arbitrary depth because only the leaf task in `clineStack` is ever active in memory (`delegateParentAndOpenChild`).
  * **Explicit Depth Counter Withholding:** Claude Code tracks nesting depth via `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH` (default `3`) and withholds the `Agent` tool once the depth counter hits the ceiling.

### 2. Context Handoff: Prompt-Only Isolation vs. Prefix-Preserving Forking
There is a fundamental tension in subagent design between **context isolation** and **handoff fidelity**:
* **Pattern A — Prompt-Only Fresh Context (`OpenCode`, `Gemini CLI`, `Goose`, `Amp`, `Cursor`, `Crush`):** The child starts with `0` parent history tokens. This is optimal for **context-polluting leaf tasks** (grep/glob exploration, `agentic_fetch` web scraping, running noisy test suites), where the child does not need conversational nuance. Goose improves this by adding a dedicated `context` parameter alongside `instructions` in `delegate`.
* **Pattern B — Prefix-Preserving Forked Context (`Claude Code Fork Subagents`, `Codex CLI V2 fork_turns="all"`):** When delegating complex implementation subtasks, forcing the parent to write a 2,000-word prompt summarizing the conversation wastes output tokens and drops subtle constraints.
  * **Claude Code's `fork` mode** passes the exact parent conversation history, system prompt, and tools so the child gets a **prompt cache hit** on the entire parent prefix.
  * **Codex CLI V2's `prepare_forked_history`** (`core/src/agent/control/spawn.rs`) refines this further: it forks the parent rollout but strips intermediate tool calls/outputs and reasoning traces (`keep_forked_rollout_item`), keeping only user/system prompts and assistant `PartialAnswer`/`FinalAnswer` items.
* **Pattern C — Intentional Context Wiping after Planning (`Aider Architect/Editor`, `Claude Code omitClaudeMd`):** Aider's `ArchitectCoder` proves that an implementation worker often performs *better* when history and repo maps (`map_tokens=0`) are stripped, leaving only the target files and the Architect's specification. Similarly, Claude Code's `omitClaudeMd: true` on `Explore` and `Plan` avoids paying for project style guides on simple file-discovery calls.

### 3. Child Return Payloads: Free-Text vs. Schema-Enforced Contracts
* **Free-Text / XML Envelopes (`OpenCode`, `Kilo Code`, `Claude Code`):** Returning the child's final assistant message inside `<subagent sessionID="..." state="completed">...</subagent>` is flexible, though Claude Code adds a critical **prompt-injection scanner** that escapes `<system-reminder>` and turn markers before the parent ingests the report.
* **Schema-Enforced Termination (`Gemini CLI complete_task`, `Goose Response Schema`, `Cline TeamOutcomeFragment`):** Gemini CLI's `LocalAgentExecutor` requires the child to call `complete_task` matching a Zod schema (e.g., `CodebaseInvestigationReportSchema` with `SummaryOfFindings`, `ExplorationTrace`, `RelevantLocations`), granting a 60-second recovery turn if the child stops without calling it. This turns subagent delegation into a typed RPC contract.

### 4. Child Lifespan: Ephemeral Workers vs. Resumable Actors
Harnesses split into two lifespans:
* **Ephemeral One-Shot Workers (`Gemini CLI local`, `Goose`, `Amp`, `Crush`, `Claude Code Explore/Plan`):** The child session terminates as soon as it returns its summary.
* **Resumable Multi-Turn Actors (`Codex V1/V2`, `OpenCode`, `Claude Code named subagents`, `Kilo Code`, `Cursor`, `OmO`):** Returning a `sessionID` / `agentId` / `AgentPath` allows the parent to ask follow-up questions (`SendMessage`, `followup_task`, `subagent(sessionID=...)`) without re-exploring the codebase.

### 5. Fast vs. Slow Model Roles
Every mature harness separates cognitive roles across model tiers:
1. **High-Speed Search & Noise Filtering (`Fast / Small` tier):** `Explore`, `Librarian`, `Scout`, `Bash`, and `agentic_fetch` subagents run on fast, low-cost models (`claude-haiku`, `gemini-3-flash-preview`, `gpt-6-luna-fast`, `kimi-for-coding-highspeed`, or Crush's `SmallModel`).
2. **Primary Execution & Tool Orchestration (`Balanced Frontier` tier):** The main loop and general workers (`Build`, `Sisyphus`, `general-purpose`, `EditorCoder`) run on models with the highest tool-call and diff-editing reliability.
3. **Slow-Reasoning Consultation & Adversarial Gate Review (`Deep Thinking` tier):** Invoked selectively for architecture, hard debugging, or plan verification (`Amp Oracle`, `OmO Oracle / plan-reviewer / ultrabrain`, `Codex high/xhigh effort`, `Aider ArchitectCoder`).
4. **Resident Background Observer (`Ultra-Cheap Sidecar` tier):** OmO's `Kibitzer` (`packages/memory-core`) watches tool events out-of-band to inject memory recall nudges without slowing down the main loop.

### 6. Cross-Provider Subagent Spawning (In-Process vs. CLI/MCP Bridging)
Cross-provider orchestration happens through two distinct mechanisms in 2026:
1. **Native In-Process Multi-Provider Routing (`OpenCode`, `OmO`, `Goose`, `Crush`, `Aider`, `Cursor`, `Factory`):** Because these harnesses abstract the LLM provider layer (`Provider.parseModel`, `litellm`, `charm.land/fantasy`), a single parent session running on Anthropic Claude can natively spawn an in-process subagent running on OpenAI GPT/o-series, Google Gemini, Kimi, or DeepSeek using a single unified tool registry.
2. **Cross-Harness CLI & MCP Delegation (`Claude Code` $\leftrightarrow$ `Codex CLI` / `Gemini CLI`):** Single-vendor harnesses (like Claude Code or Codex CLI) achieve cross-provider delegation by invoking peer harnesses via **MCP servers** or **Skills/Subagent wrappers**:
   * **MCP Server Bridging:** Custom subagents in Claude Code (`mcpServers` frontmatter in `.claude/agents/*.md`), Gemini CLI (`mcp_servers` in `.gemini/agents/*.md`), and Codex (`codex-rs/codex-mcp`) can attach an external agent process as an MCP tool server or remote A2A endpoint (`RemoteAgentInvocation` in Gemini CLI).
   * **Non-Interactive CLI Exec Skills:** A lightweight read-only subagent or skill invokes `codex exec --full-auto "<prompt>"` (`codex-rs/exec`) or `gemini -p "<prompt>"` via Bash in an isolated git worktree, capturing its stdout/diff and returning it to the parent orchestrator as a cross-model "second opinion" or adversarial reviewer.
   * **Shared `.claude/agents/*.md` Standard:** Notably, **Goose** (`summon.rs:384`), **Cursor**, **OpenCode**, and **OmO** all directly parse `.claude/agents/*.md` frontmatter, making markdown agent definitions the de facto cross-harness standard.

### 7. Inter-Agent Sibling Messaging & Shared State
Most harnesses restrict communication to a **strict parent-child star/tree** (children can only talk to the parent via their return value). However, four harnesses implement **horizontal sibling communication**:
* **OpenAI Codex CLI V2 (`core/src/agent/control/mailbox.rs` & `ext/agent-message-board`):** Any agent in the hierarchy can call `send_message` or `followup_task` targeting a sibling's canonical `AgentPath` (e.g., `/root/backend_worker` messaging `/root/frontend_worker`), or post/subscribe to shared topics on the `agent-message-board`.
* **Claude Code (`SendMessage` Sibling Roster & Agent Teams):** Standard subagents receive a one-line roster of active sibling subagents and can message them via `SendMessage`. In `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`, teammates coordinate via `~/.claude/teams/{team-name}/inboxes/{agent-name}.json` and claim dependency-tracked tasks from `~/.claude/tasks/{team-name}/` using file locks.
* **Cline SDK (`AgentTeamsRuntime` in `multi-agent.ts`):** Teammates exchange direct and broadcast messages (`team_send_message`, `team_broadcast`, `team_read_mailbox`) and collaboratively assemble reviewed `TeamOutcomeFragment` artifacts.
* **oh-my-opencode (`packages/team-core`):** Provides 12 `team_*` tools with file-backed mailboxes and lifecycle management for up to 8 team members (4 parallel).
