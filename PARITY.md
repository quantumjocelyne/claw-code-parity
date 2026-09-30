# PARITY Gap Analysis

Date: 2026-04-01

Targeted reconciliation: 2026-10-01 against main `8805386` (skills, plan/worktree tools, hooks, and plugins). Other gaps and historical line references have not been re-audited.

Scope compared:
- Upstream TypeScript: `/home/bellman/Workspace/claude-code/src/`
- Rust port: `rust/crates/`

Method:
- Read-only comparison only.
- No upstream source was copied into this repo.
- This is a focused feature-gap report for `tools`, `hooks`, `plugins`, `skills`, `cli`, `assistant`, and `services`.

## Executive summary

The Rust port has a solid core for:
- basic prompt/REPL flow
- session/runtime state
- Anthropic API/OAuth plumbing
- a compact MVP tool registry
- CLAUDE.md discovery
- MCP config parsing/bootstrap primitives

But it is still materially behind the TypeScript implementation in the following areas:
1. **Tools surface area** is much smaller.
2. **Hooks** execute PreToolUse/PostToolUse/PostToolUseFailure; broader upstream event parity remains incomplete.
3. **Plugins** have a Rust crate, management commands, and runtime integration; full upstream parity is not established.
4. **Skills** support direct `SKILL.md` loading plus `/skills` list/install/help and local discovery; bundled/MCP skill registry parity remains incomplete.
5. **CLI** breadth is much narrower in Rust.
6. **Assistant/tool orchestration** lacks the richer streaming concurrency and orchestration behavior present in TS.
7. **Services** in Rust cover API/auth/runtime basics, but many higher-level TS services are missing.

## Critical bug status on this branch

Targeted critical items requested by the user:
- **Prompt mode tools enabled**: fixed in `rust/crates/rusty-claude-cli/src/main.rs:75-82`
- **Default permission mode = danger-full-access**: fixed in `rust/crates/rusty-claude-cli/src/args.rs:12-16`, `rust/crates/rusty-claude-cli/src/main.rs:348-353`, and starter config `rust/crates/rusty-claude-cli/src/init.rs:4-9`
- **Tool input `{}` prefix bug**: fixed/guarded in streaming vs non-stream paths at `rust/crates/rusty-claude-cli/src/main.rs:2211-2256`
- **Unlimited max_iterations**: already present at `rust/crates/runtime/src/conversation.rs:143-148` with `usize::MAX` initialization at `rust/crates/runtime/src/conversation.rs:119`

Build/test/manual verification is tracked separately below and must pass before the branch is considered done.

---

## 1) tools/

### Upstream TS has
- Large per-tool module surface under `src/tools/`, including agent/task tools, AskUserQuestion, MCP tools, plan/worktree tools, REPL, schedule/task tools, synthetic output, brief/upload, and more.
- Evidence:
  - `src/tools/AgentTool/AgentTool.tsx`
  - `src/tools/AskUserQuestionTool/AskUserQuestionTool.tsx`
  - `src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts`
  - `src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts`
  - `src/tools/EnterPlanModeTool/EnterPlanModeTool.ts`
  - `src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`
  - `src/tools/EnterWorktreeTool/EnterWorktreeTool.ts`
  - `src/tools/ExitWorktreeTool/ExitWorktreeTool.ts`
  - `src/tools/RemoteTriggerTool/RemoteTriggerTool.ts`
  - `src/tools/ScheduleCronTool/*`
  - `src/tools/TaskCreateTool/*`, `TaskGetTool/*`, `TaskListTool/*`, `TaskOutputTool/*`

### Rust currently has
- A single MVP registry in `rust/crates/tools/src/lib.rs:53-371`.
- Implemented tools include `bash`, `read_file`, `write_file`, `edit_file`, `glob_search`, `grep_search`, `WebFetch`, `WebSearch`, `TodoWrite`, `Skill`, `Agent`, `ToolSearch`, `NotebookEdit`, `Sleep`, `SendUserMessage`, `Config`, `StructuredOutput`, `REPL`, `PowerShell`, `EnterPlanMode`, `ExitPlanMode`.

### Missing or broken in Rust
- **Missing large chunks of the upstream tool catalog**: I did not find Rust equivalents for AskUserQuestion, MCP resource listing/reading tools, worktree entry/exit tools (`EnterWorktree`/`ExitWorktree`), task management tools, remote trigger, synthetic output, or schedule/cron tools.
- **Tool decomposition is much coarser**: TS isolates tool-specific validation/security/UI behavior per tool module; Rust centralizes almost everything in one file (`rust/crates/tools/src/lib.rs`).
- **Likely parity impact**: lower fidelity tool prompting, weaker per-tool behavior specialization, and fewer native tool choices exposed to the model.

