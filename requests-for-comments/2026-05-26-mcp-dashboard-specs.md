# Request for comments: Agent friendly insight and dashboard MCP

## Problem statement

Agents are bad at building PostHog dashboards. For a dashboard with 5 insights, they need 12+ calls to the PostHog MCP. Run all 5 queries, turn them into insights, and lay them out.

If an agent knows what insights it wants and what queries those insights should run, it should be able to one shot the entire dashboard in a single tool call. Today it cannot.

The MCP is UI/API shaped. Not agent shaped.

The proposal in this RFC, the MCP tool for dashboards and insights accepts a JSON bundle whose objects are the existing Terraform resource shapes (`posthog_dashboard`, `posthog_insight`, `posthog_dashboard_layout`). One document, one call, three resources written transactionally, server side. The agent never runs `terraform`. We do not invent a new spec format.

## What the MCP looks like today

Here is the current flow for a 5 tile dashboard, derived from [`posthog.com/docs/model-context-protocol`](https://posthog.com/docs/model-context-protocol), the generated tool schemas in `services/mcp/src/tools/generated/`, and a real wizard session that built an "Analytics basics" dashboard.

1. Call `dashboard-create` with a name. Get back a dashboard id.
2. For each insight, call `insight-create` with a `query` blob and `dashboards: [dashboard_id]`. Get back an insight id and a tile id.
3. Call `dashboard-reorder-tiles` with the tile ids in the order you want.

That is at least N plus 2 calls for N insights. Each step can fail and leave partial state.

### Example: A real wizard session

I traced one real wizard run that asked an agent to build a 5 tile "Analytics basics" dashboard. The MCP commands actually issued before the dashboard was done.

- `search dashboard`, `search query-funnel` to find the tools.
- `info dashboard-create`, `info insight-create`, `info query-trends`, `info query-funnel`, 4 distinct `info` calls to learn the schemas.
- `schema query-trends`, `schema query-funnel` to dig further into the query shapes.
- `call dashboard-create` once.
- `call insight-create` five times, once per insight.
- `call dashboard-reorder-tiles` zero times. The agent gave up on layout.

The agent spent more calls discovering tool schemas than it spent doing work. It still failed to lay tiles out. The layout step is so awkward the agent skipped it. The user got a working dashboard with random tile order.

This is the actual `dashboard-create` call the agent made.

```json
{
  "name": "Analytics basics",
  "description": "Key business metrics: registrations, subscriptions, project activity, and churn.",
  "delete_insights": false
}
```

`delete_insights` is `required` even on create. The docstring says it controls behavior at delete time. The agent has to set it on create anyway.

This is the actual `insight-create` call for the funnel tile.

```json
{
  "name": "Subscription conversion funnel",
  "description": "Conversion from registration through subscription start to completed checkout",
  "dashboards": [1631241],
  "query": {
    "kind": "InsightVizNode",
    "source": {
      "kind": "FunnelsQuery",
      "series": [
        { "kind": "EventsNode", "event": "user_registered", "name": "Registered" },
        { "kind": "EventsNode", "event": "subscription_started", "name": "Started subscription" },
        { "kind": "EventsNode", "event": "subscription_checkout_completed", "name": "Checkout completed" }
      ],
      "dateRange": { "date_from": "-90d" },
      "funnelsFilter": {
        "funnelOrderType": "ordered",
        "funnelVizType": "steps",
        "funnelWindowInterval": 14,
        "funnelWindowIntervalUnit": "day"
      }
    }
  }
}
```

This is the actual `insight-create` call for a single series trend.

```json
{
  "name": "New user registrations",
  "description": "Daily count of new user registrations over the last 30 days",
  "dashboards": [1631241],
  "query": {
    "kind": "InsightVizNode",
    "source": {
      "kind": "TrendsQuery",
      "series": [
        { "kind": "EventsNode", "event": "user_registered", "name": "Registrations", "math": "total" }
      ],
      "dateRange": { "date_from": "-30d" },
      "interval": "day",
      "trendsFilter": { "display": "ActionsLineGraph" }
    }
  }
}
```

This is the actual `insight-create` call for a two series trend.

```json
{
  "name": "Subscriptions started vs canceled",
  "description": "Compare subscription starts against cancellations over the last 90 days",
  "dashboards": [1631241],
  "query": {
    "kind": "InsightVizNode",
    "source": {
      "kind": "TrendsQuery",
      "series": [
        { "kind": "EventsNode", "event": "subscription_started", "name": "Started", "math": "total" },
        { "kind": "EventsNode", "event": "subscription_canceled", "name": "Canceled", "math": "total" }
      ],
      "dateRange": { "date_from": "-90d" },
      "interval": "week",
      "trendsFilter": { "display": "ActionsLineGraph" }
    }
  }
}
```

A few things jump out. The query is nested four levels deep before you reach a meaningful field. `kind: "InsightVizNode"` wraps a `source` with `kind: "FunnelsQuery"` which wraps `series` with `kind: "EventsNode"`. None of that depth helps the agent. It is just ceremony.

The dashboard id `1631241` is hard coded in every insight call. The agent learned it from the `dashboard-create` response and had to thread it through. Forget the field and the insight is unattached. Type the wrong id and the insight ends up on **someone else's dashboard**. This name is also just not human readable.

### Concrete problems

**The query payload is untyped.** `insight-create` accepts the query as `source: z.record(z.string(), z.unknown())`. The only hint is a docstring that says "Product analtycs query objects like TrendsQuery, FunnelsQuery, RetentionQuery, PathsQuery, StickinessQuery, LifecycleQuery". The agent has no shape to fill in, so it guesses, and guesses wrong.

**The dashboard link is a footgun.** The `dashboards` field on `insight-create` is described as "This is a full replacement, always include all existing dashboard IDs when adding a new one." If the agent forgets, the insight gets ripped off every dashboard it was on. This is the kind of foot gun you cannot fix with prompting.

**Layout is a separate API call.** You cannot set tile positions during `dashboard-create`. You have to create the dashboard, create each insight, learn the tile ids that come back, then call `dashboard-reorder-tiles`. The agent has to thread ids through three calls.

**No HogQL first path.** `DataVisualizationNode` exists for raw HogQL queries but is buried in a union with `InsightVizNode`. Most agent generated insights would be easier to express as a HogQL SELECT plus a display type, but the schema does not surface that.

**No transactionality.** If insight 3 of 5 fails, the agent now owns a half built dashboard with 2 orphaned insights and has to clean up. Most agents will not.

**No public schema docs.** The docs page lists 200 plus tool names with one line descriptions. There are no schemas, no examples, and no link to where the query types are defined. The agent cannot read its way to a correct call.

### Cost issues

Context window accumulation maths:

The current conversation size increases linearly, but because each turn, the agent will send the entire accumulated conversation up to now to the API again, the cumulative tokens process increases quadratically for every extra turn.

Here’s some quick maths:

```
N + (N+K) + (N+2K) + ... + (N+TK) ≈ T*N + K*T²/2
```

Each wasted round gets mega expensive in tokens and time. More MCP tools also just adds pollution.

## The MCP looks like the API

The current MCP tools are a thin wrapper over the REST API. That API was designed for a single page app frontend, not for an agent. It assumes a human at a keyboard with affordances the agent does not have.

- A human can click between pages, so a dashboard, an insight, and a layout are three different endpoints. The agent has to thread state between them.
- A human can see partial state and recover, so partial writes are fine. The agent cannot, so partial writes leave junk.
- A human can hover over a field to see what it means, so the API does not need rich docstrings. The agent only has the schema, so the missing docs hurt it most.
- A human can iterate with the UI showing live previews, so the query payload can be a free form blob the UI builds piece by piece. The agent has no UI, so the blob is unauthored.
- A human can copy a dashboard id from the URL bar and paste it. The agent can do this too, but every paste is another chance to get it wrong.

The MCP today inherits all of this because it is generated from the same OpenAPI spec as the rest of the API surface. That is convenient for us but wrong for the agent. **The MCP should be shaped for agents, not for the human UI.** Agents need one shot transactions, full validation up front, self describing schemas, and clear errors. They do not need pagination, partial updates, or progressive disclosure.

This RFC takes that view as the starting point. The goal is not to add another tool that wraps another endpoint. The goal is to expose a surface that matches how the agent actually works.

## Prior art

### Grafana dashboard JSON

Grafana dashboards are a single JSON document. Panels are listed inline, each panel has its own `targets` (queries), and the layout sits inside each panel as `gridPos: {x, y, w, h}`. Provisioning lets you commit a dashboard JSON to a folder and Grafana reconciles. The shape is roughly:

```json
{
  "title": "API health",
  "panels": [
    {
      "title": "Request rate",
      "type": "graph",
      "gridPos": { "x": 0, "y": 0, "w": 12, "h": 8 },
      "targets": [{ "expr": "rate(http_requests_total[1m])" }]
    }
  ]
}
```

Worth considering. Layout lives on the panel, not in a separate reorder step. Queries live on the panel. The whole dashboard is one document. A human can hand author it.

Worth being careful about. The full Grafana schema has accreted a lot of options over the years. Many panels carry config whose purpose is no longer obvious. A smaller starting surface that grows on demand seems safer.

### Datadog dashboard JSON

Same idea as Grafana. A dashboard is a JSON document with a `widgets` array. Each widget has `definition` (query and viz) and `layout` (x, y, width, height). Datadog exposes a public REST API to apply this in one call.

### Looker LookML dashboards

Looker writes dashboards as `.dashboard.lookml` files. Elements are named, and tiles reference elements by name. References are strings, not ids. The Looker server resolves names at apply time. This is the pattern we want for `insight_ref`.

```lookml
- dashboard: orders_overview
  elements:
    - name: orders_per_day
      type: looker_line
      explore_source:
        - model: ecommerce
          explore: orders
      row: 0
      col: 0
      width: 12
      height: 6
```

Worth considering. Named references between insights and layout, resolved on the server side, rather than threading numeric ids through tool calls.

### PostHog's own Terraform provider

The most directly relevant prior art. PostHog already ships [`terraform-provider-posthog`](https://github.com/PostHog/terraform-provider-posthog) with three resources for this exact problem.

- `posthog_dashboard`, just `name`, `description`, `pinned`, `tags`. No tiles.
- `posthog_insight`, with a required `query_json` raw JSON string, plus a `dashboard_ids` set.
- `posthog_dashboard_layout`, a fully authoritative resource that manages all tiles on a dashboard. Mixes insight tiles and markdown text tiles. Each tile has `layouts_json` with `sm` and `xs` breakpoint keys for responsive widths. Unmanaged tiles get their layouts cleared.

A real example from the provider docs.

```terraform
resource "posthog_dashboard" "engineering" {
  name = "Engineering Metrics"
}

resource "posthog_insight" "pageviews" {
  name = "Page Views"
  query_json = jsonencode({
    kind = "InsightVizNode"
    source = {
      kind   = "TrendsQuery"
      series = [{ kind = "EventsNode", event = "$pageview", math = "total" }]
    }
  })
  dashboard_ids = [posthog_dashboard.engineering.id]
}

resource "posthog_dashboard_layout" "engineering" {
  dashboard_id = posthog_dashboard.engineering.id

  tiles = [
    { text_body    = "## Key Metrics",
      layouts_json = jsonencode({ sm = { x = 0, y = 0, w = 12, h = 1 } }) },
    { insight_id   = posthog_insight.pageviews.id,
      layouts_json = jsonencode({ sm = { x = 0, y = 1, w = 6, h = 4 } }),
      color = "blue", show_description = false },
  ]
}
```

## Goals

Shape the surface for the agent, not for the API or the human UI.

1. One tool call creates a dashboard, its insights, and its layout.
2. The agent gets a typed query shape that it can fill in, not a free form blob.
3. HogQL is a first class authoring path for insights. Agents speak SQL, not our insight DSL.
4. Errors come back in a shape the agent can act on without re prompting from a human.
5. The agent can do granular layout changes through the spec.
6. The spec format is documented well enough that a human can write one by hand, but the priority is the agent.

## Proposed design (for discussion)

Most of this already exists in the Terraform provider. The `posthog_dashboard`, `posthog_insight`, and `posthog_dashboard_layout` resources cover the data model, the validation, and the authoritative reconciliation. What we are adding is small.

- A thin MCP wrapper that bundles the three resources into one payload and resolves named refs server side.
- A typed `query` field on `posthog_insight` so the agent does not write `InsightVizNode` / `TrendsQuery` / `EventsNode` by hand. Additive on the TF side.

That is it. Everything below is the bundle shape and the one new field.

### The bundle shape

One MCP tool, one JSON payload, three TF resources written transactionally. Field names inside each block match `posthog_dashboard`, `posthog_insight`, and `posthog_dashboard_layout`.

```yaml
PosthogBundle:
  dashboard:                # fields from posthog_dashboard
    name: string
    description: string
    pinned: boolean
    tags: string[]
  insights:                 # map keyed by local ref string
    <ref>:                  # e.g. "weekly_active_users"
      name: string
      description: string
      tags: string[]
      query: InsightQuery   # typed, see below. Compiles to posthog_insight.query_json
  layout:                   # fields from posthog_dashboard_layout
    tiles:
      - insight: <ref>      # references a key in insights{}
        layouts:            # same shape as posthog_dashboard_layout.tiles[].layouts_json
          sm: { x: int, y: int, w: int, h: int }
          xs: { x: int, y: int, w: int, h: int }
        color: string
        show_description: boolean
      - text_body: string   # markdown text tile, max 4000 chars
        layouts:
          sm: { x: int, y: int, w: int, h: int }
```

Two things the MCP wrapper adds on top of the raw TF resource shapes.

- Named refs (`insights.weekly_active_users`) in place of TF interpolation (`${posthog_insight.weekly_active_users.id}`). The agent writes the name. The server resolves it.
- One shot bundled apply. The TF provider needs three resources with `depends_on`. The MCP wrapper takes one document and orchestrates the writes server side, transactionally.

### The single tool

```
posthog-bundle-apply:
  bundle: PosthogBundle
  mode: "create" | "update" | "upsert"
  dashboard_id: number  # required for update and upsert
```

Behavior matches `posthog_dashboard_layout`'s authoritative semantics. `update` reconciles the existing dashboard, insights, and layout to match the bundle. Tiles that fall out of the bundle are removed. Either everything writes, or nothing does.

### Query, the one change to the resource shapes

The current `posthog_insight.query_json` field is a raw JSON string. The agent has to embed an `InsightVizNode` wrapping a `TrendsQuery` wrapping `EventsNode`. We add a typed `query` field on `posthog_insight` that compiles down to `query_json`. The agent friendly shape.

```yaml
InsightQuery:
  oneOf:
    - HogQLInsight       # raw SQL plus visualization spec, the agent's first choice
    - TrendsInsight      # event series over time
    - FunnelInsight      # conversion
    - RetentionInsight   # cohort retention
    - PathsInsight       # user paths
    - StickinessInsight
    - LifecycleInsight

HogQLInsight:
  kind: "HogQL"
  sql: string
  display: "BoldNumber" | "LineGraph" | "Bar" | "StackedBar" | "Area" | "Table" | "Heatmap"
  x_axis_column: string
  y_axis_columns: string[]
  breakdown_column: string
  goal_lines:
    - label: string
      value: number
```

The agent already knows how to write SQL. SQL is a stable, documented contract. If the agent can express the question in HogQL, it does not need to learn our insight DSL.

This change is additive on the TF side. The Terraform provider also accepts `query` and compiles to `query_json` the same way. One query DSL across MCP and TF.

Open for discussion. Do we even need the typed kinds, or is HogQL enough for v1? Funnels and Retention are awkward to express in raw SQL. Trends is a thin layer on `count() GROUP BY date`. We could ship HogQL only and add typed kinds when we see real demand.

### Docs

There is no parallel docs page. The canonical reference is the existing Terraform provider docs at `registry.terraform.io/providers/PostHog/posthog`. The MCP docs page links there, plus a short page that shows the bundle wrapper, the named ref convention, and the `query` field addition.

Agents are really good at following examples. We should build a library of common shapes and patterns. Most dashboards people need start the same way, anyway.

### Validation and feedback

Validate in three stages, with agent and human friendly error output at each stage.

1. **Schema validation.** The bundle is validated against the TF resource schemas. Errors point to the exact field path and say what was expected.
   ```
   insights.weekly_active_users.query.sql: required, got undefined
   layout.tiles[0].insight: "monthly_signups" does not match any key in insights{}
   ```
2. **Reference resolution.** Every `insight` ref in `layout.tiles` must resolve to a key in `insights`. Errors list the offenders with a `did_you_mean` suggestion when there is a close match.
3. **Query validation.** Each insight query is sent through the existing HogQL validator before any writes happen. Errors include the original SQL, the parse error, and a pointer to the column or line that failed.

If validation fails, no writes happen, and the response includes every error at once.

Successful responses include the resolved ids and URLs.

```yaml
PosthogBundleApplyResponse:
  dashboard:
    id: number
    url: string
  insights:
    <ref>:
      id: number
      short_id: string
      url: string
      tile_id: number
  warnings: Warning[]
```

### A concrete example, the same "Analytics basics" dashboard, one call

Here is what the wizard session above looks like as a single `posthog-bundle-apply` call. Same 5 insights, same data, but the agent writes one bundle instead of running 6 tool calls and giving up on layout.

```json
{
  "dashboard": {
    "name": "Analytics basics",
    "description": "Key business metrics: registrations, subscriptions, project activity, and churn."
  },
  "insights": {
    "signup_funnel": {
      "name": "Subscription conversion funnel",
      "description": "Conversion from registration through subscription start to completed checkout.",
      "query": {
        "kind": "Funnel",
        "date_from": "-90d",
        "steps": [
          { "event": "user_registered", "name": "Registered" },
          { "event": "subscription_started", "name": "Started subscription" },
          { "event": "subscription_checkout_completed", "name": "Checkout completed" }
        ],
        "order": "ordered",
        "window": { "value": 14, "unit": "day" }
      }
    },
    "weekly_active_users": {
      "name": "Weekly active users",
      "description": "Unique users who logged in per week over the last 90 days.",
      "query": {
        "kind": "HogQL",
        "sql": "SELECT toStartOfWeek(timestamp) AS week, count(DISTINCT person_id) AS active_users FROM events WHERE event = 'user_logged_in' AND timestamp > now() - INTERVAL 90 DAY GROUP BY week ORDER BY week",
        "display": "LineGraph",
        "x_axis_column": "week",
        "y_axis_columns": ["active_users"]
      }
    },
    "registrations_daily": {
      "name": "New user registrations",
      "description": "Daily count of new user registrations over the last 30 days.",
      "query": {
        "kind": "HogQL",
        "sql": "SELECT toDate(timestamp) AS day, count() AS registrations FROM events WHERE event = 'user_registered' AND timestamp > now() - INTERVAL 30 DAY GROUP BY day ORDER BY day",
        "display": "LineGraph",
        "x_axis_column": "day",
        "y_axis_columns": ["registrations"]
      }
    },
    "subs_started_vs_canceled": {
      "name": "Subscriptions started vs canceled",
      "description": "Compare subscription starts against cancellations over the last 90 days.",
      "query": {
        "kind": "HogQL",
        "sql": "SELECT toStartOfWeek(timestamp) AS week, countIf(event = 'subscription_started') AS started, countIf(event = 'subscription_canceled') AS canceled FROM events WHERE event IN ('subscription_started', 'subscription_canceled') AND timestamp > now() - INTERVAL 90 DAY GROUP BY week ORDER BY week",
        "display": "LineGraph",
        "x_axis_column": "week",
        "y_axis_columns": ["started", "canceled"]
      }
    },
    "project_activity": {
      "name": "Project activity",
      "description": "Project creation, updates, and deletions over the last 30 days.",
      "query": {
        "kind": "HogQL",
        "sql": "SELECT toDate(timestamp) AS day, countIf(event = 'project_created') AS created, countIf(event = 'project_updated') AS updated, countIf(event = 'project_deleted') AS deleted FROM events WHERE event IN ('project_created', 'project_updated', 'project_deleted') AND timestamp > now() - INTERVAL 30 DAY GROUP BY day ORDER BY day",
        "display": "LineGraph",
        "x_axis_column": "day",
        "y_axis_columns": ["created", "updated", "deleted"]
      }
    }
  },
  "layout": {
    "tiles": [
      { "text_body": "## Conversion and engagement",
        "layouts": { "sm": { "x": 0, "y": 0, "w": 12, "h": 1 } } },
      { "insight": "signup_funnel",
        "layouts": { "sm": { "x": 0, "y": 1, "w": 6, "h": 5 } } },
      { "insight": "weekly_active_users",
        "layouts": { "sm": { "x": 6, "y": 1, "w": 6, "h": 5 } } },
      { "text_body": "## Growth and churn",
        "layouts": { "sm": { "x": 0, "y": 6, "w": 12, "h": 1 } } },
      { "insight": "registrations_daily",
        "layouts": { "sm": { "x": 0, "y": 7, "w": 6, "h": 4 } } },
      { "insight": "subs_started_vs_canceled",
        "layouts": { "sm": { "x": 6, "y": 7, "w": 6, "h": 4 } } },
      { "insight": "project_activity",
        "layouts": { "sm": { "x": 0, "y": 11, "w": 12, "h": 4 } } }
    ]
  }
}
```

One tool call. Five insights, two markdown headers, seven tiles total, all laid out. Every field inside `dashboard`, each `insights[ref]`, and each `layout.tiles[*]` matches the existing Terraform resource. The only thing the MCP wrapper adds is the named refs (`signup_funnel`, `weekly_active_users`, etc.) and the bundled apply. The funnel uses the typed `kind: "Funnel"` shape. The four trend tiles use `kind: "HogQL"`.

Compare that to the wizard session. The agent there wrote roughly the same number of bytes per tile, but distributed across 5 separate calls, with `kind: "InsightVizNode"` and `kind: "TrendsQuery"` and `kind: "EventsNode"` ceremony around every series, plus dashboard id threading on every call. No headers, because adding them would have been yet another tool call. The bundle collapses all of that into one document the agent writes once.

If anything in the bundle is wrong, the agent gets the full list of problems back in one response, not five interleaved errors.
