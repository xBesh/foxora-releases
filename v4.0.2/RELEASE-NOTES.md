## Foxora AI v4.0.2 — Mastra-native agent refactor + builder fix

A backend reshaping release. No visible UI changes in Chat / Code / Agents — the wins are in how the agent loop behaves, how the surfaces route to each other, and how reliably the Agents Builder saves what the AI generated.

**Phase 1 — frontend pre-flight bootstrap.** First-turn project creation no longer rides a mid-turn `set_project_path` → SSE chunk → auto-dispatch dance. The frontend POSTs `/foxora/workspace/bootstrap` BEFORE the agent stream starts, resolves a project path (open project, or a fresh one derived from the prompt slug), binds it on Redux + the session, and only then opens the agent stream. The Mastra MessageList stays frozen at a single resource for the whole turn — the cause of every "thread ownership mismatch" log line is gone with it. Net delete: ~100 lines of poll-and-retry code in `RightSidebar.tsx`.

**Phase 2 — isTaskComplete scorers replace heuristic detectors.** The two "narrate-and-stop" detectors (workflow-followup-resolved, build-intent-produced-action) that used to live as if/else branches inside `onIterationComplete` are now Mastra-native `createScorer()` definitions. The agent loop calls them after every iteration; when a scorer fails, Mastra owns the feedback injection. `code-hooks.ts` shrank by ~190 lines and is now pure observability. New file: `agents/code-task-complete-scorers.ts`.

**Phase 3 — code-agent prompt slim, ~100 → ~30 lines.** Tool descriptions + sub-agent descriptions + workspace skills + per-tool input examples do the work the prompt used to prescribe. The HOW YOU WORK / NON-NEGOTIABLE / VOICE block is the irreducible core: short, pointed, no triage tables. Reads faster, costs less, leans on the framework primitives the rest of the agent already advertises.

**Phase 4 — `x-foxora-workspace-mode` header is the single source of truth.** Frontend always declares the active surface (`chat | code | agents`) explicitly. The engine middleware reads it, stamps `requestContext.workspaceMode`, and each agent's dynamic resolver can branch off it directly — no more re-deriving from `agentId` URL + `workspaceType` + `resourceId`. Legacy derivation kept as a fallback for callers that haven't migrated.

**Phase 5 — dead conversational-mode branch deleted from code-agent prompt.** After Phase 1, the code-agent never reaches the model without a project. The ~75-line GREETING / BUILD / PATH triage block was unreachable code; it's gone. Removes the `os.homedir()` import and the entire `${workspacesDir}/<slug>-${today}` template path. The prompt is now single-path: project IS open.

**Sub-agent extraction.** `architect`, `code-reviewer`, `web-researcher` moved to dedicated files under `agents/`. The supervisor pattern still works the same way (`codeAgent.agents = { … }`), but each sub-agent has its own scoped tool set + scoped memory store now. Foundation for future consolidation if we ever want one foxoraAgent that internally supervises all three modes — Mastra 1.25 supports `agents`, `memory`, `tools`, `workspace`, `model`, `instructions` all as `DynamicArgument` per-request.

**Workspace skills.** Four SKILL.md bundles ship in the engine resources tree: `anti-patterns`, `bootstrap-fallback`, `output-format`, `tool-decision-tree`. The agent loads them on-demand via Mastra's native `skill('<id>')` tool. Skill bodies stay out of the system prompt — the agent pulls just what it needs, when it needs it.

**Background-task lifecycle wiring.** New `background-task` / `workflow-progress` / `tool-call-input-delta` chunk handlers in `chunkRouter.ts`. Cursor-style streaming of tool arguments (write_file content materializing as the model types it) now renders inline on the tool row; `run_tests` + `bash_background` lifecycle events feed a dedicated `BackgroundTasksRail`; mastra-native workflows (`scan-project`, `debug-triage`, `bootstrap-from-boilerplate`) emit per-step `WorkflowProgressInline` chips.

**Engine + Tauri JSONL logging.** `file-logger.ts` writes structured logs to `~/.foxora/logs/engine.log` with rotation + console interception. Tauri's `app_logger.rs` command sinks Rust tracing + frontend console patches to `~/.foxora/logs/app.log`. "Where did this turn go?" is now a one-grep question.

**Builder save flow fix.** Probing the Agents Builder revealed a two-layer regression where `emit_skill_spec` calls often failed Zod validation on the first attempt (model forgot the boilerplate `schema: "foxora.skill.v1"` literal) — and the frontend's `toolResultHandler.ts` treated the resulting `{ error: true, validationErrors: {…} }` retry result as a real spec, rendering the BuilderPreviewCard with every field empty. **Engine fix:** the `schema` literal is now `.default()`-ed to itself, eliminating the failure mode entirely. **Frontend fix:** the handler gates on the success marker `type: "skill_spec_emitted"` / `"agent_spec_emitted"` before forwarding to the preview card. Error results still surface to the model for retries — only the UI suppresses the blank-card flicker.

**Build infra.** Notarized macOS bundle now ships in TWO flavors:
  • `foxora_4.0.2_aarch64.dmg` — Apple Silicon native, ~smaller, recommended.
  • `foxora_4.0.2_universal.dmg` — fat .app via `lipo`, runs on Intel + Apple Silicon.

Both styled with the dmg-background.png Finder layout, both signed with Developer ID, both notarized + stapled, both gatekeeper-accepted.

— xBesh Labs, LLC