---

## 2) hooks/

### Upstream TS has
- A full permission and tool-hook system with **PermissionRequest**, **PreToolUse**, **PostToolUse**, and failure/cancellation handling.
- Evidence:
  - `src/hooks/toolPermission/PermissionContext.ts:25,222`
  - `src/hooks/toolPermission/handlers/coordinatorHandler.ts:32-38`
  - `src/hooks/toolPermission/handlers/interactiveHandler.ts:412-429`
  - `src/services/tools/toolHooks.ts:39,435`
  - `src/services/tools/toolExecution.ts:800,1074,1483`
  - `src/commands/hooks/index.ts:5-8`

### Rust currently has
- Config loading/merging and an executable hook runner in `rust/crates/runtime/src/hooks.rs`.
- PreToolUse, PostToolUse, and PostToolUseFailure integrated into `rust/crates/runtime/src/conversation.rs`; pre-hooks can update input or affect permission decisions, with failure/cancellation handling and hook feedback.

### Missing or broken in Rust
- **Broader hook event parity remains incomplete**: the Rust `HookEvent` enum contains only those three events; a PermissionRequest hook event is not implemented (the permission request type itself does exist).
- Executable tool hooks are present; full upstream hook behavior parity is not established.

---

## 3) plugins/

### Upstream TS has
- Built-in and bundled plugin registration plus CLI/service support for validate/list/install/uninstall/enable/disable/update flows.
- Evidence:
  - `src/plugins/builtinPlugins.ts:7-17,149-150`
  - `src/plugins/bundled/index.ts:7-22`
  - `src/cli/handlers/plugins.ts:51,101,157,668`
  - `src/services/plugins/pluginOperations.ts:16,54,306,435,713`
  - `src/services/plugins/pluginCliCommands.ts:7,36`

### Rust currently has
- A dedicated `rust/crates/plugins/` crate with discovery, manifest validation, install/update/uninstall, enable/disable, registry, and lifecycle support in `src/lib.rs`.
- `/plugin` (aliases `/plugins`, `/marketplace`) with list/install/enable/disable/uninstall/update in `rust/crates/commands/src/lib.rs`.
- Plugin tool/hook aggregation, initialization/shutdown, and runtime reload after management changes in `rust/crates/rusty-claude-cli/src/main.rs`.

### Missing or broken in Rust
- **No top-level plugin CLI subcommand** comparable to `claude plugin ...` in `rust/crates/rusty-claude-cli/src/args.rs`; slash-command management is implemented.
- Full upstream plugin parity is not established; loading, management, validation, and runtime integration are no longer absent.

---

## 4) skills/

### Upstream TS has
- Bundled skills registry and loader integration, plus a `skills` command.
- Evidence:
  - `src/commands/skills/index.ts:6`
  - `src/skills/bundledSkills.ts:44,99,107,114`
  - `src/skills/loadSkillsDir.ts:65`
  - `src/skills/mcpSkillBuilders.ts:4-21,40`

### Rust currently has
- A `Skill` tool that loads local `SKILL.md` files directly:
  - `rust/crates/tools/src/lib.rs:1244-1255`
  - `rust/crates/tools/src/lib.rs:1288-1323`
- CLAUDE.md / instruction discovery exists in runtime prompt loading:
  - `rust/crates/runtime/src/prompt.rs:203-208`
- `/skills` supports list (also the default action), install `<path>`, and help, with local skill-root discovery in `rust/crates/commands/src/lib.rs`.

### Missing or broken in Rust
- **No visible bundled-skill registry equivalent** to TS `bundledSkills.ts` / `loadSkillsDir.ts` / `mcpSkillBuilders.ts`.
- Rust supports **direct file loading, local discovery, installation, and command integration**; full upstream bundled/MCP skill registration parity remains incomplete.

---

## 5) cli/

### Upstream TS has
- Broad CLI handler and transport surface.
- Evidence:
  - `src/cli/handlers/agents.ts:2-32`
  - `src/cli/handlers/auth.ts`
  - `src/cli/handlers/autoMode.ts:24,35,73`
  - `src/cli/handlers/plugins.ts:2-3,101,157,668`
  - `src/cli/remoteIO.ts:25-35,118-127`
  - `src/cli/transports/SSETransport.ts`
  - `src/cli/transports/WebSocketTransport.ts`
  - `src/cli/transports/HybridTransport.ts`
  - `src/cli/transports/SerialBatchEventUploader.ts`
  - `src/cli/transports/WorkerStateUploader.ts`

### Rust currently has
- Minimal top-level subcommands in `rust/crates/rusty-claude-cli/src/args.rs:29-39` and `rust/crates/rusty-claude-cli/src/main.rs:67-90,242-261`.
- Slash commands are defined in `rust/crates/commands/src/lib.rs` (`SLASH_COMMAND_SPECS`), including `/skills` and plugin management.

