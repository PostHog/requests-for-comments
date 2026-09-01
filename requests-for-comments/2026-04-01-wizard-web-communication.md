# Request for comments: PostHog Wizard CLI → Web App Communication

**Decision maker:** Vincent Ge

## Problem statement

We're pushing the Wizard (`npx @posthog/wizard`) to become the default entry point for PostHog onboarding. It detects the framework, authenticates via OAuth, runs a Claude agent that installs the SDK and modifies code using MCP skills, optionally installs the PostHog MCP server into editor clients, and reports success.

Right now, the wizard runs entirely in the terminal. The web app has no idea it's happening, even if you just started in the web app and ran the command there.

1. **No visibility** — The wizard has a rich TUI (Ink-based React terminal UI) with real-time status, task lists, and screen-by-screen progress. None of this is visible in the PostHog web app.
2. **No continuity** — The wizard finishes, the user opens PostHog in their browser, and they see the same generic onboarding flow as someone who hasn't run the wizard. The web app doesn't know they already installed `posthog-js`, configured session replay, or set up environment variables.
3. **No handoff** — The wizard creates insights and dashboards, but the web app doesn't know about them.
4. **No information preserved about the taxonomy** — The wizard is the best source of truth for the taxonomy of the user's app and even business. Information super valuable to PostHog Code and PostHog AI should be preserved, event schema should be preserved, etc.

We want the wizard to push its state to the PostHog API so the web app can show live progress, adapt onboarding, and capture structured diagnostics.

## Context

### How the wizard actually works today

The wizard is built on the Claude Agent SDK. Here's the real flow as implemented in `bin.ts` → `run.ts` → `agent-runner.ts`:

![Wizard flow](/images/2026-04-01-wizard-web-communication/wizard-flow.png)

Once the agent starts, it runs a single query. We hook into it to push state to CLI TUI. There is no input mechanism and the app is entirely controlled by the agent.

### What we already have

**`WizardSession`** (`src/lib/wizard-session.ts`) — tracks everything: `RunPhase` (Idle/Running/Completed/Error), integration, credentials, framework context, discovered features, MCP outcome, health check results, outro data. This is the state we want to push.

**`WizardStore`** (`src/ui/tui/store.ts`) — a reactive nanostore wrapper around `WizardSession`. Emits change events, tracks screen transitions, and fires analytics on every transition.

The CLI lifecycle is straightforward:

1. Agent emits changes
2. Changes update the store
3. The UI renders the store

We hook into step 2 to also push state to the web app.

## Part 1 — Wizard side: exposing a task-stream interface

Ownership: Docs & Wizard team

This bit is the code we'll ship to expose the task-stream interface to the web app on the Wizard side.

### Architecture overview

![Architecture overview](/images/2026-04-01-wizard-web-communication/wizard-architecture.png)

### Wizard state

The state we push has two layers: the **run** itself and the **todos** inside it.

The run is tracked by `RunPhase` on the `WizardStore` — one of four values for the whole session:

```typescript
export enum RunPhase {
  Idle = 'idle',
  Running = 'running',
  Completed = 'completed',
  Error = 'error',
}
```

Inside a running session, the agent maintains a list of todos. Each todo has its own status:

```typescript
export enum TaskStatus {
  Pending = 'pending',
  InProgress = 'in_progress',
  Completed = 'completed',
  Failed = 'failed', // This is set only when the wizard proactively aborts the run.
  Cancelled = 'cancelled', // This is set only when the wizard is cancelled by the user. Or on timeout from the PostHog app.
}
```

The agent emits whole list updates that overwrite the previous list `TaskStatus[]`. The agent will expand scope and re-plan the todos mid-run. This means we'll emit full list state overwrites on every change.

### Hook points

We hook into the following points:

