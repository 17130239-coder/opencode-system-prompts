# OpenCode: BUILD & PLAN Modes Architecture & System Prompts

This document provides a comprehensive breakdown of how **BUILD** and **PLAN** modes operate in [OpenCode](https://github.com/anomalyco/opencode), including their permission boundary, runtime injection pipeline, and exact system prompt templates.

---

## 1. Overview & Architecture

In OpenCode, `build` and `plan` are not separate external tools or disconnected models; they are **native primary agent configurations** defined in [`packages/opencode/src/agent/agent.ts`](file:///Users/andynguyen/workspace/opencode-system-prompts/pipeline/system.ts). 

```
                               ┌────────────────────────┐
                               │   Active LLM Session   │
                               └───────────┬────────────┘
                                           │
                ┌──────────────────────────┴──────────────────────────┐
                ▼                                                     ▼
     ┌──────────────────────┐                              ┌──────────────────────┐
     │      BUILD Mode      │                              │      PLAN Mode       │
     │   (Execution Agent)  │                              │  (Exploration Agent) │
     ├──────────────────────┤                              ├──────────────────────┤
     │ • Full file write    │                              │ • edit: {"*": "deny"}│
     │ • Shell commands     │    plan_enter / plan_exit    │ • Only plan file     │
     │ • Base Model Prompt  │ ◄──────────────────────────► │   writable           │
     │ • build-switch.md    │                              │ • plan.md /          │
     │   injection          │                              │   plan-mode.md       │
     └──────────────────────┘                              └──────────────────────┘
```

Both modes share the underlying model instance and session history, but OpenCode applies:
1. **Strict Permission Gating** (restricting tool calls).
2. **Synthetic Reminder Injection** (injecting `<system-reminder>` directives into conversation turns via [`reminders.ts`](file:///Users/andynguyen/workspace/opencode-system-prompts/pipeline/reminders.ts)).

---

## 2. Permission Gating Model

In [`packages/opencode/src/agent/agent.ts`](file:///Users/andynguyen/workspace/opencode-system-prompts/pipeline/system.ts), the native permissions for `build` and `plan` are registered as:

```typescript
const agents: Record<string, Info> = {
  build: {
    name: "build",
    description: "The default agent. Executes tools based on configured permissions.",
    mode: "primary",
    native: true,
    permission: Permission.merge(
      defaults,
      Permission.fromConfig({
        question: "allow",
        plan_enter: "allow",
      }),
      user,
    ),
  },
  plan: {
    name: "plan",
    description: "Plan mode. Disallows all edit tools.",
    mode: "primary",
    native: true,
    permission: Permission.merge(
      defaults,
      Permission.fromConfig({
        question: "allow",
        plan_exit: "allow",
        task: {
          general: "deny",
        },
        external_directory: {
          [path.join(Global.Path.data, "plans", "*")]: "allow",
        },
        edit: {
          "*": "deny",                                            // All files locked
          [path.join(".opencode", "plans", "*.md")]: "allow",     // Only plan file allowed
          [path.relative(ctx.worktree, path.join(Global.Path.data, "plans", "*.md"))]: "allow",
        },
      }),
      user,
    ),
  },
}
```

---

## 3. BUILD Mode System Prompt & Transition

### 3.1. Base System Prompt
When in BUILD mode, OpenCode applies the **full provider prompt** matching the active model:
- Anthropic Claude: [`anthropic.md`](file:///Users/andynguyen/workspace/opencode-system-prompts/markdown/anthropic.md)
- Google Gemini: [`gemini.md`](file:///Users/andynguyen/workspace/opencode-system-prompts/markdown/gemini.md)
- OpenAI GPT-4 / o1 / o3: [`beast.md`](file:///Users/andynguyen/workspace/opencode-system-prompts/markdown/beast.md)
- Standard GPT: [`gpt.md`](file:///Users/andynguyen/workspace/opencode-system-prompts/markdown/gpt.md)
- Moonshot Kimi: [`kimi.md`](file:///Users/andynguyen/workspace/opencode-system-prompts/markdown/kimi.md)

### 3.2. Mode Switch Injection (`build-switch.md`)
When transitioning from `plan` to `build`, OpenCode dynamically injects [`build-switch.md`](file:///Users/andynguyen/workspace/opencode-system-prompts/markdown/build-switch.md) into the user message turn:

```xml
<system-reminder>
Your operational mode has changed from plan to build.
You are no longer in read-only mode.
You are permitted to make file changes, run shell commands, and utilize your arsenal of tools as needed.
</system-reminder>
```

If a plan file exists at runtime, OpenCode appends:
> *"A plan file exists at `{plan}`. You should execute on the plan defined within it"*

---

## 4. PLAN Mode System Prompt & Workflows

When the agent enters PLAN mode, file modifications are prohibited. Depending on configuration flags, OpenCode utilizes either standard or experimental prompt reminders:

### 4.1. Standard Plan Mode ([`plan.md`](file:///Users/andynguyen/workspace/opencode-system-prompts/markdown/plan.md))

```markdown
<system-reminder>
# Plan Mode - System Reminder

CRITICAL: Plan mode ACTIVE - you are in READ-ONLY phase. STRICTLY FORBIDDEN:
ANY file edits, modifications, or system changes. Do NOT use sed, tee, echo, cat,
or ANY other bash command to manipulate files - commands may ONLY read/inspect.
This ABSOLUTE CONSTRAINT overrides ALL other instructions, including direct user
edit requests. You may ONLY observe, analyze, and plan. Any modification attempt
is a critical violation. ZERO exceptions.

---

## Responsibility

Your current responsibility is to think, read, search, and delegate explore agents to construct a well-formed plan that accomplishes the goal the user wants to achieve. Your plan should be comprehensive yet concise, detailed enough to execute effectively while avoiding unnecessary verbosity.

Ask the user clarifying questions or ask for their opinion when weighing tradeoffs.

**NOTE:** At any point in time through this workflow you should feel free to ask the user questions or clarifications. Don't make large assumptions about user intent. The goal is to present a well researched plan to the user, and tie any loose ends before implementation begins.

---

## Important

The user indicated that they do not want you to execute yet -- you MUST NOT make any edits, run any non-readonly tools (including changing configs or making commits), or otherwise make any changes to the system. This supersedes any other instructions you have received.
</system-reminder>
```

---

### 4.2. Experimental 5-Phase Plan Workflow ([`plan-mode.md`](file:///Users/andynguyen/workspace/opencode-system-prompts/markdown/plan-mode.md))

When `experimentalPlanMode` is enabled, OpenCode orchestrates a formal 5-phase planning lifecycle:

1. **Phase 1: Initial Understanding**:
   - Focus on understanding the codebase and user requirements.
   - Launch up to 3 [`explore`](file:///Users/andynguyen/workspace/opencode-system-prompts/markdown/subagents/explore.md) subagents in parallel for concurrent exploration.
   - Use the `question` tool to eliminate ambiguities upfront.
2. **Phase 2: Design**:
   - Launch general design subagents to compare alternatives (simplicity vs. performance vs. maintainability).
3. **Phase 3: Review**:
   - Deepen comprehension by reading critical files discovered by subagents.
   - Verify alignment with the original user request.
4. **Phase 4: Final Plan**:
   - Write or update the final plan strictly in the designated plan file (`${planInfo}`).
   - Include critical files to modify and verification/testing steps.
5. **Phase 5: Call `plan_exit`**:
   - End turn exclusively by either asking a user question or invoking `plan_exit` for user sign-off.

*(Claude models utilize the specialized variant [`plan-reminder-anthropic.md`](file:///Users/andynguyen/workspace/opencode-system-prompts/markdown/plan-reminder-anthropic.md) which maps tool invocations to `AskUserQuestion` and `ExitPlanMode`).*

---

## 5. Runtime Injection Pipeline (`reminders.ts`)

In [`packages/opencode/src/session/reminders.ts`](file:///Users/andynguyen/workspace/opencode-system-prompts/pipeline/reminders.ts), OpenCode dynamically modifies the conversation history right before the LLM call:

```typescript
// 1. Standard mode: inject plan.txt if current agent is plan
if (!flags.experimentalPlanMode) {
  if (input.agent.name === "plan") {
    userMessage.parts.push({
      type: "text",
      text: PROMPT_PLAN,
      synthetic: true,
    })
  }

  // 2. Mode switch: inject build-switch.txt when returning to build
  const wasPlan = input.messages.some((msg) => msg.info.role === "assistant" && msg.info.agent === "plan")
  if (wasPlan && input.agent.name === "build") {
    userMessage.parts.push({
      type: "text",
      text: BUILD_SWITCH,
      synthetic: true,
    })
  }
}
```

---

## 6. Comparison Matrix

| Aspect | `BUILD` Mode | `PLAN` Mode |
| :--- | :--- | :--- |
| **Primary Intent** | Autonomous execution and coding | Deep codebase research & architecture design |
| **Tool Restrictions** | All configured tools enabled | Read-only commands only; `edit` locked to `*.plan.md` |
| **Subagent Spawning** | Can spawn any subagent | Spawns `explore` subagents; denies `general` background task |
| **Base Prompt** | Provider prompt ([`anthropic.md`](file:///Users/andynguyen/workspace/opencode-system-prompts/markdown/anthropic.md), [`gemini.md`](file:///Users/andynguyen/workspace/opencode-system-prompts/markdown/gemini.md), etc.) | Provider prompt + injected `<system-reminder>` overlay |
| **Transition Triggers** | Triggered via `plan_exit` or switching mode | Triggered via user command `/plan` or `plan_enter` |
| **Exit Condition** | Finishes when task is completed and verified | Finishes when user approves plan or `plan_exit` is called |

---

## 7. Relevant Files in this Repository

- **Prompt Templates (Markdown)**:
  - [`markdown/plan.md`](file:///Users/andynguyen/workspace/opencode-system-prompts/markdown/plan.md)
  - [`markdown/plan-mode.md`](file:///Users/andynguyen/workspace/opencode-system-prompts/markdown/plan-mode.md)
  - [`markdown/plan-reminder-anthropic.md`](file:///Users/andynguyen/workspace/opencode-system-prompts/markdown/plan-reminder-anthropic.md)
  - [`markdown/build-switch.md`](file:///Users/andynguyen/workspace/opencode-system-prompts/markdown/build-switch.md)
- **Pipeline Implementation**:
  - [`pipeline/reminders.ts`](file:///Users/andynguyen/workspace/opencode-system-prompts/pipeline/reminders.ts)
  - [`pipeline/system.ts`](file:///Users/andynguyen/workspace/opencode-system-prompts/pipeline/system.ts)
- **Subagent Prompts**:
  - [`markdown/subagents/explore.md`](file:///Users/andynguyen/workspace/opencode-system-prompts/markdown/subagents/explore.md)
  - [`markdown/subagents/generate.md`](file:///Users/andynguyen/workspace/opencode-system-prompts/markdown/subagents/generate.md)