### Missing or broken in Rust
- **Missing major CLI subcommand families**: agents, plugins, mcp management, auto-mode tooling, and many other TS commands.
- **Missing remote/transport stack parity**: I did not find Rust equivalents to TS remote structured IO / SSE / websocket / CCR transport layers.
- **Slash command breadth is much narrower** than TS command inventory under `src/commands/`.
- **Prompt-mode parity bug** was present and is now fixed for this branch’s prompt path.

---

## 6) assistant/

### Upstream TS has
- Rich tool orchestration and streaming execution behavior, including concurrency/cancellation/fallback logic.
- Evidence:
  - `src/services/tools/StreamingToolExecutor.ts:35-214`
  - `src/services/tools/toolExecution.ts:455-569,800-918,1483`
  - `src/services/tools/toolOrchestration.ts:134-167`
  - `src/assistant/sessionHistory.ts`

### Rust currently has
- A straightforward agentic loop in `rust/crates/runtime/src/conversation.rs:130-214`.
- Streaming API adaptation in `rust/crates/rusty-claude-cli/src/main.rs:1998-2058`.
- Tool-use block assembly and non-stream fallback handling in `rust/crates/rusty-claude-cli/src/main.rs:2211-2256`.

### Missing or broken in Rust
- **No TS-style streaming tool executor** with sibling cancellation / fallback discard semantics.
- **PreToolUse/PostToolUse/PostToolUseFailure hooks participate** in assistant execution via `rust/crates/runtime/src/conversation.rs`.
- **No comparable orchestration layer for richer tool event semantics** found.
- Historically broken parity items in prompt mode were:
  - prompt tool enablement (`main.rs:75-82`) — now fixed on this branch
  - streamed `{}` tool-input prefix behavior (`main.rs:2211-2256`) — now fixed/guarded on this branch

---

## 7) services/

### Upstream TS has
- Very broad service layer, including API, analytics, compact/session memory, prompt suggestions, plugin services, MCP service helpers, LSP management, policy limits, team memory sync, notifier/tips, etc.
- Evidence:
  - `src/services/api/client.ts`, `src/services/api/claude.ts`, `src/services/api/withRetry.ts`
  - `src/services/oauth/client.ts`, `src/services/oauth/index.ts`
  - `src/services/mcp/*`
  - `src/services/plugins/*`
  - `src/services/lsp/*`
  - `src/services/compact/*`
  - `src/services/SessionMemory/*`
  - `src/services/PromptSuggestion/*`
  - `src/services/analytics/*`
  - `src/services/teamMemorySync/*`

### Rust currently has
- Core service equivalents for:
  - API client + SSE: `rust/crates/api/src/client.rs`, `rust/crates/api/src/sse.rs`, `rust/crates/api/src/types.rs`
  - OAuth: `rust/crates/runtime/src/oauth.rs`
  - MCP config/bootstrap primitives: `rust/crates/runtime/src/mcp.rs`, `rust/crates/runtime/src/mcp_client.rs`, `rust/crates/runtime/src/mcp_stdio.rs`, `rust/crates/runtime/src/config.rs`
  - prompt/context loading: `rust/crates/runtime/src/prompt.rs`
  - session compaction/runtime usage: `rust/crates/runtime/src/compact.rs`, `rust/crates/runtime/src/usage.rs`

### Missing or broken in Rust
- **Missing many higher-level services**: analytics, prompt suggestion, team memory sync, richer LSP service management, notifier/tips ecosystem, and much of the surrounding product/service scaffolding.
- Rust is closer to a **runtime/API core** than a full parity implementation of the TS service layer.

---

## Highest-priority parity gaps after the critical bug fixes

1. **Hook execution parity**
   - Tool-hook execution exists; broader upstream event coverage (including PermissionRequest hooks) remains incomplete.

2. **Plugin system parity**
   - Install/load/manage and runtime integration exist; remaining upstream parity needs a focused comparison.

3. **CLI breadth parity**
   - Missing many upstream command families and remote transports.

4. **Tool surface parity**
   - MVP tool registry exists, but a large number of upstream tool types are absent.

5. **Assistant orchestration parity**
   - Core loop exists, but advanced streaming/execution behaviors from TS are missing.

## Recommended next work after current critical fixes

1. Finish build/test/manual verification of the critical bug patch.
2. Extend **hook event parity** beyond the implemented tool-hook pipeline.
3. Assess remaining **plugin parity** against the implemented Rust crate and slash commands.
4. Expand the CLI/tool matrix deliberately rather than adding one-off commands without shared orchestration support.