| Trigger | Wiring |
|--------|--------|
| The wizard agent starts a run | `WizardStore` phase change → session create (`RunPhase.Running`) |
| Anything the agent does during the run (plans events, writes todos, changes a todo's status, re-plans) | `TodoWrite` in message triggers → `setTodos()` → `syncTodos()` — full state overwrite |
| The wizard agent finishes or errors out | `WizardStore` phase change → `RunPhase.Completed` / `RunPhase.Error` → final push |

**Every `TodoWrite` intercept is a full overwrite of the list.** The agent is free to add, remove, rename, reorder, or completely re-plan the todos mid-run, and we just replace whatever we previously pushed with the new list — no diffing, no merging on our side. Consumers should treat each push as the new source of truth.

Within a single run the individual items still transition through the normal lifecycle the agent drives (`pending` → `in_progress` → `completed` or `failed`), so if a todo survives from one push to the next its status moves forward as you'd expect. But nothing guarantees a given todo id exists in the next push at all.

Each hook point emits a generic task-stream update — the onboarding wizard is the first consumer, but the schema itself is not onboarding-specific.

### Task stream schema

The wire payload the wizard pushes is intentionally generic. Onboarding is the first caller, but migrations, audits, single-task installs, and troubleshooting runs can all use the same transport with a different `workflow_id` / `skill_id` pair:

```typescript
interface TaskStreamUpdate {
  // session_id is always `${workflow_id}-${skill_id}-${started_at}`
  // e.g. "onboarding-posthog_integration-2026-04-17T14:32:10Z"
  session_id: string;
  workflow_id: 'onboarding' | 'migration' | 'audit' | '<other workflow names>' | string;
  skill_id: 'posthog_integration' | 'revenue_analytics_setup' | '<other skills>' | string;
  started_at: string;         // ISO 8601, matches the timestamp in session_id
  run_phase: 'idle' | 'running' | 'completed' | 'error';
  tasks: Array<{
    id: string;
    title: string;
    status: TaskStatus;
  }>;
  event_plan?: unknown;
  error?: { type: string; message: string };
}
```

Every run is a new `session_id`. The wizard never updates an old session — re-running `onboarding` + `posthog_integration` is a new row with a newer timestamp, not a mutation of the previous run. Consumers get the current view by picking the latest session for a given `(workflow_id, skill_id)` pair, and keep the older rows as run history.

The wizard only emits `workflow_id: 'onboarding'` payloads for v1, but the schema is designed so additional workflows and skills can plug in later without a migration.

### What the wizard will ship for V1

- **Sync on by default** — whenever the wizard has an OAuth session or API key, it pushes to the PostHog API with no extra flag needed. `--no-telemetry` is the escape hatch for users who explicitly want to avoid having various tracking like LLMA and other analytics PostHog collects.
- **Task stream push module** — a new `task-stream-push.ts` that hooks into `WizardStore.emitChange()` and debounces fire-and-forget pushes with the current task list, run phase, event plan, and `run_type`. Zero impact on the TUI — if the push fails, the wizard doesn't notice. Onboarding is the first consumer, but the module stays generic so future migration / audit / single-task runs reuse it.
- **Timestamping** — The wizard will attach UTC ISO 8601 timestamps to every payload sent. On conflict, latest update wins.
- **No opinions on transport beyond HTTP POST** — We'll ship the interface to implement any transport we want. We'll leave this open until we have an API/socket to hit on the web app side.

### What the wizard needs from PostHog

The only dependency the wizard has on PostHog is an HTTP/socket endpoint scoped to a project that accepts `TaskStreamUpdate` payloads and authorizes via the existing OAuth session or API key.

Anything beyond that — storage model, real-time fanout, UI — is Part 2.

---

## Part 2 — Suggested: PostHog web app onboarding handling

This part is a suggested implementation as discussed in the #quick-call with the growth team, not a commitment.

### Suggested API shape

- **Wizard sessions endpoint** — `POST /api/projects/{id}/wizard/sessions/` to create a session and persist the final state. State can be a JSONB column overwritten on each push. Since the agent will not emit a concrete list of todos, we'll need to persist the entire task stream in the session. This _may_ change in the future as we rework task queue. It _may_ become a known, fixed, finite list of todos.
- **Push transport over WebSockets/SSE** — Instead of the web app polling for active sessions, state updates stream to connected clients. The CRUD endpoint remains for session create and final-state persistence. Simpler end-to-end, and unblocks rendering wizard progress anywhere in the app (see Edwin / daniloc's sandboxed wizard idea).
- **Session expiry** — Mark sessions as abandoned if `run_phase = 'running'` and no update for 10+ minutes. We've caugh the wizard on compaction hanging for many minutes depending on the day. Anthropic chokes sometimes. 
- **Append-only run history** — Each wizard run is its own row. Session ids are formatted `<workflow_id>-<skill_id>-<timestamp>` (e.g. `posthog_integration-nextjs-2026-04-17T14:32:10Z`), so re-running the wizard against the same workflow and skill just creates a newer entry — there's no conflict resolution or merging, only "what's the latest run for this `<workflow_id, skill_id>` pair?". The web app reads the latest run when it needs a current view, and can walk the history for diagnostics or diff-mode-style features later.

### Suggested UI — wizard-aware onboarding

When the app receives signal that a wizard session has started, we can switch to displaying the wizard progress instead of the generic onboarding flow. This essentially replaces the existing `Install` flow.

As soon as the API sees the session-start signal, the web app opens (or pushes the user to) the onboarding page with live progress so the terminal → browser transition feels seamless rather than disjoint.

![Onboarding split screen](/images/2026-04-01-wizard-web-communication/wizard-onboarding-ui.png)

While this is running, we can take the user through the other steps such as:
- `Configure`
- `Session replay`
- `Import data`
- ... etc

We can split the screen into a progress list of what the wizard is doing, and the configuration steps that are available to the user.

The final step can be taking the user to the dashboards and insights that the wizard created.

**Error state** — When the wizard errors, or when an individual task emits `status: 'failed'`, show the error type, the failing task, and the last known state so support can diagnose without terminal screenshots.

### UI variants worth A/B testing

There isn't one obvious right answer for how aggressive the onboarding surface should be while the wizard runs. Three approaches worth trying, ideally behind an experiment:

1. **Full-screen onboarding** — user can't roam the app; we use the screen real estate to show live wizard progress, explain the events being instrumented, and tease what's possible once the data lands. Prompt for anything we couldn't capture at the end.
2. **No dedicated onboarding** — user roams the app freely; wizard progress lives in a persistent sidebar or floating window that's always visible but never blocking.
3. **Hybrid / resumable** — default to full-screen with a clear "skip" that drops the progress UI into the sidebar and lets the user come back to the full-screen view anytime.

Ship these behind a feature flag / experiment and let the data pick the winner.

While the wizard is installing, we also want to keep attention on the page. One idea worth trying is a "Hogtoks" feed — short vertical demo videos the user can scroll through instead of staring at a progress bar.

## Potential future work

The Wizard basically lets you do what ever you want to the user's code base. Once we have this V1, we can build so many more cool things. I'm inviting you all to think about the growth levers you'd like to build into this.

The Wizard needs a pretty good round of refactoring before we can really start messing with its behavior, especially with more input from the web app. But when that's possible, there are many super cool things we can do:

- **Wizard query refactors** — Split the big query into smaller tasks to do, like just installing the sdk, adding single events, doing insights, doing dashboards. This lets the workflow be a chain of skills, which might give us a stable set of `tasks` to be executed in sequence.
- **Migrations**: If they set up Product Analytics, and we detect they're using a competitor for feature flags, we can 1:1 migrate them. In the web app we can show them how much they'd save if they migrate. If they would like to try, we can just do it.
- **Two-way communication** — The wizard could poll the web app for commands, or the web app could push commands to the wizard. This would allow the wizard to wait for user input, or to be paused and resumed by the web app.
  - Example 1: On web, the user can tweak which events to instrument, what they look like, which insights and dashboards to create, etc.
  - Example 2: On web, if the user configures remote proxies or makes other config suggestions, we can sync this to the wizard. Maybe via polling.
- **Test correctness** — Fire test events from the wizard to the web app to verify the implementation. A richer version Sarah has been thinking about: replay-based verification where the wizard boots a dev server plus a headless browser, performs user flows, and checks the expected events land in the project.
- **Code review mode** — A web app-triggered run that reviews an existing PostHog integration for robustness and suggests fixes, rather than starting from scratch.
- **Diff mode** — Re-run the wizard months after the first install. It remembers the first run, recognizes new pages/routes you've added since then, and offers to instrument only the delta. Could eventually run as a background process watching for new commits.
- **Audit runs / single-task runs** — Same transport, different `run_type`. Lets us run the wizard for targeted jobs (one integration, one fix) without pretending it's a fresh onboarding.
- **Smooth web → wizard handoff** — Let the user kick off a wizard run from a web CTA with context already attached, instead of copying a command into a terminal cold.
- **In-wizard signup** — Let the user register an account (email + password or similar) from inside the wizard so they never have to touch the browser first. Open questions we'd need to answer before shipping this: how they pay, how we avoid signup spam, and how usage cost attribution works.
- **Taxonomy** — The wizard is the best source of truth for the taxonomy of the user's app and even business. Information super valuable to PostHog Code and PostHog AI should be preserved, event schema should be preserved, etc.
- **What if we run the wizard in a sandbox?** PostHog Code's gonna build these, I think we can entirely bypass the user's computer. This is already in the plans.
