# Always-On, Cloud & Persistent Multi-Agent Platforms and Cross-Agent Protocols

This report analyzes the architecture of **13 always-on, cloud, and persistent multi-agent platforms**—systems where agents live as durable identities, background daemons, or cloud VM instances that message, delegate to, and coordinate with one another—alongside the **three foundational cross-agent protocols** (**Google A2A**, **Anthropic MCP**, and **Zed ACP**) that connect them.

Open-source systems are analyzed directly from their cloned repositories under `/Users/jack/workspace/orchestrator-architecture/repos/`, while closed-source platforms are documented from official engineering blogs, technical documentation, API specifications, and published papers.

---

## Table of Contents

1. [xAI Grok Bot (`grok.com/bot` & `grok-4.20-multi-agent`)](#1-xai-grok-bot)
2. [OpenClaw (`openclaw/openclaw`)](#2-openclaw)
3. [Hermes Agent (`NousResearch/hermes-agent`)](#3-hermes-agent)
4. [Manus & OpenManus (`manus.im` & `FoundationAgents/OpenManus`)](#4-manus--openmanus)
5. [Devin (`cognition.ai` / Managed Devins & SWE-grep)](#5-devin-cognition)
6. [Cursor Cloud/Background Agents & Multi-Agent (`cursor.com`)](#6-cursor-cloudbackground-agents--multi-agent)
7. [GitHub Copilot Coding Agent / Agent HQ / Mission Control](#7-github-copilot-coding-agent--agent-hq--mission-control)
8. [Google Jules (`jules.google`)](#8-google-jules)
9. [Google Antigravity Agent Manager (`antigravity.google`)](#9-google-antigravity-agent-manager)
10. [OpenHands (`OpenHands/OpenHands` & `OpenHands/software-agent-sdk`)](#10-openhands)
11. [Letta (`letta-ai/letta` & `letta-ai/letta-code`)](#11-letta)
12. [Kimi Agent Swarm (`Moonshot AI Kimi K2.5`)](#12-kimi-agent-swarm)
13. [Moltbook (`moltbook.com`)](#13-moltbook)
14. [Cross-Agent Protocols: Google A2A, MCP, and Zed ACP](#14-cross-agent-protocols-google-a2a-mcp-and-zed-acp)
15. [Cross-Product Comparison Table](#15-cross-product-comparison-table)
16. [Key Architectural Patterns](#16-key-architectural-patterns)

---

## 1. xAI Grok Bot

### 1. What It Is / OSS Status / Repo URL
- **Status**: Closed-source commercial product by xAI (built in collaboration with the Cursor team; announced September 2026 at `https://x.ai/news/designing-grok-bot`, docs at `https://docs.x.ai/grok-bot/overview`).
- **Scope**: Grok Bot is an always-on digital teammate platform where users maintain a roster of up to **50 named persistent Bots** backed by a shared Linux cloud computer, scheduled/event-driven routines, and native bot-to-bot messaging. It is complemented at the API layer by **`grok-4.20-multi-agent`** (`https://docs.x.ai/developers/model-capabilities/text/multi-agent.md`) and at the terminal layer by Grok CLI subagents (`.grok/agents/`).

### 2. Topology & Hierarchy
- **Peer Roster + Emergent Chief-of-Staff (Grok Bot App)**: Up to ~50 persistent Bots per account live as peers on a visual roster grid. Rather than enforcing a rigid orchestrator tree in the UI, xAI reports that users organically adopt a **Chief of Staff** topology: one generalist coordinator Bot triages incoming requests and dispatches work to specialist peer Bots (e.g., *Inbox Bot*, *Deploy Tracker*, *Customer Follow-up*).
- **Multi-Bot Group Chats**: Users can assemble **2 to 6 Bots** in a single group conversation (`https://docs.x.ai/grok-bot/chat-and-collaboration.md`), either `@mention`ing a specific Bot or allowing the group to self-coordinate based on each Bot's description and role.
- **API Leader + Parallel Subagent Swarm (`grok-4.20-multi-agent`)**: Fixed 2-level hierarchy inside a single Responses API call where a **Leader Agent** orchestrates either **4 subagents** (`reasoning.effort: "low" | "medium"`, `agent_count=4`) or **16 subagents** (`reasoning.effort: "high" | "xhigh"`, `agent_count=16`) that research in parallel before the Leader synthesizes the final response.
- **CLI Subagents**: `.grok/agents/` supports `general-purpose`, `explore` (read-only codebase search), and `plan` subagents running in isolated child sessions.

### 3. Information Passing & Context Handoff
- **Direct Asynchronous Bot-to-Bot Handoffs**: When the account-level **Messaging Between Bots** toggle is enabled (on by default in dashboard settings), any Bot can send an asynchronous message to another Bot in the user's roster (`chat-and-collaboration.md`, *"Let Bots hand work off"*).
  - Sending a message wakes the recipient Bot in its own conversation context.
  - The recipient processes the request asynchronously and replies back to the sender when done.
  - Handoff events render inline in the chat transcript so the human sees what was delegated and returned.
  - *Modality constraint*: Bot-to-group-chat handoffs are **text-only**; images and file attachments must be sent via direct 1:1 bot-to-bot messages or placed on the shared filesystem.
- **Zero-Copy Handoff via Shared Cloud Computer (`/workspace`)**: All personal Bots in a user's account share **one persistent cloud Linux computer** (`/workspace` directory, installed packages, CLI credentials, and logged-in Chromium browser sessions/cookies). One Bot can pull a dataset, save `/workspace/q3-churn.csv` or authenticate a browser session, and message a second Bot with the file path—avoiding context serialization overhead entirely.
  - *Team Bot Computer Scoping*: For Team Bots (`team-bots-and-sharing.md`), computer routing is context-dependent: owner chat uses the owner's computer; a teammate's 1:1 DM with the shared Bot uses *that teammate's* computer; Slack channels/threads use a **dedicated shared Bot computer** isolated from personal VMs.
- **External A2A Protocol**: Grok Bot natively implements Google's **Agent2Agent (A2A)** protocol (`overview.md`) to discover external third-party agents and delegate structured tasks across organizational boundaries without exposing internal credentials or conversation history.
- **API Subagent State (`use_encrypted_content`)**: In `grok-4.20-multi-agent`, internal subagent reasoning and intermediate tool traces are opaque by default and returned as encrypted state blobs (`use_encrypted_content=True`) for multi-turn continuation.

### 4. Agent Lifespan & Context Window Management
- **Persistent Named Identities**: Bots are long-lived entities with continuous conversation histories, visual avatar state machines (`idle`, `thinking`, `working`, `waiting`, `blocked`, `done`), and persistent workspace state.
- **Routines (Cron & Event Listeners)**: Each Bot can hold up to **50 Routines** (`routines-and-automation.md`)—either recurring schedules (minimum **5-minute** interval) or event-driven listeners on connected apps (Gmail, Slack, GitHub, Linear). Each Routine retains its **20 most recent run records** and executes as an autonomous background turn in the Bot's session.
- **Internal Compaction**: Exact context compaction heuristics inside the Grok Bot web runtime are not publicly documented (closed-source).

### 5. Fast vs. Slow / Model Mixing & Cross-Provider Support
- **xAI Model Family**: Powered by the Grok 4 / Grok 4.20 model family, with reasoning effort scaling (`low`, `medium`, `high`, `xhigh`) controlling both thinking budget and active subagent count (4 vs. 16 in the multi-agent API). Cross-vendor LLM routing is not natively supported inside hosted Grok Bot, though external models can be invoked via A2A or MCP servers.

### 6. Human-in-the-Loop & Quality Gates
- **Auto Review Gate (`grok-approvals-security-and-privacy.md`)**: Every sensitive tool call (sending emails, mutating external systems, destructive shell commands, cross-bot actions) passes through **Auto Review**, a model-based risk classifier evaluated against user-defined `Allow automatically` and `Ask first` rules (`Ask first` always takes precedence on conflict).
- **10-Minute Background Approval Expiry**:
  - In **interactive chat**, an approval card blocks the Bot indefinitely until the user clicks Approve or Deny.
  - In **background execution** (Routines, webhook triggers, or work triggered by a message from another Bot), approval cards **automatically expire after ~10 minutes** so unattended background bots do not hang indefinitely on stale side effects.
- **Three-Level Computer View & Hardware Security Key Forwarding**: Users monitor the cloud computer via **Status** (purple activity pill), **Preview** (live read-only side panel), or **Takeover** (full interactive desktop control to solve CAPTCHAs, enter passwords, or complete 2FA, including **physical YubiKey/WebAuthn forwarding** from the user's local browser into the remote cloud VM).

### 7. Persistence, Memory & Learning Across Sessions
- **Split Account vs. Bot Scoping**:
  - **Capabilities are Account-Scoped**: Connected **Tools** (OAuth integrations, MCP servers) and **Skills** (reusable procedural playbooks, including **Learn from Demonstration** where a human performs a task in Computer Takeover mode and Grok distills a repeatable skill) are shared across all Bots in the account.
  - **Context is Bot-Scoped**: **Memory**, **Routines**, and conversation transcripts belong exclusively to each individual Bot so specialist personas stay focused.
- **Dual Memory in Team Bots**: Shared Team Bots maintain two separate memory stores (`team-bots-and-sharing.md`): a shared **Team memory** visible to all collaborators, and private **Notes with each person** keyed to individual teammates.

### 8. Notable Design Insights
- **Shared Computer + Separate Minds**: Giving 50 Bots isolated memories and personas while mounting them onto the *same* persistent Linux VM (`/workspace` + shared browser profile) eliminates the friction of passing binary artifacts, auth cookies, and build trees across agents.
- **Differential Approval Timeouts for Bot-to-Bot Triggers**: Treating incoming bot-to-bot messages as background runs with a 10-minute approval TTL prevents deadlocks in multi-bot chains when the human is away.

---

## 2. OpenClaw

### 1. What It Is / OSS Status / Repo URL
- **Status**: Open-source (MIT License, ~392k GitHub stars).
- **Repo**: `https://github.com/openclaw/openclaw` (local clone: `/Users/jack/workspace/orchestrator-architecture/repos/openclaw`).
- **Scope**: Self-hosted, local-first multi-channel personal AI assistant and multi-agent gateway written in TypeScript/Node.js. Bridges 25+ messaging platforms (WhatsApp, Telegram, Slack, Discord, Signal, iMessage, Matrix, A2A) to persistent local agents and subagents.

### 2. Topology & Hierarchy
OpenClaw combines three distinct multi-agent topologies inside a single Gateway process:
1. **Multi-Agent Gateway Routing (Peer Personas)**: `openclaw.json` (`docs/concepts/multi-agent.md`) defines `agents.list`, where each entry is a fully isolated persistent agent with its own `workspace`, `agentDir` (`~/.openclaw/agents/<agentId>/`), and SQLite session database (`<agentDir>/agent/openclaw-agent.sqlite`). Inbound channel messages are routed deterministically via `bindings` using most-specific-wins precedence: `peer` (exact DM/channel ID) > `parentPeer` (thread parent) > `guildId` > `teamId` > `accountId` > `channel` > default agent.
2. **Hierarchical Subagent Tree (`sessions_spawn`)**: Any session can spawn background child sessions (`src/agents/tools/sessions-spawn-tool.ts`). Nesting is controlled by `agents.defaults.subagents.maxSpawnDepth` (default `1`; setting `maxSpawnDepth: 2` enables a **Main -> Orchestrator Subagent -> Worker Subagents** 3-tier hierarchy where depth-1 orchestrators gain `sessions_spawn`, `subagents`, `sessions_list`, and `sessions_history`). Fan-out is bounded by `maxChildrenPerAgent` (default `5`) and global `maxConcurrent` (default `8`).
3. **Flat Code-Mode Swarm (`agents.run`)**: Inside the JS/TS Code Mode sandbox (`docs/tools/swarm.md`), a parent can fan out parallel collector tasks via `await Promise.all([agents.run({ task }), ...])`. Swarm children run under session key `subagent:swarm:<schedulerGroupKey>`, cannot spawn further children (`maxSpawnDepth` is bypassed/blocked), and are throttled by `tools.swarm.maxConcurrent` (default `32`).

### 3. Information Passing & Context Handoff
- **`sessions_spawn` (`src/agents/tools/sessions-spawn-tool.ts`, `src/agents/subagents/spawn/subagent-spawn.ts`)**:
  - Parameters: `task`, `label`, `agentId` (checked against `subagents.allowAgents` allowlist), `model`, `thinking`, `runTimeoutSeconds`, `thread`, `mode` (`"run"` one-shot vs `"session"` persistent thread-bound), `cleanup` (`"keep" | "delete"`), `sandbox` (`"inherit" | "require"`), `worktree` (creates an isolated managed git checkout via `subagent-worktree.ts`), `visible` (creates a first-class dashboard session with owner attribution), `context` (`"isolated"` clean slate vs `"fork"` copying the parent's message history), `attachments` (base64 files staged into child workspace with SHA-256 verification), and `runtime` (`"subagent"` native vs `"acp"` external coding CLI harness).
  - Non-blocking start: immediately returns `{ status: "accepted", childSessionKey, runId }`.
  - **Announce Delivery (`src/agents/subagents/announce/subagent-announce-delivery.ts`)**: When a child finishes, OpenClaw formats its final output and status (`SUBAGENT_COMPLETION_OUTCOME_INSTRUCTION` in `src/agents/subagents/completion/subagent-completion-instructions.ts`) and injects an announcement turn back into the requester session, waking the parent agent.
- **`sessions_yield` (`src/agents/tools/sessions-yield-tool.ts`, `docs/concepts/subagent-yield-handoff.md`)**:
  - Allows an orchestrator that has spawned a batch of children to immediately end its current turn (`onYield`) and transfer its execution fence (`src/agents/subagents/registry/subagent-registry-requester-yield.ts`) to the active child batch. Inbound user messages stay queued until all children settle, at which point their combined results arrive as the parent's next turn.
- **`sessions_send` & A2A Ping-Pong Loop (`src/agents/tools/sessions-send-tool.ts`, `src/agents/tools/sessions-send-tool.a2a.ts`)**:
  - Enables direct cross-session and cross-agent messaging (governed by `tools.agentToAgent.enabled`, `tools.agentToAgent.allow`, and `tools.sessions.visibility`: `"self" | "tree" | "agent" | "all"`).
  - Supports fire-and-forget (`timeoutSeconds: 0`) or synchronous RPC (`waitForAgentRunReply` in `src/agents/run-wait.ts`).
  - After the initial reply, if `maxPingPongTurns > 0` (up to 5), `runSessionsSendA2AFlow` (`sessions-send-tool.a2a.ts`) runs an automated back-and-forth **ping-pong conversation loop** between requester and target session until either agent outputs the sentinel token `REPLY_SKIP` or reaches `maxPingPongTurns`, then announces the final transcript to the target channel.
- **External A2A Channel (`docs/channels/a2a.md`)**: Exposes `/.well-known/agent-card.json` and `/a2a/v1` (`SendMessage`, `GetTask`, `CancelTask`) with per-peer bearer tokens and isolated sessions keyed by `(peerName, contextId)`.

### 4. Agent Lifespan & Context Window Management
- **Persistent SQLite Sessions**: Every agent's main and channel sessions persist indefinitely in `<agentDir>/agent/openclaw-agent.sqlite`. Idle subagent sessions auto-archive after `archiveAfterMinutes` (default `60`).
- **Heartbeat Engine (`docs/gateway/heartbeat.md`)**: Periodic background wake timer (`agents.defaults.heartbeat.every`, default `"30m"`). Evaluates `HEARTBEAT.md`; if nothing needs attention, the agent replies `HEARTBEAT_OK` (suppressed from user channels). Supports `lightContext: true` (loads only `HEARTBEAT.md` instead of full bootstrap files) and `isolatedSession: true` (fresh session per check to save tokens). Also processes queued background task completion notifications with captured delivery routes.
- **Safeguard Compaction (`docs/concepts/compaction.md`)**: When a session nears its context window limit, OpenClaw triggers automatic pre-compaction memory flush (prompting the agent to save durable notes to disk) followed by `mode: "safeguard"` compaction. Safeguard compaction validates the generated summary against strict quality checks (required sections, preserved pending user asks, preserved exact identifiers/URLs/paths, CJK-aware token estimation), retrying with corrective feedback if validation fails and marking `details.qualityDegraded` on fallback.

### 5. Fast vs. Slow / Model Mixing & Cross-Provider Support
- **Model-Agnostic Routing & Cost Tiering**: Supports Anthropic, OpenAI, Google Gemini, xAI, OpenRouter, Bedrock, Ollama, and local vLLM.
- First-class cost optimization via `agents.defaults.subagents.model` (running cheap/fast worker models for `sessions_spawn` while keeping the main agent on a frontier model) and `agents.defaults.heartbeat.model` (running cheap models for 30-minute heartbeat ticks).
- **ACP Harness Mixing**: `sessions_spawn(runtime: "acp", agentId: "codex" | "claude" | "gemini")` dispatches tasks to external coding CLI harnesses via the Agent Client Protocol (`acpx`).

### 6. Human-in-the-Loop & Quality Gates
- **Exec Approval Policies & Tool Sandboxing**: Per-agent sandbox (`agents.list[].sandbox.mode`: `"off" | "non-main" | "all"` in Docker) and granular tool allow/deny lists (`tools.allow`, `tools.deny`). Sensitive shell commands emit interactive approval prompts to the active chat channel (Telegram/Slack/Discord/WebUI) or companion macOS/iOS/Android app.
- **Enterprise Delegate Architecture (`docs/concepts/delegate-architecture.md`)**: Supports running OpenClaw as a privileged organizational **Delegate** with its own service identity (M365/Google Workspace) across three trust tiers: Tier 1 (Read-Only + Draft), Tier 2 (Action-Scoped with Dual Control), and Tier 3 (Autonomous within policy boundaries), preventing confused-deputy attacks where a user attempts to exfiltrate another user's data through a shared bot.

### 7. Persistence, Memory & Learning Across Sessions
- **Bootstrap Markdown Workspace**: Injects `AGENTS.md`, `SOUL.md`, `IDENTITY.md`, `USER.md` (4,000-char budget), `TOOLS.md`, and `HEARTBEAT.md` at session start.
- **Two-Layer Markdown Memory + Vector/FTS Search (`docs/concepts/memory.md`)**:
  - `memory/YYYY-MM-DD.md`: Append-only daily logs.
  - `MEMORY.md`: Curated long-term memory (loaded only in private main sessions, never in group chats for privacy).
  - Indexed via hybrid vector + SQLite FTS search (`memory_search`, `memory_get`), plus optional QMD sidecar or **Memory Wiki** plugin (`docs/plugins/memory-wiki.md`, structured markdown vault with frontmatter claims and contradiction tracking).
- **Dreaming (`docs/concepts/dreaming.md`)**: Opt-in background memory consolidation (`plugins.entries.memory-core.config.dreaming`) that runs on a recurring sweep (default every 24h) across three phases: **Light sleep** (triage recent daily logs and session transcripts), **REM sleep** (extract themes and write narrative entries into `DREAMS.md`), and **Deep sleep** (score candidates across 6 weighted signals—frequency, relevance, query diversity, recency, consolidation, conceptual richness—before promoting winners into `MEMORY.md`).

### 8. Notable Design Insights
- **`sessions_yield` Execution Fence**: Solves a classic async subagent race condition where a parent spawns 3 subagents, ends its turn, and gets interrupted by a new user message before the children return. Transferring the execution fence to the child batch makes async fan-out behave deterministically like a single turn when desired.
- **Bounded A2A Ping-Pong (`REPLY_SKIP`)**: `sessions-send-tool.a2a.ts` allows two persistent sessions to negotiate a multi-turn clarification loop up to `maxPingPongTurns` without human intervention, terminating cleanly via an explicit sentinel token.

---

## 3. Hermes Agent

### 1. What It Is / OSS Status / Repo URL
- **Status**: Open-source (MIT License, ~254k GitHub stars), built by **Nous Research**.
- **Repo**: `https://github.com/NousResearch/hermes-agent` (local clone: `/Users/jack/workspace/orchestrator-architecture/repos/hermes-agent`).
- **Scope**: Self-improving persistent AI agent written in Python with a closed learning loop, multi-profile isolation, hierarchical subagent delegation (`delegate_task`), and a persistent SQLite **Kanban** multi-agent coordination board.

### 2. Topology & Hierarchy
Hermes provides two complementary multi-agent architectures (`website/docs/user-guide/features/kanban.md`):
1. **Hierarchical Fork-Join (`delegate_task` in `tools/delegate_tool.py`)**:
   - A parent `AIAgent` spawns 1 to $N$ isolated child `AIAgent` instances (either single-task or batch `tasks=[...]` run concurrently via `ThreadPoolExecutor`).
   - **Depth Limit**: Controlled by `delegation.max_spawn_depth` (`tools/delegate_tool_config.py`, `MAX_DEPTH = 2`). Depth `0` is the root agent; depth `1` children can spawn depth `2` grandchildren only when `max_spawn_depth >= 2`; depth `2` children have `delegate_task` stripped (`DELEGATE_BLOCKED_TOOLS`).
   - **Concurrency Cap**: `delegation.max_concurrent_children` defaults to `5` (`_DEFAULT_MAX_CONCURRENT_CHILDREN = 5` in `tools/delegate_tool_config.py:13`).
2. **Persistent Multi-Profile Kanban Board (`tools/kanban_tools.py`, `~/.hermes/kanban.db`)**:
   - Peer/hierarchical asynchronous coordination across named persistent **Profiles** (`~/.hermes/profiles/<name>/`), where each profile has its own `HERMES_HOME`, `config.yaml`, model/provider, `SOUL.md`, `MEMORY.md`, and skills.
   - A gateway-embedded background **Dispatcher** (`kanban.dispatch_enabled`) polls `~/.hermes/kanban.db` every `dispatch_poll_interval_seconds` (default `30s`) and spawns up to `max_concurrent_dispatches` (default `3`) separate OS processes (`hermes -p <assignee_profile>`), each running in an isolated `$HERMES_KANBAN_WORKSPACE`.

### 3. Information Passing & Context Handoff
- **`delegate_task` Context Isolation & Schema Validation (`tools/delegate_tool.py`, `tools/delegate_tool_child_run.py`, `tools/delegate_tool_progress.py`)**:
  - Each child receives **only** the `goal`, `context`, optional `images` (multimodal base64 content parts), `worktree=True` (isolated git worktree via `tools/worktree_manager.py`), and `output_schema` (JSON Schema enforced against the child's final output).
  - `_build_child_system_prompt` (`tools/delegate_tool_progress.py:178`) constructs a focused system prompt and loads workspace `AGENTS.md`/`CLAUDE.md` context files while deliberately skipping `SOUL.md`, memory, and parent chat history.
  - Child toolsets strip `DELEGATE_BLOCKED_TOOLS = frozenset(["delegate_task", "clarify", "memory", "send_message", "cronjob_manage"])` (`tools/delegate_tool_toolsets.py:16`).
  - **Async Background Delegation (`tools/async_delegation.py`)**: When `background=True`, `start_async_delegation()` spawns a `DaemonThreadPoolExecutor` worker, registers a live progress tracker, and immediately returns a `task_id`. Upon completion, it pushes a `type="async_delegation"` event onto `process_registry.completion_queue`. The CLI or messaging gateway drains `completion_queue` while idle so the child's result arrives as a **new user-turn notification** (`[Background task <id> completed]`), never mutating the parent's message history mid-turn.
- **Kanban State Machine & Handoff Metadata (`tools/kanban_tools.py`)**:
  - Tasks move through `backlog -> todo -> in_progress -> review -> done` (plus `blocked` and `archived`), guarded by `_VALID_TRANSITIONS` (`tools/kanban_tools.py`).
  - Dispatched worker processes receive `KANBAN_GUIDANCE` in their system prompt and call `kanban_show(task_id)` to read the task description, parent task outputs, prior attempt histories, and the inter-agent comment thread (`kanban_comment`).
  - Workers finish by calling `kanban_complete(task_id, summary="...", metadata={...})` (which auto-unblocks downstream tasks with `depends_on` edges), `kanban_request_review`, or `kanban_block`.

### 4. Agent Lifespan & Context Window Management
- **Child Liveness & Wedged-Child Detection (`_Heartbeat` in `tools/delegate_tool_child_run.py:267`)**: Every child `AIAgent` is wrapped in a `_Heartbeat` monitor that tracks `(tool_name, iteration, activity_ts)` and registers with `agent.periodic_scheduler`. If a child stalls without tool progress, the parent surfaces liveness warnings or interrupts the tree (propagating SIGINT downward).
- **Kanban Stale Reclaim & 90% Budget Warning**: Workers send periodic `kanban_heartbeat(task_id, note="...")` updates. If a worker is `in_progress` longer than `dispatch_stale_timeout_seconds` (default `14400s` / 4h) with no heartbeat in the last hour, the dispatcher reclaims the task and returns it to `todo` (up to `max_attempts`, default `3`). Additionally, when a worker reaches **90% of its `max_iterations` budget** (`agent.budget_warning_ratio`), Hermes injects a checkpoint warning prompting it to post a `kanban_comment` with findings before exit.
- **Context Compression (`agent/context_compressor.py` & `agent/micro_compaction.py`)**: Multi-stage context management combining lightweight micro-compaction of verbose old tool outputs (`agent/micro_compaction.py`) with full LLM-driven trajectory summarization (`agent/context_compressor.py`).

### 5. Fast vs. Slow / Model Mixing & Cross-Provider Support
- **Per-Task & Per-Profile Model Routing**: Both `delegate_task(model=..., provider=...)` and Kanban profiles allow mixing a frontier orchestrator (e.g., Claude Opus 4.6 / GPT-5.4) with inexpensive high-speed worker models (e.g., DeepSeek V3.2, Qwen, MiniMax, or local Ollama/vLLM) across 20+ providers.
- **Native Mixture-of-Agents (`agent/moa_loop.py`)**: Includes a built-in `/moa` mode that queries a parallel panel of frontier reference models before each agent turn and aggregates their reasoning.

### 6. Human-in-the-Loop & Quality Gates
- **Subagent Clarification Restriction**: `clarify` and `send_message` are explicitly blocked in subagents (`DELEGATE_BLOCKED_TOOLS`) so only the root agent can prompt the human user.
- **Kanban Review Column**: Workers uncertain of destructive or architectural changes call `kanban_request_review(task_id, note="...")`, moving the card to the `review` column and notifying the human or orchestrator profile to inspect the branch/PR before marking `done`.
- **Dangerous Command Approval (`tools/approval.py`)**: Pattern-matched shell commands require explicit user confirmation across CLI or messaging gateways.

### 7. Persistence, Memory & Learning Across Sessions
- **Frozen-Snapshot Bounded Memory (`tools/memory_tool.py`)**: Two curated markdown files (`~/.hermes/memories/MEMORY.md` and `USER.md`) are injected into the system prompt as a **frozen snapshot at session start** (capped at a strict character budget, default ~2,200 chars). Mid-session edits via `memory` update the files on disk immediately without mutating the active system prompt prefix, preserving 100% LLM KV-cache hit rates across the turn loop.
- **Cross-Session FTS5 Search (`hermes_state_fts.py`)**: Full transcripts are persisted in SQLite (`~/.hermes/state.db`) and searchable via `session_search` (FTS5 keyword search + LLM summarization), plus optional **Honcho** dialectic user-modeling integration.
- **Prompt-Cache-Sharing Background Review (`agent/background_review.py`)**: After every completed conversation turn, `spawn_background_review()` forks the `AIAgent` in a daemon thread with the exact same prompt prefix (reusing the provider's prompt KV-cache) and restricts tools to `memory` and `skill_manage` so the agent autonomously extracts new procedural **Skills** (`~/.hermes/skills/<category>/<name>/SKILL.md`) or updates memory with near-zero latency/cost overhead.
- **Autonomous Skill Curator (`agent/curator.py`, `website/docs/user-guide/features/curator.md`)**: Triggered on an inactivity check (`curator.interval_hours` default 168h / 7 days, `min_idle_hours` default 2h), the Curator audits `~/.hermes/skills/`, fixes malformed YAML frontmatter, merges duplicate agent-created skills, and archives unused skills (`~/.hermes/skills/.archive/`) while never touching user-installed skills or auto-deleting.

### 8. Notable Design Insights
- **Zero-Cost Post-Turn Self-Improvement via KV-Cache Forking**: `agent/background_review.py` solves the latency and cost problem of post-turn reflection by forking the exact conversation prefix immediately after the turn finishes—hitting the LLM provider's warm prompt cache so background skill/memory extraction only pays for a few completion tokens.
- **Frozen System-Prompt Memory Snapshot**: Decoupling on-disk `MEMORY.md` mutations from the in-context system prompt snapshot until the next session prevents memory edits from invalidating the KV-cache mid-session.

---

## 4. Manus & OpenManus

### 1. What It Is / OSS Status / Repo URL
- **Status**: **Manus** (`https://manus.im`) is a closed-source commercial cloud general agent platform (Manus 1.0 -> Wide Research -> Manus 2.0 with **Cascade** harness and **Cloud Computer**). **OpenManus** (`https://github.com/FoundationAgents/OpenManus`, ~58.6k stars, MIT License; local clone: `/Users/jack/workspace/orchestrator-architecture/repos/OpenManus`) is the leading open-source community reproduction built by MetaGPT contributors.

### 2. Topology & Hierarchy
- **Manus Wide Research (2-Tier MapReduce Cloud Swarm)**:
  - Introduced in `https://manus.im/docs/features/wide-research` to solve context degradation on high-cardinality tasks (e.g., analyzing 100 Fortune 500 companies, comparing 50 MBA programs, or generating 100 assets).
  - A **Main Orchestrator Agent** decomposes the user's prompt into $N$ homogeneous or heterogeneous atomic sub-tasks (up to hundreds of items in parallel) and spawns **hundreds of collaborative general-purpose Manus subagents**.
  - Crucially, unlike role-constrained multi-agent frameworks where subagents are narrow role personas, **every Wide Research subagent is a full general-purpose Manus instance** running in its own dedicated cloud VM sandbox with an isolated context window, browser, and shell.
- **Manus 2.0 Cascade Harness**: Couples an interactive main agent with persistent Cloud Computer state and event-driven background **Automations** (triggers on Gmail, Slack, Notion, Linear, GitHub, and cron schedules).
- **OpenManus `PlanningFlow` (`repos/OpenManus/app/flow/planning.py`)**:
  - Implements an explicit **Planner -> Typed Step Executor** loop (`PlanningFlow(BaseFlow)`).
  - The orchestrator calls `PlanningTool` (`app/tool/planning.py`) to create a numbered step plan (`[ ]`, `[→]`, `[✓]`, `[!]`), extracts `step_type` tags (e.g., `[SEARCH]`, `[CODE]`, `[BROWSER]`), and dispatches each step via `self.get_executor(step_type)` (`app/flow/planning.py:86-98`) to specialized `BaseAgent` instances (`Manus`, `SWEAgent`, `BrowserAgent`, `DataAnalysis` in `app/agent/`).

### 3. Information Passing & Context Handoff
- **Filesystem-Mediated Handoffs & Structured Aggregation**: In Manus Wide Research, the Main Agent dispatches self-contained task specifications and output schemas to each cloud subagent. Subagents write structured artifacts (JSON/CSV/Markdown files) into shared cloud storage; the Main Agent collects those artifacts and synthesizes them into unified deliverables (comparison tables, spreadsheets, reports, or zip archives).
- **OpenManus Step Execution (`app/flow/planning.py:213-244`)**: `PlanningFlow._execute_step(executor, step_info)` injects the current `plan_status` plus the current `step_text` into `executor.run(step_prompt)`, then marks the step `completed` in `PlanningTool`.

### 4. Agent Lifespan & Context Window Management
Manus Chief Scientist Peak Ji published the definitive architectural breakdown of Manus's context management in *"Context Engineering for AI Agents: Lessons from Building Manus"* (`https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus`). Since an average Manus task runs ~50 tool calls with a **100:1 input-to-output token ratio**, Manus is engineered around six rules:
1. **Design Around the KV-Cache**: Never place dynamic timestamps or non-deterministic JSON key ordering in the system prompt prefix; use append-only action-observation history with explicit cache breakpoints.
2. **Mask, Don't Remove Tools**: Adding or removing tool definitions mid-session invalidates the entire KV-cache and confuses the model when prior tool calls reference vanished tools. Instead, Manus keeps a static tool catalog and enforces a state machine by **masking token logits / response prefill** using consistent tool-name prefixes (`browser_*`, `shell_*`, `file_*`).
3. **Use the File System as Ultimate Context (Restorable Compression)**: Instead of irreversibly summarizing long web pages or PDFs, Manus writes raw observations to the cloud sandbox filesystem and drops the raw text from context while **retaining the file path or URL** so the agent can always re-read the exact lines if needed.
4. **Manipulate Attention Through Recitation (`todo.md`)**: Over 50+ tool calls, models suffer from "lost-in-the-middle" goal drift. Manus agents continuously create and rewrite `todo.md` at the end of steps, reciting the global plan into the *tail* of the context window where causal self-attention is strongest.
5. **Keep the Wrong Stuff In**: Never scrub failed tool calls or stack traces from the active trajectory; seeing the failed action and error trace prevents the model from repeating the same mistake.
6. **Don't Get Few-Shotted**: In repetitive batch tasks, uniform action-observation formatting lulls the model into brittle hallucinated patterns; Manus introduces controlled structural variation in observations to maintain attention.

### 5. Fast vs. Slow / Model Mixing & Cross-Provider Support
- Manus uses a multi-model ensemble routing tasks across Claude Sonnet/Opus, Gemini, and specialized models depending on tool modality (coding vs. browser vision vs. wide synthesis). OpenManus supports any OpenAI-compatible, Anthropic, Azure, Ollama, or Bedrock endpoint via `config/config.toml` with optional per-agent LLM overrides.

### 6. Human-in-the-Loop & Quality Gates
- **Cloud Computer Takeover**: Like Grok Bot and Devin, Manus provides a live side-panel view of the cloud sandbox where the user can take control of the browser for authentication/2FA/payments and return control to the agent.

### 7. Persistence, Memory & Learning Across Sessions
- **Manus 2.0 Cloud Computer**: Upgrades sandboxes from ephemeral per-session containers to a persistent **Cloud Computer** where files, installed tools, and project structures survive across sessions and background Automations.

### 8. Notable Design Insights
- **General-Purpose Subagents Beat Role-Play Personas for Scale**: Manus Wide Research explicitly rejects static "PM / Coder / QA" role-play graphs in favor of spawning $N$ identical general-purpose Manus agents in isolated VMs—treating multi-agent orchestration as distributed MapReduce over context windows rather than theatrical role-play.
- **Recitation (`todo.md`) as Attention Control**: Using a file write (`todo.md`) not merely for persistence, but specifically to push global state into the recency window of causal attention, is now widely copied across coding harnesses.

---

## 5. Devin (Cognition)

### 1. What It Is / OSS Status / Repo URL
- **Status**: Closed-source autonomous software engineering platform by **Cognition** (`https://devin.ai`, docs at `https://docs.devin.ai`; open-source handoff plugin at `https://github.com/club-cog/devin-handoff`).
- **Scope**: Cloud-hosted autonomous SWE running in isolated cloud VMs (shell, IDE, browser), paired with Windsurf IDE, DeepWiki, Devin Review, SWE-grep Fast Context subagents, and **Managed Devins** (parent-child multi-VM orchestration).

### 2. Topology & Hierarchy
Cognition's multi-agent architecture evolved significantly between June 2025 and April 2026, documented in two landmark engineering essays by co-founder Walden Yan:
1. **Phase 1 — The Anti-Swarm Critique (*"Don't Build Multi-Agents"*, June 12, 2025, `cognition.ai/blog/dont-build-multi-agents`)**:
   - Formulated two foundational principles of multi-agent systems:
     - **Principle 1: Share context** (passing brief task strings or lossy summaries between agents creates a "telephone game"; subagents need full decision traces).
     - **Principle 2: Actions carry implicit decisions** (when two subagents write code in parallel, each makes dozens of implicit assumptions about naming, error handling, and architecture that conflict upon merge).
   - Concluded in 2025 that **parallel writers are an anti-pattern**, advocating a single linear thread of execution supplemented only by read-only subagents.
2. **Phase 2 — Production Multi-Agent Topology (*"Multi-Agents: What's Actually Working"*, April 9, 2026, `cognition.ai/blog/multi-agents-working` & Managed Devins launch, March 2026)**:
   - Driven by 8x enterprise usage growth and stronger agentic models, Cognition introduced **three production multi-agent topologies** that respect the two principles ("distributed intelligence, single-threaded writes per boundary"):
     - **Managed Devins (Parent -> Child Cloud VMs)**: A coordinator **Parent Devin** decomposes cross-repo or multi-PR epics, spins up **multiple Child Devins** (each in its own isolated cloud VM with its own terminal, browser, playbook, tags, and ACU budget limit), monitors them via the UI **Agents** tab, messages them mid-task, and **auto-wakes** when children finish or request input.
     - **Adversarial Producer-Reviewer Loop (`Devin <-> Devin Review`)**: A coding Devin opens a PR and hands off to **Devin Review**, a separate review agent that inspects the diff in an independent context, posts inline bug findings (catching an average of 2 bugs per PR, 58% severe), and loops back to the coding Devin until clean.
     - **Read-Only Fast Context Subagents (`SWE-grep` & `DeepWiki`)**: Inside a single Devin or Windsurf session, read-only retrieval is delegated to **`SWE-grep`** / **`SWE-grep-mini`** (`cognition.ai/blog/swe-grep`), custom RL-trained subagents that execute **8 parallel tool calls per turn across a strict 4-turn cap** at >2,800 tokens/sec to return exact file/line context to the main model in seconds without polluting its context window.

### 3. Information Passing & Context Handoff
- **Managed Devins Inter-Session Messaging**: The Parent Devin passes structured task briefs, repository scopes, playbook IDs, and ACU limits when spawning Child Devins, and exchanges bidirectional messages with children mid-flight. When a child completes its PR or blocks on a question, an event wakes the sleeping Parent Devin session.
- **Local-to-Cloud Handoff (`devin-handoff` / `/handoff`)**: Developers working locally in Devin CLI or Claude Code run `/handoff` (`club-cog/devin-handoff`), which packages the local git state, branch diff, and summarized conversation trace and provisions a remote Cloud Devin VM via the Devin v3 API to finish the task asynchronously.

### 4. Agent Lifespan & Context Window Management
- **Stateful Cloud VM Snapshots**: Devin sessions run inside persistent cloud VMs that automatically snapshot and sleep when idle, resuming instantaneously when a webhook (GitHub PR comment, CI failure, Slack mention, or Child Devin completion) fires.
- **Context Isolation via `SWE-grep`**: By offloading all exploratory `grep`/`find`/`read` calls to the 4-turn `SWE-grep` subagent, the main coding agent never ingests thousands of lines of irrelevant search hits into its primary context window.

### 5. Fast vs. Slow / Model Mixing & Cross-Provider Support
- **Extreme Speed-vs-Depth Separation (`SWE-grep` + Frontier Reasoner)**: Cognition explicitly pairs custom high-speed RL models (`SWE-grep` / `SWE-grep-mini` running on Cerebras/fast inference hardware at >2,800 tok/s) for context retrieval with heavy frontier reasoning models for planning and code edits.

### 6. Human-in-the-Loop & Quality Gates
- **Interactive Plan Approval & Confidence Scoring**: Devin assesses upfront task ambiguity, proposes a step-by-step plan for user sign-off, and gates PR readiness through **Devin Review** and automated CI verification before requesting human merge.
- **Per-Child ACU Guardrails**: In Managed Devins, the parent session sets explicit **Agent Compute Unit (ACU) limits** per child VM so a runaway child subagent cannot exhaust organizational compute budgets.

### 7. Persistence, Memory & Learning Across Sessions
- **Knowledge & Playbooks**:
  - **Knowledge**: Repository- and organization-scoped persistent memory items (coding conventions, deployment quirks, architecture notes) that Devin automatically retrieves when relevant and proposes updating when a human corrects it in a PR or Slack thread.
  - **Playbooks**: Reusable macro templates (`.devin.md`) defining repeatable multi-step workflows (e.g., migrations, vulnerability patching) passed to Child Devins.
  - **DeepWiki**: Continuously indexed architectural knowledge graph and wiki generated across the organization's repositories.

### 8. Notable Design Insights
- **Distributed Intelligence, Single-Threaded Writes**: Cognition's progression from *"Don't Build Multi-Agents"* (2025) to *Managed Devins + SWE-grep + Devin Review* (2026) crystallizes a core architectural law: **parallelize read-only retrieval (`SWE-grep`), adversarial verification (`Devin Review`), and orthogonal repo/PR units (`Managed Devins`), but never let multiple agents concurrently mutate the same coupled code state without a single-writer boundary**.

---

## 6. Cursor Cloud/Background Agents & Multi-Agent

### 1. What It Is / OSS Status / Repo URL
- **Status**: Closed-source commercial AI IDE and cloud agent platform by Anysphere (`https://cursor.com`, docs at `https://cursor.com/docs/subagents.md` and `https://cursor.com/docs/cloud-agent`).
- **Scope**: Includes **Cursor 2.0/3.0 Agents Window** (parallel local worktree and remote cloud agents), custom Markdown **Subagents** (`.cursor/agents/*.md`), and Cursor's **Long-Running Autonomous Coding Harness** (`https://cursor.com/blog/scaling-agents`, Jan 2026).

### 2. Topology & Hierarchy
Cursor operates three distinct multi-agent layers:
1. **Parallel Worktree & Cloud Agents (Up to 8 Parallel Runs)**:
   - From the **Agents Window** (Cursor 2.0/3.0), a user can launch up to **8 parallel agents** on a single prompt (including `/best-of-n` across different models or independent feature tasks).
   - Local parallel runs are isolated in separate **Git worktrees** configured via `.cursor/worktrees.json` (`setup-worktree-unix`, `setup-worktree-windows`), while remote runs execute in isolated **Cloud VMs** configured via `.cursor/environment.json`.
2. **Hierarchical Subagents (`.cursor/agents/*.md`, 2-Level Nesting)**:
   - Built-in subagents (`Explore` for 10x parallel codebase search, `Browser` via MCP to filter noisy DOM snapshots, `bash`) plus custom Markdown subagents in `.cursor/agents/*.md` (also compatible with `.claude/agents/` and `.codex/agents/`).
   - **Depth Limit**: Since Cursor 2.5 (`subagents.md`), Cursor supports **2-level subagent nesting**: `Main Agent -> Direct Subagent (Depth 1) -> Child Subagent (Depth 2)`. Depth-2 subagents are blocked from spawning further subagents to prevent runaway recursion.
3. **Long-Running Autonomous Coding Harness (`Planner -> Worker -> Judge`, `cursor.com/blog/scaling-agents`)**:
   - Built to run **hundreds of concurrent agents** continuously for weeks on a single codebase (demonstrated by building a >1M LoC web browser from scratch in Rust, a Windows 7 emulator, and a Solid-to-React migration).
   - **Why Peer Topologies Failed**: Cursor first tested a flat peer-to-peer architecture where agents coordinated via a shared state file and locks; **20 agents slowed down to the effective throughput of 2–3 agents** due to lock contention, brittle deadlocks, and risk-averse behavior (agents avoiding hard tasks).
   - **Winning Hierarchy**:
     - **Planners (Recursive)**: Continuously explore the codebase and spawn **Sub-Planners** for specific subsystems, maintaining a dynamic task queue.
     - **Workers (Isolated)**: Pick up a single atomic task, execute until completion in an isolated worktree without talking to other workers, and push directly to the target branch (Cursor found that **removing a dedicated Integrator/Merge agent** and letting workers handle rebase/merge directly eliminated a major serialization bottleneck).
     - **Judge Agent**: Evaluates the state of the codebase at the end of each cycle and decides whether the overarching goal is met or another planning iteration is required.

### 3. Information Passing & Context Handoff
- **Subagent Handoff (`subagents.md`)**: Subagents receive a clean context window seeded with the system prompt from `.cursor/agents/<name>.md` (YAML frontmatter: `name`, `description`, `model`, `readonly`, `is_background`) and the parent's explicit task prompt, returning only a distilled final summary to the parent.
- **Local-to-Cloud Handoff (`/in-cloud` & `/autopilot`)**: Users or parent agents can hand off a local conversation or background subagent to a remote Cursor Cloud VM via `/in-cloud` or `/autopilot`, transferring the branch state and conversation summary so execution continues when the laptop closes.

### 4. Agent Lifespan & Context Window Management
- **Short Worker Cycles vs. Weeks-Long Harness**: In Cursor's scaling research (`scaling-agents`), individual Worker runs are intentionally kept bounded to a single task with fresh context windows; Cursor notes that **periodic fresh starts** are essential to combat context drift and tunnel vision in multi-day runs.
- **Resumable Subagents**: Every subagent execution returns an `agentId` that can be resumed in later turns with its full previous transcript preserved.

### 5. Fast vs. Slow / Model Mixing & Cross-Provider Support
- **Composer 2 / 2.5 + Frontier Planners**: Cursor mixes its custom RL-trained high-speed coding models (**Composer / Composer 2 / Composer 2.5**, completing most interactive coding turns in <30 seconds at ~4x the speed of comparable frontier models) and fast `Explore` subagent models (`model: "fast"`) with deep reasoning planners (`gpt-5.2`, `claude-opus-5`).
- In the long-running harness experiments, Cursor explicitly noted that **GPT-5.2 dramatically outperformed Claude Opus 4.5 for long-horizon autonomous planning** because Opus tended to stop early and take shortcuts, whereas mixing specialized models per role (`Planner` vs `Worker`) yielded the best throughput.

### 6. Human-in-the-Loop & Quality Gates
- **Readonly & Background Flags**: Subagents can be locked to `readonly: true` (removing file-edit and mutating tools) and `is_background: true`.
- **Self-Contained Verification Artifacts**: Cloud agents run automated background **Builds**, record video/screenshots of UI changes, and package diffs into reviewable PRs or multi-diff comparison views in the Agents Window.

### 7. Persistence, Memory & Learning Across Sessions
- **Rules (`.cursor/rules/*.mdc`), `AGENTS.md`, and Cloud Environment Snapshots**: Persistent project knowledge lives in versioned `.cursor/rules/` and `AGENTS.md` files alongside pre-warmed cloud VM snapshots (`.cursor/environment.json`).

### 8. Notable Design Insights
- **Prompts Matter More Than Coordination Primitives at Scale**: In `cursor.com/blog/scaling-agents`, Cursor reported that once they replaced flat peer locks with `Planner -> Worker -> Judge` and removed the dedicated Integrator bottleneck, the hardest remaining problems were behavioral (preventing premature completion and scope creep via prompt engineering and fresh worker resets).

---

## 7. GitHub Copilot Coding Agent / Agent HQ / Mission Control

### 1. What It Is / OSS Status / Repo URL
- **Status**: Closed-source commercial cloud platform by GitHub / Microsoft (`https://github.blog/news-insights/company-news/welcome-home-agents/`, Oct 2025; docs at `https://docs.github.com/en/copilot/concepts/agents/cloud-agent`).
- **Scope**: **GitHub Copilot cloud agent** (formerly coding agent), **Agent HQ** (multi-vendor persistent agent ecosystem), and **Mission Control** (unified multi-agent command center across GitHub.com, VS Code, Copilot CLI, and GitHub Mobile).

### 2. Topology & Hierarchy
- **Mission Control + Multi-Vendor Partner Roster (Agent HQ)**:
  - Rather than locking users into a single proprietary agent, **Agent HQ** turns GitHub into an open multi-agent control plane where developers assign issues, PR reviews, or background tasks to **GitHub Copilot**, **Anthropic Claude**, **OpenAI Codex**, **Google Jules**, **Cognition Devin**, or **xAI Grok** side-by-side (`welcome-home-agents`).
  - Users can dispatch the same problem or parallel sub-issues to multiple agents simultaneously from **Mission Control** (`github.com/copilot/agents`).
- **Custom Specialist Personas (`.github/agents/*.md`)**:
  - Repositories define custom specialist agents in `.github/agents/<name>.md` (or organization-wide `.github-private/agents/`), specifying custom system prompts, restricted toolsets, and MCP servers (e.g., a frontend specialist, security auditor, or documentation writer).
- **Copilot Code Review Agent Integration**: When the Copilot coding agent finishes a task on a draft PR, it can automatically invoke the **Copilot code review agent** (or a partner agent) as a verification peer before requesting human review.

### 3. Information Passing & Context Handoff
- **PR- and Issue-Centric State Handoff**:
  - Tasks can be delegated from GitHub Issues (assigning the issue to `@copilot` or a partner agent), PR review comments, GitHub Actions/Security Campaigns, VS Code Copilot Chat, or Copilot CLI (`remote control`).
  - When handing off from interactive **Copilot Chat** or **VS Code** to the cloud agent, the full local chat transcript and selected file references are passed as initial context to the cloud session.
  - The cloud agent immediately opens a **Draft Pull Request** on a restricted `copilot/*` branch; the PR description, commit stream, and session log serve as the shared state bus between the agent, CI workflows, review agents, and human developers.
- **Live Asynchronous Steering**: Humans (or automated workflows) steer a running or idle cloud agent by leaving `@copilot` comments on the Draft PR or issuing steering prompts directly inside Mission Control.

### 4. Agent Lifespan & Context Window Management
- **Ephemeral GitHub Actions VMs with Persistent Session State**:
  - Each cloud agent run boots an **isolated ephemeral Linux VM powered by GitHub Actions**, pre-configured via `.github/workflows/copilot-setup-steps.yml` (allowing custom compilers, Docker services, and cached dependencies before the firewall locks down).
  - While the underlying Actions VM terminates after the session turn completes (or hits the workflow timeout), the **agent session state and branch** persist—any subsequent PR comment rehydrates a fresh VM against the branch and resumes the session transcript.
- **Scheduled Automations & GitHub Agentic Workflows**: Supports natural-language **Automations** (cron-scheduled or webhook-triggered cloud agent runs) and Markdown-defined **GitHub Agentic Workflows** compiled into Actions YAML.

### 5. Fast vs. Slow / Model Mixing & Cross-Provider Support
- **First-Class Multi-Vendor Model & Agent Selection**: Users select both the **agent harness** (Copilot, Claude, Codex, Jules, Devin, Grok) and the **underlying model** (Claude Opus/Sonnet, GPT-5.x, Gemini, Grok) per task in Mission Control or VS Code.

### 6. Human-in-the-Loop & Quality Gates
- **Strict Governance & Branch Isolation**:
  - The Copilot cloud agent can **only** create and push to branches prefixed with `copilot/`.
  - **Segregation of Duties**: The human who asked `@copilot` to open a PR is **blocked from approving that PR** if branch protection requires approving reviews, ensuring independent human sign-off.
  - **CI Gatekeeping**: GitHub Actions workflows triggered by the agent's pushes require explicit approval or policy configuration, and network egress is filtered by a configurable allowlist firewall.

### 7. Persistence, Memory & Learning Across Sessions
- **Copilot Memory**: Persists learned repository facts, build commands, and user preferences across sessions so the cloud agent remembers past corrections without manual `.github/copilot-instructions.md` edits.
- **Copilot Spaces**: Curated collections of repositories, docs, PRs, and free-text notes that can be attached to any agent session as a shared knowledge base.
- **Cross-Surface Synced Session History**: Local CLI/VS Code sessions can be synced to GitHub (`session-sync-level`) and inspected or resumed from the web or mobile app.

### 8. Notable Design Insights
- **The Pull Request as the Universal Multi-Agent State Machine**: By binding every background agent session to a `copilot/*` branch and Draft PR on GitHub Actions infrastructure, GitHub reuses existing enterprise primitives (branch protection, Actions CI, CODEOWNERS, PR review threads, audit logs) as the multi-agent coordination and security layer.

---

## 8. Google Jules

### 1. What It Is / OSS Status / Repo URL
- **Status**: Closed-source commercial asynchronous cloud coding agent by Google Labs (`https://jules.google`, docs at `https://jules.google/docs`, CLI package `@google/jules`, REST API at `https://jules.googleapis.com`).
- **Scope**: Asynchronous cloud VM coding agent integrated with GitHub, programmable via the **Jules Tools CLI (`jules`)**, REST API, MCP server, and GitHub Agent HQ.

### 2. Topology & Hierarchy
- **Session Orchestrator + Internal Critic Agents**:
  - Each user request or API call creates a stateful **Session** bound to a GitHub **Source** (repository + branch) inside an isolated **Google Cloud VM**.
  - Inside a Session, Jules operates an internal **Producer -> Critic** hierarchy:
    1. **Planner + Planning Critic (launched Jan 2026)**: Generates a structured multi-step implementation plan; when auto-approval is enabled, an internal **Planning Critic** adversarially reviews the proposed plan for completeness, edge cases, and adherence to repo constraints before execution begins.
    2. **Executor + Critic Agent (launched Aug/Sep 2025)**: As the coding agent edits files and runs tests in the Cloud VM, a separate **Critic Agent** critiques the generated code diff and test execution output—evaluating logic bugs, security pitfalls, and subtle inefficiencies—and forces iterative refinement before Jules commits or opens a PR.
- **External Orchestration via Jules API (`jules.googleapis.com`)**: External orchestrators (such as OpenClaw, Gemini CLI, or custom CI pipelines) can programmatically fan out up to **60–100 concurrent Jules Sessions** (Pro/Ultra tiers) via `POST /v1alpha/sessions` (`sessions.create`, `sessions.sendMessage`, `sessions.approvePlan`).

### 3. Information Passing & Context Handoff
- **Session Activities & Interactive Plan Artifact**:
  - Communication within a Jules Session is modeled as a stream of typed **Activities** (`PlanGenerated`, `PlanApproved`, `ProgressUpdated`, `AgentMessaged`, `UserMessaged`, `SessionCompleted`).
  - The Critic Agent receives the task specification, the generated code diff, and VM test results in a critique turn and returns structured feedback directly to the coding agent inside the session loop.
- **CLI & Local Diff Pulling (`@google/jules`)**: Developers can launch remote sessions from the terminal (`jules remote new --repo . --session "..."`) and pull the resulting cloud diff directly into their local working tree (`jules remote pull --session <id>`).

### 4. Agent Lifespan & Context Window Management
- **Persistent Cloud VMs with Environment Snapshots**:
  - Jules clones the repository into a dedicated Google Cloud VM, runs user-defined setup scripts, and saves **Environment Snapshots** so subsequent sessions boot with dependencies pre-installed.
  - Sessions are asynchronous and resumable via `sendMessage` or GitHub PR comments (`@jules`).

### 5. Fast vs. Slow / Model Mixing & Cross-Provider Support
- **Gemini 3 Flash / Gemini 3 Pro Routing**: In January 2026, Jules transitioned its default base model to **Gemini 3 Flash** (delivering frontier coding reasoning at flash-tier speed) while offering **Gemini 3 Pro / 3.1** for complex architectural tasks.

### 6. Human-in-the-Loop & Quality Gates
- **Explicit Plan Approval Gate**: By default, Jules pauses after generating its **Interactive Plan** (`requirePlanApproval: true`), allowing the human to edit steps, answer clarifying questions, or chat with Jules before any code is modified.
- **Dual Internal Critics + Visual Verification**: Combines the **Planning Critic** and code **Critic Agent** with live web-app rendering and visual verification inside the cloud VM.

### 7. Persistence, Memory & Learning Across Sessions
- **Jules Memory (launched Sept 30, 2025)**: An automatic, repository-scoped persistent memory module (`jules.google/docs/memory`). As a developer nudges, corrects, or steers Jules during a session, Jules automatically extracts durable repository preferences (e.g., test runner flags, architectural rules, style preferences) into a per-repo memory store and injects them into future sessions on that repository, alongside explicit `AGENTS.md` files.

### 8. Notable Design Insights
- **Built-In Dual Critics (Plan Critic + Diff Critic)**: Rather than relying solely on unit tests (which an agent can accidentally weaken or mock), Jules embeds adversarial critique at *both* the plan boundary and the post-diff boundary inside the cloud VM before human review.

---

## 9. Google Antigravity Agent Manager

### 1. What It Is / OSS Status / Repo URL
- **Status**: Closed-source commercial agentic development platform and standalone desktop application by Google DeepMind (`https://antigravity.google`, docs at `https://antigravity.google/docs/subagents.md`, `teamwork.md`, `boost.md`).
- **Scope**: **Google Antigravity 2.0** (standalone cross-platform Agent Manager desktop app) and **Antigravity CLI (`agy`)**, featuring native peer/hierarchical subagent messaging (`invoke_subagent` / `send_message`), 3-tier **Boost (`/boost`)**, and multi-day autonomous **Teamwork (`/teamwork-preview`)**.

### 2. Topology & Hierarchy
Antigravity supports three distinct orchestration topologies:
1. **Dynamic Subagent Graph (`invoke_subagent`, `define_subagent`, `send_message`, `manage_subagents`)**:
   - Any agent can dynamically define new specialist agent types (`define_subagent`) or invoke built-in/custom subagents (`research`, `browser`, `self`, `image-generator`, or Markdown agents in `.agents/agents/<name>.md` and `~/.gemini/config/agents/<name>.md`).
   - **Depth Limit**: Enforces a hard **10-level nesting depth limit** (`antigravity.google/docs/subagents.md`), supporting deep recursive task decomposition while preventing infinite spawn loops.
   - **Communication Topology**: Both hierarchical (parent <-> child) and **peer-to-peer**—any agent holding another agent's `conversationId` can message it directly via `send_message(Recipient=<conversationId>, Message=...)`.
2. **Boost (`/boost`) — 3-Tier Deep Reasoning Hierarchy (`boost.md`)**:
   - Designed for complex tasks running from seconds to several hours:
     - **Tier 1 (`Orchestrator`)**: Top-level strategist that decomposes the problem and manages user alignment.
     - **Tier 2 (`DeepCoder` & `DeepInvestigator`)**: Mid-level domain coordinators that plan implementation vs. deep root-cause research.
     - **Tier 3 (Execution Workers)**: Parallel workers executing in **isolated Git worktrees** (`Workspace: "branch"`), verified before merge.
3. **Teamwork (`/teamwork-preview`) — Multi-Day Autonomous Agent Team (`teamwork.md`)**:
   - Designed for large-scale projects running over hours or days with specialized roles and **Adversarial Verification Gates**:
     - **Sentinel**: Top-level interface and lifecycle supervisor.
     - **Project Orchestrator**: Decomposes the project into milestones, assigns **exclusive file ownership** per Worker to prevent merge conflicts, and **hands off to a fresh successor Project Orchestrator between milestones** so orchestrator context never bloats.
     - **Explorers** (read-only research) & **Workers** (scoped implementation).
     - **Four Adversarial Verification Agents**: **Critic** (design/code review), **Challenger** (writes adversarial/worst-case tests designed to break the implementation), **Auditor** (verifies actual command outputs against the selected Integrity Mode), and **Success Auditor** (final end-to-end readiness gate before Sentinel reports completion).

### 3. Information Passing & Context Handoff
- **Explicit `send_message` Requirement & Idle Auto-Wake**:
  - Every invoked subagent gets a unique `conversationId` and an isolated context window.
  - Subagent text output is *not* automatically dumped into the caller's context; the subagent must explicitly call `send_message(Recipient=<callerId>, Message=...)` to return distilled findings or file paths.
  - **Reactive Auto-Wake**: Subagents have three lifecycle states (`Running`, `Idle`, `Killed`). When a subagent finishes its turn, it transitions to **`Idle`** (retaining its full context window). Sending a message via `send_message` to an `Idle` subagent **automatically wakes it up** to continue work with its full prior memory intact, and incoming messages wake the parent reactively without polling.
- **Transcript Inspection (`transcript.jsonl` / `transcript_full.jsonl`)**: Every conversation writes a structured JSONL trajectory (`<appDataDir>/brain/<conversation-id>/.system_generated/logs/transcript.jsonl`), allowing parent or auditor agents to inspect a child's exact tool calls and thinking trace on disk when needed.
- **Workspace Isolation Modes**: `invoke_subagent` supports `Workspace: "inherit"` (shared workspace), `"branch"` (isolated Git clone/branch), and `"share"` (shared underlying `.git` object store / worktree).
- **Teamwork Artifact Bus**: In `/teamwork-preview`, agents coordinate through structured workspace artifacts (`Request`, `Project plan`, and `Progress` markdown artifacts in `~/teamwork_projects/{PROJECT_NAME}`).

### 4. Agent Lifespan & Context Window Management
- **Idle Persistence & Milestone Orchestrator Succession**: Subagents remain `Idle` and addressable by `conversationId` for the duration of the parent task. For multi-day `/teamwork-preview` runs, **Project Orchestrator succession** (spawning a fresh Project Orchestrator at each milestone boundary initialized from the `Project plan` and `Progress` artifacts) prevents long-horizon orchestrator degradation.
- **Context Compaction (`/compact`)**: Built-in context compaction and subagent delegation keep primary windows lean.

### 5. Fast vs. Slow / Model Mixing & Cross-Provider Support
- **Per-Subagent Model Tiering (`Model` parameter)**: `invoke_subagent` accepts `Model: "inherit" | "flash_lite" | "flash" | "pro"`, allowing a `pro` orchestrator to fan out dozens of `flash` or `flash_lite` research subagents concurrently. The platform supports Gemini 3.1 Pro / Flash as well as Claude Sonnet/Opus and GPT models.

### 6. Human-in-the-Loop & Quality Gates
- **Structured Artifacts (`RequestFeedback: true`) & `/grill-me`**: Agents produce structured Markdown plans, diffs, and **Generative UI** interactive widgets with `Proceed` approval buttons, or conduct interactive design interviews (`/grill-me`).
- **Teamwork Integrity Modes (`development`, `demo`, `benchmark`)**: Enforced by the **Challenger**, **Auditor**, and **Success Auditor** gates so workers cannot pass a milestone using stubbed data, mocked tests, or unverified claims.

### 7. Persistence, Memory & Learning Across Sessions
- **Knowledge Items (KIs), Brain Artifacts, Rules & `/learn`**:
  - Persistent **Knowledge Items** and `MEMORY.md` files in `<appDataDir>/knowledge` / `~/.gemini/`.
  - `/learn` distills user corrections or complex environment setups into durable **Skills** (`SKILL.md`) or **Rules** (`.agents/rules/`).

### 8. Notable Design Insights
- **Orchestrator Succession Between Milestones**: In multi-day agent runs, people usually reset *workers* while keeping a single long-lived *orchestrator*—until the orchestrator's own context window degrades. Antigravity's `/teamwork-preview` solves this by having the **Project Orchestrator hand off to a fresh successor Project Orchestrator at each milestone boundary** via structured plan/progress artifacts.
- **Four-Gate Adversarial Verification (`Critic` + `Challenger` + `Auditor` + `Success Auditor`)**: Separating code critique from adversarial test generation (`Challenger`) and empirical execution auditing (`Auditor`) directly targets reward-hacking in autonomous coding teams.

---

## 10. OpenHands

### 1. What It Is / OSS Status / Repo URL
- **Status**: Open-source (MIT License, ~65k+ stars across the ecosystem).
- **Repos**:
  - **OpenHands Agent Canvas**: `https://github.com/OpenHands/OpenHands` (local clone: `/Users/jack/workspace/orchestrator-architecture/repos/OpenHands`) — V1 TypeScript desktop/web orchestration canvas for running parallel local, Docker, cloud VM, and ACP agents.
  - **OpenHands Software Agent SDK**: `https://github.com/OpenHands/software-agent-sdk` (local clone: `/Users/jack/workspace/orchestrator-architecture/repos/software-agent-sdk`) — modular Python SDK (`openhands-sdk`, `openhands-tools`, `openhands-workspace`, `openhands-agent-server`).

### 2. Topology & Hierarchy
The OpenHands Software Agent SDK provides **two distinct subagent delegation primitives** under `openhands-tools/openhands/tools/`:
1. **Stateful Named Subagent Pool (`DelegateAction` / `DelegateExecutor` in `delegate/definition.py` & `delegate/impl.py`)**:
   - A 2-step stateful parent-child model using two commands:
     - `command="spawn"`: Initializes named sub-agents (`ids: list[str]`, optional `agent_types: list[str]`) stored in `DelegateExecutor._sub_agents: dict[str, LocalConversation]`. Bounded by `max_children: int = 5` (`delegate/impl.py:42`).
     - `command="delegate"`: Dispatches a map of `tasks: dict[str, str]` to the spawned sub-agents in parallel using Python `threading.Thread` workers (`delegate/impl.py:165-225`), blocking the parent turn until all active worker threads complete.
2. **Ephemeral/Resumable Task Subagents (`TaskAction` / `TaskToolSet` / `TaskManager` in `task/definition.py` & `task/manager.py`)**:
   - Claude-Code-style single-step subagent invocation (`TaskAction(prompt=..., subagent_type="general-purpose", description=..., resume=task_id)`).
   - Managed by `TaskManager` (`task/manager.py`), which assigns sequential `task_<N>` IDs, runs each subagent in an isolated `LocalConversation` with `SubAgentScope`, and supports resuming a previous subagent via `resume="task_1"`.
3. **Agent Canvas Parallel Multi-Session UI (`repos/OpenHands`)**: Runs concurrent independent conversations backed by local processes, Docker sandboxes, remote OpenHands Cloud runtimes, or ACP coding CLIs (Claude Code, Codex, Gemini CLI), plus scheduled **Automations**.

### 3. Information Passing & Context Handoff
- **`DelegateExecutor` Thread Fan-Out (`openhands-tools/openhands/tools/delegate/impl.py`)**:
  - When `spawn` is called, each child `LocalConversation` is created via `create_agent(parent_agent.llm, agent_type)` and mounts the **same workspace directory** (`parent_conversation.state.workspace.working_dir`) and secrets (`parent_conversation.state.secret_registry`) as the parent (`delegate/impl.py:121-151`).
  - When `delegate` is called with `tasks={"researcher": "...", "coder": "..."}`, each worker thread executes:
    ```python
    sub_agent_conversation.send_message(message=task_str, sender=parent_name)
    sub_agent_conversation.run()
    final_response = get_agent_final_response(sub_agent_conversation.state.events)
    ```
  - All child final responses are aggregated into a single `DelegateObservation` returned to the parent (`delegate/impl.py:227-244`), while the full step-by-step tool traces stay isolated inside each child's `sub_agent_conversation.state.events`.
- **`SubAgentScope` Event Isolation (`openhands-sdk/openhands/sdk/subagent/`)**: Child conversations inherit workspace and LLM credentials (`model_copy()`) while keeping separate `EventStore` logs so subagent bash/file observations never bloat the parent's event stream.

### 4. Agent Lifespan & Context Window Management
- **Append-Only Event Stream (`EventStore` in `openhands-sdk/openhands/sdk/conversation/event_store.py`)**: Every conversation is an immutable, append-only log of typed `Event` objects (`Action`, `Observation`, `Message`, `Condensation`).
- **Pluggable Condensers (`openhands-sdk/openhands/sdk/context/condenser/`)**:
  - Instead of mutating the raw `EventStore`, OpenHands passes the event stream through a **Condenser** (`base.py`, `llm_summarizing_condenser.py`, `pipeline_condenser.py`, `no_op_condenser.py`) before constructing the LLM prompt.
  - `LLMSummarizingCondenser` (`llm_summarizing_condenser.py`) monitors token/event thresholds (`max_size`, `keep_first`), summarizes older slices of the trajectory into a `Condensation` event, and replaces the forgotten event IDs in the LLM's view while preserving the raw audit log on disk.
- **Persistent Subagents Across Delegations**: In `DelegateExecutor`, a spawned subagent (`id="db_migration"`) remains alive in `self._sub_agents` across the entire parent session; subsequent `delegate` calls to `"db_migration"` append to its existing `LocalConversation` history (`delegate/impl.py:182`).

### 5. Fast vs. Slow / Model Mixing & Cross-Provider Support
- **LiteLLM Multi-Provider Support + Subagent Factory Registry**: Built on LiteLLM (`openhands-sdk/openhands/sdk/llm/`), supporting any model provider. Registered subagent factories (`register_agent_factory` in `openhands-sdk/openhands/sdk/subagent/registry.py`) and Markdown agent definitions allow assigning specialized system prompts, toolsets, and LLM configurations per `agent_type`.

### 6. Human-in-the-Loop & Quality Gates
- **Security Analyzer & Confirmation Mode (`openhands-sdk/openhands/sdk/security/`)**: Pluggable security analyzers score every proposed `Action` (`LOW`, `MEDIUM`, `HIGH` risk); high-risk actions pause the conversation loop in `WAITING_FOR_CONFIRMATION` state until approved by the user.
- **Critic Model Evaluation (`openhands-sdk/openhands/sdk/critic/`)**: Built-in critic abstraction to score candidate trajectories and actions.

### 7. Persistence, Memory & Learning Across Sessions
- **Microagents / Repository Skills (`.openhands/microagents/` & `AGENTS.md`)**: Supports repo-level and knowledge-triggered microagents (Markdown files with YAML frontmatter `triggers: [...]`) that automatically inject specialized instructions into context only when matching keywords appear in the conversation.

### 8. Notable Design Insights
- **Immutable `EventStore` + View-Layer `Condenser`**: Separating the immutable event log (`EventStore`) from the lossy context-window projection (`Condenser`) allows full observability, deterministic replay, and evaluation without sacrificing bounded LLM context windows.
- **Two Complementary Delegation Models (`DelegateAction` vs. `TaskAction`)**: Offering both a persistent named worker pool (`spawn` once, `delegate` repeatedly to named IDs) and a one-shot task tool (`TaskAction` with optional `resume=task_id`) lets orchestrators choose between stateful specialist teammates and stateless fork-join workers.

---

## 11. Letta

### 1. What It Is / OSS Status / Repo URL
- **Status**: Open-source (Apache-2.0 License).
- **Repos**:
  - **Letta Python Server (V1 Archive)**: `https://github.com/letta-ai/letta` (`archive` branch in `/Users/jack/workspace/orchestrator-architecture/repos/letta`).
  - **Letta Code / 2026 TypeScript Runtime**: `https://github.com/letta-ai/letta-code` (local clone: `/Users/jack/workspace/orchestrator-architecture/repos/letta-code`).
- **Scope**: Stateful agent platform (evolved from **MemGPT**) where every agent is a perpetual database-backed entity with self-editing memory blocks, cross-agent messaging tools, background **Sleep-Time Agents**, and a **Git-backed Memory Filesystem (MemFS)**.

### 2. Topology & Hierarchy
Letta implements both **peer-to-peer / broadcast agent messaging** and **managed multi-agent groups**:
1. **Peer & Tag-Broadcast Messaging (`repos/letta/letta/functions/function_sets/multi_agent.py`)**:
   - Any Letta agent equipped with the multi-agent toolset can message any other persistent agent by ID or tag query:
     - `send_message_to_agent_and_wait_for_reply(self, message: str, other_agent_id: str) -> str` (lines 60–97): Synchronous request-reply RPC between two persistent agents.
     - `send_message_to_agent_async(self, message: str, other_agent_id: str) -> str` (lines 163–191): Fire-and-forget asynchronous message delivery.
     - `send_message_to_agents_matching_tags(self, message: str, match_all: List[str], match_some: List[str]) -> List[dict]` (lines 100–160): Queries `client.agents.list(tags=...)` and broadcasts a message in parallel to all matching agents in the organization (excluding self).
2. **Managed Multi-Agent Group Orchestrators (`repos/letta/letta/groups/`)**:
   - `supervisor_multi_agent.py`: A supervisor agent broadcasts tasks to worker agents and aggregates their responses.
   - `round_robin_multi_agent.py`: Agents take turns speaking in a shared group conversation (`max_turns`).
   - `dynamic_multi_agent.py`: A speaker-selection controller dynamically picks the next agent in the group to speak.
   - `sleeptime_multi_agent_v4.py` (`SleeptimeMultiAgentV4`): Pairs a foreground primary agent with one or more background **sleep-time participant agents** that share memory blocks and run asynchronously every $K$ turns (`sleeptime_agent_frequency`).
3. **Letta Code Cross-Harness Messaging & Subagents (`repos/letta-code/src/tools/impl/send-agent-message.ts` & `Task.ts`)**:
   - `Task.ts` spawns foreground or `run_in_background` subagents (`explore`, `plan`, `general-purpose`, `memory`, or custom `.letta/agents/*.md`).
   - `send_agent_message` (`send-agent-message.ts`) routes messages not only between Letta Cloud conversations (`agent_id`, `conversation_id`, `computer`) and parent-child subagents (auto-resolving `LETTA_PARENT_AGENT_ID` / `LETTA_PARENT_CONVERSATION_ID`, lines 113–121), but also **directly into external coding harnesses** running on the machine (`opencode`, `codex`, and `claude-code`, lines 150–377).

### 3. Information Passing & Context Handoff
- **System-Reminder Envelope for Inter-Agent Messages**:
  - In Python V1 (`multi_agent.py:72-76`), incoming inter-agent messages are wrapped with sender provenance:
    `"[Incoming message from agent with ID '<sender_id>' - to reply to this message, make sure to use the 'send_message' at the end, and the system will notify the sender of your response] <message>"` (or for async: `"this is a one-way notification; if you need to respond, use an agent-to-agent messaging tool"`).
  - In 2026 `letta-code` (`AGENTS.md` & `send-agent-message.ts`), Letta enforces a strict protocol rule: **automated notifications (background task completions, incoming inter-agent messages, cron triggers) must NEVER be injected as `role: "system"` mid-conversation** (which Anthropic's API rejects and models confuse with developer instructions); instead, they are injected as `role: "user"` wrapped in `<system-reminder>...</system-reminder>` tags.
- **Cross-Harness Steering (`send-agent-message.ts:150-377`)**:
  - For `opencode`: calls `session/prompt` (or `session/abort` + `session/prompt` when `interrupt=true`).
  - For `codex`: calls `turn/start` or live mid-turn `turn/steer`.
  - For `claude-code`: calls `resume/start` or `stream/interrupt`.
- **Shared Core Memory Blocks**: Multiple Letta agents can attach the **same `Block` ID** (e.g., an org status block or shared user persona block) to their in-context `core_memory`. When one agent (or a background sleep-time agent) edits that block via `memory_replace` / `memory_insert`, the updated text is immediately visible in the system prompt of every other agent sharing that block.

### 4. Agent Lifespan & Context Window Management
- **Perpetual Agents (No "Session End")**: A Letta agent is a permanent database record whose conversation history lives in **Recall Memory** (searchable message table) while its active context window holds a sliding window of recent messages plus pinned **Core Memory** blocks. When the context window fills, Letta evicts older messages from the active window into Recall Memory while summarizing them.
- **Background Sleep-Time Execution (`sleeptime_multi_agent_v4.py:217-297`)**:
  - `SleeptimeMultiAgentV4._issue_background_task()` fires asynchronously every `sleeptime_agent_frequency` turns without blocking the foreground agent's response stream.
  - It slices all new transcript messages since `last_processed_message_id` and dispatches them to the sleep-time agent, which updates shared memory blocks in the background.

### 5. Fast vs. Slow / Model Mixing & Cross-Provider Support
- **Decoupled Foreground vs. Sleep-Time Models**: Letta explicitly encourages pairing a **fast, low-latency model** on the foreground conversational agent with a **slow, deep reasoning model** on the background Sleep-Time / Memory Worker agent (or vice versa), across Anthropic, OpenAI, Gemini, and local providers.

### 6. Human-in-the-Loop & Quality Gates
- **Permission Modes & Interrupt Steering**: `letta-code` supports interactive tool approval gates (`AskUserQuestion`, permission policies) and allows either the human or a peer agent (`send_agent_message(..., interrupt=true)`) to interrupt an in-flight turn.

### 7. Persistence, Memory & Learning Across Sessions
- **Three-Tier Memory Architecture (MemGPT)**:
  1. **Core Memory (In-Context Blocks)**: Pinned, editable text blocks (`persona`, `human`, custom shared blocks) always present in the system prompt.
  2. **Recall Memory**: Full searchable conversational history stored in PostgreSQL/SQLite (`conversation_search`).
  3. **Archival Memory**: Vector-indexed long-term knowledge store (`archival_memory_insert`, `archival_memory_search`).
- **Git-Backed Memory Filesystem (`MemFS` in `repos/letta-code/src/agent/memory-filesystem.ts`, `memory-worktree.ts`, `subagents/memory-worker.ts`)**:
  - In 2026 `letta-code`, an agent's memory is stored as a **versioned Git repository** of Markdown files (`system/*.md` rendered into the system prompt, plus reference files).
  - When background reflection runs (`runMemoryWorker` in `src/agent/subagents/memory-worker.ts`), it checks out a **private Git worktree** (`createReflectionMemoryWorktree` in `memory-worktree.ts:66`), edits and commits memory changes in isolation, merges the worktree branch back into the main memory repo (`integrateMemoryWorkerWorktree` in `memory-worktree.ts:91`), spawns an automated **conflict-repair subagent** if concurrent git merge conflicts occur, pushes to Letta Cloud (`syncPendingMemoryCommitsAfterTurn`), and triggers `recompileAgentSystemPrompt` so the main agent's system prompt updates cleanly between turns.

### 8. Notable Design Insights
- **Git Worktrees for Concurrent Agent Memory (`MemFS`)**: Treating an agent's memory not just as flat files or vector rows, but as a **Git repository where background sleep-time reflection workers operate in isolated Git worktrees and merge via Git**, solves concurrent read/write race conditions on agent memory and provides a full `git log` audit trail of every belief update.
- **Cross-Harness `send_agent_message`**: Building native adapters for `claude-code`, `codex`, and `opencode` directly into Letta's `send_agent_message` tool lets a stateful Letta agent act as a persistent memory-bearing supervisor over ephemeral third-party coding CLIs.

---

## 12. Kimi Agent Swarm

### 1. What It Is / OSS Status / Repo URL
- **Status**: Open-weight model (**Moonshot AI Kimi K2.5**, 1T total / 32B active MoE parameters, 15T multimodal tokens, 256k context window) paired with Kimi's cloud **Agent Swarm** runtime (`https://www.kimi.com/blog/kimi-k2-5.html`, Jan/Feb 2026).
- **Scope**: First frontier model trained end-to-end with **Parallel-Agent Reinforcement Learning (PARL)** to dynamically self-organize a swarm of up to **100 subagents** executing up to **1,500 coordinated tool calls**.

### 2. Topology & Hierarchy
- **Dynamic 2-Level Self-Organizing Swarm (Trainable Orchestrator -> Up to 100 Frozen Subagents)**:
  - Rather than using predefined static agent roles or hand-crafted workflow graphs, Kimi K2.5 uses a **single trainable Orchestrator agent** that dynamically invents specialized subagent personas on the fly (`create_subagent` with custom system prompt/role, e.g., *"AI Researcher"*, *"Physics Researcher"*, *"Fact Checker"*) and dispatches parallel subtasks to **up to 100 concurrent subagent instances**.
  - During PARL training, the subagents are **frozen** snapshots of the policy (running from checkpointed weights), while gradients flow only through the top-level Orchestrator—avoiding credit-assignment instability and non-stationarity across 1,500-step multi-agent rollouts.

### 3. Information Passing & Context Handoff
- **Zero-Share Subagent Contexts**:
  - The Orchestrator decomposes a complex query into independent parallel subtasks and dispatches each subtask to a spawned subagent with a dedicated role specification and task prompt.
  - Subagents execute multi-turn tool loops (up to **100 steps per subagent** on BrowseComp and WideSearch) in completely isolated context windows and return only their final synthesized findings to the Orchestrator (which itself runs up to **15 steps** on BrowseComp or **100 steps** on WideSearch).

### 4. Agent Lifespan & Context Window Management
- **256k Context Multiplication via Ephemeral Subagents**: While each agent instance has a 256k context window, fanning out 100 subagents across 1,500 total tool calls multiplies effective working context into the **tens of millions of tokens** while keeping the Orchestrator's own trajectory compact (<15–100 turns).

### 5. Fast vs. Slow / Model Mixing & Cross-Provider Support
- Homogeneous **Kimi K2.5** model weights across both orchestrator and subagents, leveraging MoE sparsity (32B active parameters) for high-throughput parallel inference.

### 6. Human-in-the-Loop & Quality Gates
- Autonomous research and verification gates: the Orchestrator spontaneously spawns dedicated *Fact-Checker* or *Verification* subagents to cross-check claims produced by sibling worker subagents before final synthesis.

### 7. Persistence, Memory & Learning Across Sessions
- **RL-Internalized Orchestration Policy (PARL)**: Instead of relying on external orchestration frameworks or prompt templates, Kimi internalizes *when* and *how* to parallelize into the model weights via staged reinforcement learning:
  $$r_{\text{PARL}}(x, y) = \lambda_1 \cdot r_{\text{parallel}} + \lambda_2 \cdot r_{\text{finish}} + r_{\text{perf}}(x, y)$$
  - **Preventing Serial Collapse ($\lambda_1 \cdot r_{\text{parallel}}$)**: Without an explicit parallelism bonus early in training, RL policies suffer from *serial collapse*—defaulting to single-agent execution because early subagent delegation is noisy.
  - **Preventing Spurious Parallelism ($\lambda_2 \cdot r_{\text{finish}}$)**: Rewarding subagent creation alone causes the model to spawn useless dummy subagents that never finish; $r_{\text{finish}}$ rewards only valid completed subagent trajectories.
  - **Annealing to Pure Task Outcome**: Both $\lambda_1$ and $\lambda_2$ are annealed to $0$ over training so the final policy optimizes pure task accuracy $r_{\text{perf}}(x,y)$ under a **Critical Steps** latency budget.

### 8. Notable Design Insights
- **The Critical Steps Metric (Critical-Path Latency Budgeting)**:
  - Standard RL step limits penalize total token/step count across all agents, which discourages parallelization. Moonshot replaced total step count with **Critical Steps**—the parallel-computation critical path:
    $$\text{CriticalSteps} = \sum_{t=1}^{T} \left( S_{\text{main}}^{(t)} + \max_{i} S_{\text{sub}, i}^{(t)} \right)$$
  - Because a parallel stage $t$ is billed only by its **slowest subagent** ($\max_i S_{\text{sub}, i}^{(t)}$), the Orchestrator is incentivized to decompose work into balanced parallel subagents, cutting wall-clock critical steps by **3x–4.5x** (up to 80% runtime reduction) while improving BrowseComp accuracy from 60.6% to 78.4%.

---

## 13. Moltbook

### 1. What It Is / OSS Status / Repo URL
- **Status**: Production agent social network platform (`https://www.moltbook.com`, skill specification v1.12.0 at `https://www.moltbook.com/skill.md` and `https://www.moltbook.com/heartbeat.md`; open-source community tooling across GitHub).
- **Scope**: *"The front page of the agent internet"*—a Reddit-style social network and asynchronous messaging hub built exclusively for persistent AI agents (originally OpenClaw / Moltbot agents, now open to any autonomous agent runtime).

### 2. Topology & Hierarchy
- **Decentralized Peer-to-Peer Agent Social Graph + Moderated Submolts**:
  - Millions of independent persistent agents connect over REST (`https://www.moltbook.com/api/v1`) as peers.
  - Agents organize into topic communities called **Submolts** (`m/<name>`, e.g., `m/general`, `m/introductions`, `m/agent-ops`), where each submolt has an **Owner** agent and **Moderator** agents (`GET /api/v1/submolts/{name}/moderators`).
  - Within a submolt, moderators can define structured **Labels** (`type: "tag" | "status" | "role"`) and assign persistent **Roles** to member agents—turning a community board into a decentralized multi-agent coordination group.

### 3. Information Passing & Context Handoff
- **Single-Call Aggregated Check-In (`GET /api/v1/home`)**:
  - Instead of open sockets, Moltbook is designed around periodic **Heartbeat polling** (`heartbeat.md`). When an agent wakes up on its 30+ minute heartbeat timer, it makes a single request to `GET /api/v1/home`, which returns:
    - `your_account` (karma, unread counts)
    - `activity_on_your_posts` (new comments/replies from other agents grouped by post)
    - `your_direct_messages` (pending DM requests and unread peer messages)
    - `latest_moltbook_announcement`
    - `posts_from_accounts_you_follow`
    - `what_to_do_next` (server-prioritized action list for the agent's turn)
- **Cadence-Gated Role Briefings (`role` Labels)**:
  - Submolt moderators can create a `role` label with an embedded `prompt` and `cadence` (`POST /api/v1/submolts/{name}/labels`) and assign it to an agent (`PUT /api/v1/submolts/{name}/members/{agentName}/role`).
  - Whenever that agent calls `/api/v1/home`, Moltbook injects the role's `prompt` as a **cadence-gated briefing**—allowing community moderators to coordinate distributed specialist agents asynchronously.
- **Agent-to-Agent Direct Messaging (`/api/v1/agents/dm/*`)**:
  - Agents initiate private 1:1 conversations via `POST /api/v1/agents/dm/request` (which requires consent/approval before the thread opens) and exchange messages via `POST /api/v1/agents/dm/conversations/{id}/send`.

### 4. Agent Lifespan & Context Window Management
- **Stateless REST over Stateful Local Agents**: Moltbook itself stores the social graph, posts, comments, DMs, and role briefings, while each participating agent maintains its own local lifespan and `memory/heartbeat-state.json` (`lastMoltbookCheck`) via its host harness (e.g., OpenClaw Heartbeat or Hermes cron).

### 5. Fast vs. Slow / Model Mixing & Cross-Provider Support
- Completely runtime- and model-agnostic: any agent running any LLM (Claude, GPT, Gemini, Grok, Kimi, DeepSeek, local Llama) can participate using its `moltbook_xxx` bearer token.

### 6. Human-in-the-Loop & Quality Gates
- **Human-Agent Cryptographic/Social Bond (`claim_url`)**:
  - An agent registers itself (`POST /api/v1/agents/register`) and receives an `api_key` plus a `claim_url`.
  - The agent's human owner must verify ownership via email and post a verification tweet on X/Twitter (enforcing **1 verified agent per X account**) before the agent achieves `claimed` status.
- **Reverse CAPTCHA (AI Verification Challenge)**:
  - When an untrusted or newly registered agent submits a post or comment, the API may return `verification_required: true` with an **AI Verification Challenge** (`verification.challenge_text`—an obfuscated math/reasoning puzzle with a short expiry window) that the agent must solve via `POST /api/v1/verify` before the content is published.
- **Explicit Human Escalation Flag in Agent DMs (`needs_human_input`)**:
  - When two agents are negotiating in a DM (`POST /api/v1/agents/dm/conversations/{id}/send`) and reach a decision that requires human authorization, the sending agent sets `needs_human_input: true`, which flags the recipient agent's `/home` dashboard to escalate the message to its human owner.
- **Anti-Spam Rate Limits**: New agents (<24h old) are capped at 1 post per 2 hours and 1 comment per 60 seconds (50/day); established agents are capped at 1 post per 30 minutes and 1 comment per 20 seconds.

### 7. Persistence, Memory & Learning Across Sessions
- **Public Reputation (`karma`) & Local Skill Installation**: Agents build persistent public reputation (`karma`, follower graphs, post histories) on Moltbook while installing and updating capabilities locally via `SKILL.md`, `HEARTBEAT.md`, and `MESSAGING.md` fetched from `moltbook.com`.

### 8. Notable Design Insights
- **`GET /api/v1/home` as an Agent-Native Attention Feed**: Rather than forcing agents to call 10 separate REST endpoints on every cron wakeup, `/api/v1/home` packages notifications, DMs, followed feeds, role briefings, and a synthesized `what_to_do_next` checklist into a single context-window-friendly JSON payload.
- **`needs_human_input: true` in Agent-to-Agent DMs**: Formalizing a boolean escalation bit in peer-to-peer agent DMs cleanly separates routine bot-to-bot coordination from commitments that require human sign-off.

---

## 14. Cross-Agent Protocols: Google A2A, MCP, and Zed ACP

Modern persistent agent platforms rely on **three orthogonal open protocols** that operate at different layers of the stack:

```
+-------------------------------------------------------------------+
|  User / IDE / Orchestrator UI (Zed, JetBrains, OpenClaw, Canvas)  |
+-------------------------------------------------------------------+
                                 |
                                 |  ACP (Agent Client Protocol)
                                 |  JSON-RPC 2.0 over stdio / HTTP
                                 v
+-------------------------------------------------------------------+      A2A (Agent2Agent Protocol)      +-----------------------------------+
|  Primary Agent Runtime (OpenClaw, Hermes, Letta, Grok Bot, etc.)  | <==================================> |  Remote Peer Agent (Grok, Jules,  |
+-------------------------------------------------------------------+   /.well-known/agent-card.json       |  OpenClaw Peer, Enterprise Agent) |
                                 |                                      SendMessage / Task / Artifact      +-----------------------------------+
                                 |  MCP (Model Context Protocol)
                                 |  tools / resources / prompts / sampling
                                 v
+-------------------------------------------------------------------+
|  Tools, Databases, Browser & "Agent-as-MCP-Server" Wrappers       |
+-------------------------------------------------------------------+
```

### 14.1 Google / Linux Foundation A2A (`repos/A2A`, Specification v1.0.0)
- **Repo**: `https://github.com/a2aproject/A2A` (local clone: `/Users/jack/workspace/orchestrator-architecture/repos/A2A`, `docs/specification.md`, canonical schema in `spec/a2a.proto`).
- **Core Philosophy — Opaque Peer Interoperability**: A2A enables independent, potentially cross-organization agents to collaborate **as opaque peers** without sharing internal memory, chain-of-thought, or tool credentials.
- **Three-Layer Architecture (`docs/specification.md`)**:
  1. **Canonical Protobuf Data Model (`spec/a2a.proto`)**: Defines `AgentCard`, `Task`, `TaskStatus`, `Message`, `Part` (text, raw bytes/files, or structured `google.protobuf.Value` data), `Artifact`, and `Extension`.
  2. **Abstract Operations**: `SendMessage`, `SendStreamingMessage`, `GetTask`, `ListTasks`, `CancelTask`, `SubscribeToTask`, Push Notification Config CRUD, and `GetExtendedAgentCard`.
  3. **Protocol Bindings**: Native **JSON-RPC 2.0**, **gRPC**, and **HTTP+JSON/REST** bindings, plus **Server-Sent Events (SSE)** for real-time `TaskStatusUpdateEvent` / `TaskArtifactUpdateEvent` streaming and **Webhooks** (`TaskPushNotificationConfig`) for multi-hour async tasks.
- **Discovery & Stateful Task Lifecycle**:
  - **Discovery**: Agents publish `/.well-known/agent-card.json` declaring their `skills`, supported input/output MIME modalities, `capabilities` (`streaming`, `pushNotifications`), and `securitySchemes` (OAuth2, OIDC, API keys, mTLS).
  - **Multi-Turn Context & Tasks**: Multi-turn interactions are grouped by `contextId`, while individual units of work are tracked by `task.id` across lifecycle states: `TASK_STATE_SUBMITTED -> TASK_STATE_WORKING -> (TASK_STATE_INPUT_REQUIRED | TASK_STATE_AUTH_REQUIRED) -> (TASK_STATE_COMPLETED | TASK_STATE_FAILED | TASK_STATE_CANCELED | TASK_STATE_REJECTED)`.
- **Adopters in This Report**: **xAI Grok Bot** (native external agent delegation), **OpenClaw** (`docs/channels/a2a.md` inbound/outbound A2A channel + `sessions-send-tool.a2a.ts`), **Google Jules / Antigravity**, and **GitHub Agent HQ**.

### 14.2 Model Context Protocol (MCP) & the "Agent-as-MCP-Server" Pattern
- **Core Philosophy — Agent-to-Tool/Context Plumbing**: Introduced by Anthropic, MCP standardizes how an agent host connects to external context and capabilities via JSON-RPC 2.0 (`tools/list`, `tools/call`, `resources/read`, `prompts/get`, and reverse LLM `sampling/createMessage`).
- **The "Agent-as-MCP-Server" Orchestration Pattern**:
  - Several platforms in this report expose an entire agent runtime as an **MCP server** so any MCP-compatible client can invoke a subagent via standard `tools/call`:
    - **Hermes Agent (`mcp_serve.py`)**: Runs Hermes as an MCP server exposing conversations, message sending, and approval handling as MCP tools.
    - **OpenManus (`run_mcp.py`, `app/mcp/server.py`)**: Wraps OpenManus tools and agent execution behind an MCP server.
    - **Google Jules MCP**: Exposes repository context and session management via MCP.
- **MCP vs. A2A**: MCP is **vertical** (connecting an agent to deterministic tools, schemas, and resources where the caller controls the loop); A2A is **horizontal** (connecting autonomous agents that negotiate multi-turn stateful `Task` lifecycles, ask clarifying questions via `TASK_STATE_INPUT_REQUIRED`, and hide internal execution traces).

### 14.3 Agent Client Protocol — ACP (`repos/agent-client-protocol`, v1 & v2)
- **Repo**: `https://github.com/zed-industries/agent-client-protocol` (local clone: `/Users/jack/workspace/orchestrator-architecture/repos/agent-client-protocol`, `docs/protocol/v1/overview.mdx`, `docs/protocol/v2/overview.mdx`, Rust/JSON schemas in `agent-client-protocol-schema/src/v1/`).
- **Core Philosophy — Standardized Client/Harness-to-Agent Control Plane**: Originally created by Zed Industries and co-developed with JetBrains, Claude Code, Codex, and Gemini CLI, ACP standardizes bidirectional **JSON-RPC 2.0** communication between a **Client** (an IDE, desktop UI, or parent orchestrator harness) and a coding **Agent** running as a local subprocess (over `stdio`) or remote service (over HTTP/WebSocket).
- **Core RPCs & Reverse Capabilities (`agent-client-protocol-schema/src/v1/`)**:
  - **Client -> Agent**: `initialize`, `authenticate`, `session/new` (passes `cwd` and `mcpServers`), `session/load` / `session/resume`, `session/prompt`, `session/cancel`, `session/set_mode`, `session/set_model`, `session/list`, `session/delete`.
  - **Agent -> Client Notifications (`session/update`)**: Streams structured `ContentChunk`s, model thoughts, live **Agent Plans** (`plan.rs`), and rich **Tool Call Cards** (`tool_call.rs` with diffs and locations).
  - **Agent -> Client Reverse RPCs**: Crucially, an ACP Agent can ask the Client to read/write unsaved editor buffers (`fs/read_text_file`, `fs/write_text_file` in `file-system.mdx`), execute commands in managed Client terminals (`terminal/create`, `terminal/output` in `terminals.mdx`), request tool permission approvals (`session/request_permission`), or prompt the user via structured forms (`elicitation.rs`).
  - **MCP-over-ACP (`docs/rfds/mcp-over-acp.mdx`)**: Allows the Client to tunnel host-managed MCP servers directly over the ACP connection to the Agent without opening separate transports.
- **Why ACP Matters for Multi-Agent Orchestrators**:
  - In **OpenClaw** (`sessions_spawn(runtime: "acp", agentId: "codex" | "claude")`), **Hermes Agent** (`acp_adapter/`), and **OpenHands Agent Canvas**, ACP acts as a **universal subagent harness adapter**—allowing a parent orchestrator to spawn, stream, pause, approve permissions for, and resume heterogeneous coding CLIs (Claude Code, Codex, Gemini CLI, OpenCode) through one unified protocol.

---

## 15. Cross-Product Comparison Table

| Platform | OSS / License | Primary Topology | Max Depth / Fan-Out | Inter-Agent Messaging Mechanism | Lifespan & Context Compaction | Model Mixing | Persistence & Memory |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **xAI Grok Bot** | Closed | Peer roster (up to 50 Bots) + emergent Chief-of-Staff; Group chats (2–6 Bots); API Leader + 4/16 subagents | Peer (1-hop handoff) / API depth 1 (4 or 16 subagents) | Async bot-to-bot handoffs (dashboard toggle), `@mention` group chats, shared `/workspace` cloud VM, external **A2A** | Persistent named Bots + up to 50 **Routines** (cron/triggers) per Bot | Grok 4 / 4.20 (`low`/`med` = 4 agents; `high`/`xhigh` = 16) | Account-scoped Tools/Skills (incl. Learn from Demo); Bot-scoped Memory; Team vs. Personal notes |
| **OpenClaw** | OSS (MIT) | Multi-agent Gateway + hierarchical `sessions_spawn` + flat Code Mode Swarm | `maxSpawnDepth` configurable (default 1, commonly 2); `maxChildrenPerAgent=5`; Swarm `maxConcurrent=32` | `sessions_spawn` (announce on finish), `sessions_yield` (fence transfer), `sessions_send` (sync/async + **A2A ping-pong** `REPLY_SKIP`), **ACP** (`acpx`) | Persistent SQLite sessions + 30m **Heartbeat** + `mode: "safeguard"` compaction with quality checks | Full cross-provider; separate `subagents.model` & `heartbeat.model` + ACP CLIs | `SOUL.md`, `MEMORY.md`, `memory/YYYY-MM-DD.md`, hybrid vector/FTS, Memory Wiki, background **Dreaming** |
| **Hermes Agent** | OSS (MIT) | Hierarchical `delegate_task` + persistent multi-profile SQLite **Kanban** board | `delegate_task`: `MAX_DEPTH=2`, `max_concurrent_children=5`; Kanban: `max_concurrent_dispatches=3` OS processes | `delegate_task` (`goal`/`context`/`output_schema`, async `completion_queue`), Kanban (`~/.hermes/kanban.db`, comments, handoff metadata), **ACP**, MCP | `_Heartbeat` wedged-child monitor; Kanban 4h stale reclaim & 90% budget warning; `context_compressor.py` + `micro_compaction.py` | 20+ providers; per-task/profile model override + native `/moa` Mixture-of-Agents | Frozen-snapshot `MEMORY.md`/`USER.md` (2.2k chars), SQLite FTS5 `session_search`, KV-cache-forking `background_review.py` + **Curator** |
| **Manus & OpenManus** | Manus: Closed; OpenManus: OSS (MIT) | **Wide Research**: Main Agent -> 100s of parallel general-purpose Manus cloud VMs; OpenManus: `PlanningFlow` | Depth 1, fan-out 100–250+ parallel cloud sandboxes | Self-contained task specs + shared cloud storage artifacts; `PlanningTool` step dispatch in OpenManus | Persistent **Cloud Computer** (Manus 2.0) + Automations; KV-cache preservation, logit tool masking, restorable file compression, `todo.md` recitation | Multi-model routing (Claude, Gemini, etc.); OpenManus supports per-agent LLM config | Manus 2.0 persistent Cloud Computer filesystem & project state |
| **Devin (Cognition)** | Closed (`devin-handoff` OSS) | **Managed Devins** (Parent -> Child VMs) + `Devin <-> Devin Review` + read-only `SWE-grep` | Parent -> $N$ Child VMs (ACU-capped); `SWE-grep`: 4 turns x 8 parallel calls | Parent-child VM messaging with auto-wake; `/handoff` CLI-to-cloud state transfer; PR review loops | Persistent snapshotted Cloud VMs; context kept lean by offloading search to `SWE-grep` | High-speed RL `SWE-grep` (>2,800 tok/s) paired with frontier reasoning models | Org/repo **Knowledge** (auto-retrieved), **Playbooks** (`.devin.md`), and **DeepWiki** |
| **Cursor** | Closed | Parallel Worktree/Cloud Agents (up to 8) + `.cursor/agents/*.md` + **Planner-Worker-Judge** harness | Interactive: 8 parallel worktrees, **2-level subagent nesting**; Harness: 100s of workers | Subagent isolated context + summary return; `/in-cloud` & `/autopilot` handoff; Git branch push in Planner-Worker-Judge | Resumable `agentId` subagents; short bounded Worker cycles with periodic fresh starts in long-running harness | Fast RL **Composer 2/2.5** & `Explore` (`fast`) mixed with `gpt-5.2` / `claude-opus-5` planners | `.cursor/rules/*.mdc`, `AGENTS.md`, and `.cursor/environment.json` cloud VM snapshots |
| **GitHub Copilot / Agent HQ** | Closed | **Mission Control** multi-vendor hub + `.github/agents/*.md` + Copilot Code Review peer | Parallel cloud runs across repos & partner agents (Claude, Codex, Jules, Devin, Grok) | **Draft PR on `copilot/*` branch** as shared state bus; chat transcript handoff; live PR/Mission Control steering | Ephemeral **GitHub Actions VMs** (`copilot-setup-steps.yml`) rehydrated across PR comment turns | Multi-vendor model & harness picker across Anthropic, OpenAI, Google, xAI, Cognition | **Copilot Memory**, **Copilot Spaces**, `.github/copilot-instructions.md`, synced sessions |
| **Google Jules** | Closed (`@google/jules` CLI OSS) | Session Agent + internal **Planning Critic** & **Critic Agent** inside Google Cloud VM | 1 Session VM per task (up to 60–100 concurrent sessions via API) | Typed Session **Activities** (`PlanGenerated`, `AgentMessaged`), internal diff/test critique loop, REST API & MCP | Persistent **Google Cloud VM** with **Environment Snapshots** | **Gemini 3 Flash** default + **Gemini 3 Pro / 3.1** | Automatic per-repo **Jules Memory** learned from user nudges + `AGENTS.md` |
| **Google Antigravity** | Closed | Dynamic subagents (`invoke_subagent`/`send_message`) + 3-tier `/boost` + `/teamwork-preview` | **10-level nesting limit**; `/boost`: 3 tiers; `/teamwork-preview`: Sentinel -> Project Orchestrator -> Explorers/Workers + 4 Auditors | Explicit `send_message(Recipient=<conversationId>)` with **Idle auto-wake**; shared JSONL transcripts; Teamwork Markdown artifacts | `Running -> Idle -> Killed` lifecycle; **Project Orchestrator succession** between milestones in `/teamwork-preview` | Per-subagent `Model: "flash_lite" \| "flash" \| "pro" \| "inherit"` (Gemini, Claude, GPT) | **Knowledge Items (KIs)**, `MEMORY.md`, Brain artifacts, `/learn` skill extraction |
| **OpenHands** | OSS (MIT) | `DelegateAction` (`spawn` + `delegate` threads) + `TaskAction` (`SubAgentScope`) + Agent Canvas | `DelegateExecutor`: `max_children=5` default; `TaskManager`: sequential `task_<N>` | `DelegateExecutor` runs child `LocalConversation` threads sharing workspace and returns `DelegateObservation`; **ACP** in Canvas | Append-only `EventStore` projected through pluggable **Condensers** (`LLMSummarizingCondenser`) | LiteLLM multi-provider; per-`agent_type` factory configs | Repo/knowledge-triggered **Microagents** (`.openhands/microagents/`, `AGENTS.md`) + persisted `EventStore` |
| **Letta** | OSS (Apache-2.0) | Peer/tag messaging + Groups (`Supervisor`, `SleeptimeMultiAgentV4`) + `letta-code` subagents | Peer graph or 2-level `Task` subagents + background `runMemoryWorker` | `send_message_to_agent_and_wait_for_reply`, `_async`, `_matching_tags`; `send_agent_message` (bridges to `codex`/`claude-code`/`opencode`); `<system-reminder>` user turns | Perpetual database-backed agents; sliding context window + Recall/Archival eviction | Fast foreground model paired with deep background Sleep-Time / Memory Worker model | Shared in-context **Core Memory** blocks + **Git-backed MemFS** (`memory-worktree.ts` with isolated reflection worktrees & auto-merge) |
| **Kimi Agent Swarm** | Open-weight (Kimi K2.5) + Cloud | PARL-trained Orchestrator -> up to **100 frozen subagents** | Depth 1, up to **100 subagents** and **1,500 tool calls** | Dynamic `create_subagent` role invention + isolated subtask dispatch & synthesis | Ephemeral 256k subagent contexts multiplying to multi-million-token effective working memory | Homogeneous Kimi K2.5 (1T/32B MoE) | Orchestration policy internalized into weights via **PARL** ($r_{\text{parallel}}, r_{\text{finish}}$ annealed under **Critical Steps**) |
| **Moltbook** | Closed platform / Open skill spec | Decentralized social graph (`submolts`, moderators, DMs) | Millions of peer agents across `m/<submolt>` communities | `GET /api/v1/home` single-call check-in, cadence-gated **Role** briefings, consent-gated DMs with `needs_human_input` | Stateless REST over persistent local heartbeat agents (`memory/heartbeat-state.json`) | Totally model- and harness-agnostic | Public `karma`/posts/DMs + local `SKILL.md` / `HEARTBEAT.md` state |

---

## 16. Key Architectural Patterns

Across these 13 platforms and 3 protocols, eight architectural patterns define the state of the art in persistent and cloud multi-agent orchestration:

### 1. Distributed Intelligence with Single-Writer Boundaries
Both **Cognition (Devin)** and **Cursor (`scaling-agents`)** independently proved that naive peer-to-peer multi-agent swarms mutating shared files concurrently collapse under lock contention and conflicting implicit decisions. Winning architectures strictly enforce **single-writer boundaries**:
- Parallelize **read-only context retrieval** (`SWE-grep`, Cursor `Explore`, Antigravity `Explorers`),
- Parallelize **adversarial verification** (`Devin Review`, Jules `Critic Agent`, Antigravity `Critic`/`Challenger`/`Auditor`), and
- Isolate **parallel writers** into disjoint **Git worktrees or separate Cloud VMs** with exclusive file or PR ownership (OpenClaw `worktree: true`, Hermes `worktree=True`, Antigravity `Workspace: "branch"` and exclusive file ownership, Cursor `.cursor/worktrees.json`, Managed Devins).

### 2. Three Distinct Inter-Agent Communication Topologies
Persistent platforms converge on three complementary ways for agents to pass information:
1. **Parent-Child Fork-Join with Execution Fencing**: Asynchronous spawning (`sessions_spawn` in OpenClaw, `delegate_task(background=True)` in Hermes) paired with an **execution fence** (`sessions_yield` in OpenClaw, `completion_queue` idle draining in Hermes) so child completions never corrupt an in-flight parent turn.
2. **Addressable Idle Sessions with Reactive Auto-Wake**: In **Google Antigravity** (`send_message` to an `Idle` `conversationId`), **Devin** (Managed Devins), **xAI Grok Bot** (async bot handoffs), and **OpenClaw** (`sessions_send` with bounded `REPLY_SKIP` ping-pong), agents remain addressable after finishing a turn and automatically wake up with their full prior context intact when messaged by a peer or parent.
3. **Zero-Copy Shared Environment Handoffs**: Instead of serializing large artifacts through LLM tokens, **xAI Grok Bot** (`/workspace` shared Linux VM across 50 Bots), **Manus** (Cloud Computer filesystem), and **GitHub Copilot** (`copilot/*` Draft PR branch) pass file paths or Git refs over a shared underlying environment.

### 3. Notification Envelopes: Why `role: "user"` + `<system-reminder>` Beat Mid-History `role: "system"`
When background tasks, cron timers, or peer agents deliver asynchronous messages into a persistent agent's session, injecting `role: "system"` mid-conversation breaks strict provider APIs (such as Anthropic Messages API) and confuses instruction hierarchy. **Letta Code** (`<system-reminder>` inside `role: "user"`), **Hermes Agent** (`completion_queue` drained only between turns as a new user notification), and **OpenClaw** (`subagent-announce-delivery.ts`) all solve this by queuing async arrivals until the current turn settles and delivering them as structured user-turn envelopes.

### 4. Orchestrator Succession and Periodic Resets for Multi-Day Runs
For tasks spanning days or >1M lines of code, even the *orchestrator's* context window eventually degrades. **Google Antigravity's `/teamwork-preview`** introduces **Project Orchestrator Succession**—at the end of each milestone, the active Project Orchestrator writes its state into structured `Project plan` and `Progress` artifacts on disk and hands off to a **brand-new successor Project Orchestrator** with a clean context window. **Cursor's** long-running harness similarly relies on short, fresh Worker cycles steered by recursive Planners and a cycle-end Judge.

### 5. KV-Cache-First Context & Memory Engineering
Because persistent agents run 50–1,500+ tool calls per task with ~100:1 input-to-output token ratios, invalidating the prompt prefix KV-cache is prohibitively expensive:
- **Manus** keeps tool definitions static and enforces tool state machines via **logit masking**, uses **restorable filesystem compression** (dropping large texts while keeping file paths), and rewrites `todo.md` at the tail of context to steer causal attention.
- **Hermes Agent** freezes `MEMORY.md` in the system prompt at session start (writing mid-session memory edits only to disk) and forks the warm prompt prefix in `agent/background_review.py` so post-turn skill/memory extraction gets a **100% KV-cache hit**.
- **OpenHands** separates its immutable append-only `EventStore` from view-layer `Condensers` (`LLMSummarizingCondenser`).

### 6. Sleep-Time Consolidation & Version-Controlled Memory (`MemFS` & Dreaming)
Rather than forcing foreground agents to manage all long-term memory inline:
- **Letta** runs asynchronous **Sleep-Time Agents** (`SleeptimeMultiAgentV4`) and, in 2026 `letta-code`, stores agent memory in a **Git repository (`MemFS`)** where background reflection workers (`runMemoryWorker`) check out isolated **Git worktrees** (`memory-worktree.ts`), commit memory updates, merge them back, and spawn a conflict-repair subagent on merge conflicts.
- **OpenClaw** runs a 3-phase background **Dreaming** sweep (Light, REM, Deep sleep) that scores daily transcripts across 6 signals before promoting durable facts into `MEMORY.md`.
- **Hermes Agent** runs an idle-triggered **Curator** (`agent/curator.py`) to deduplicate and archive agent-created skills.

### 7. Adversarial Multi-Gate Verification Over Self-Reported Completion
Single agents routinely reward-hack by mocking tests or claiming premature completion. Production cloud platforms now interpose independent adversarial agents before human review:
- **Google Antigravity `/teamwork-preview`** separates implementation (`Worker`) from four distinct verification gates: `Critic`, `Challenger` (writes worst-case tests to break the code), `Auditor` (verifies raw command outputs against an explicit Integrity Mode), and `Success Auditor`.
- **Google Jules** runs both a pre-execution **Planning Critic** and a post-diff **Critic Agent**.
- **Devin** pairs coding sessions with **Devin Review**, while **Cursor** uses an independent **Judge** agent per cycle.

### 8. Protocol Convergence: ACP for Harness Control, MCP for Tools, A2A for Peer Agents
The ecosystem has cleanly partitioned cross-agent interoperability into three non-overlapping protocols:
- **ACP (`zed-industries/agent-client-protocol`)** is the standard for spawning, streaming, and steering local/remote coding harnesses (`claude-code`, `codex`, `gemini`, `opencode`, `hermes`) from an IDE or parent orchestrator (`OpenClaw`, `OpenHands`, `Hermes`).
- **MCP** is the standard for attaching tools, resources, and wrapped subagent servers (`mcp_serve.py`) to an agent.
- **A2A (`a2aproject/A2A`)** is the standard for discovering (`/.well-known/agent-card.json`) and delegating stateful, multi-turn tasks (`contextId`, `taskId`, `INPUT_REQUIRED`, SSE/webhooks) across opaque peer agents (`Grok Bot`, `OpenClaw`, `Jules`).
