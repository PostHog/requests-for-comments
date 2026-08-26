# Request for comments: Adopt relationship-based access control (ReBAC)

**Author:** Reece Jones
**Date:** 2026-08-13

## TLDR

Our access control model no longer scales with our access control needs. As we add access control to more places, we are increasingly relying on one-off hacks that slow development and break easily. Additionally, auditing existing access control logic has become more difficult because it is distributed throughout many different parts of the codebase. This RFC proposes we adopt Relationship-Based Access Control (ReBAC) as a general purpose access control engine to consolidate access control code, decouple logic from consumers, and increase auditability.

## Problem statement

The current access control implementation is hard to develop for, fragile, and difficult to audit, because of the following reasons.


### Access control logic spread across different places

The logic that evaluates access control rules is spread across many places in the codebase. For example:
- *Django viewsets and serializers*
- *HogQL AST parser and printer*
- *Filesystem API*
- *Search API*
- *Frontend access checks*
- *One-off database queries*

This:
1. Couples access control logic to other unrelated logic. For example, access control has become highly coupled to the HogQL implementation. This means that HogQL contributors must be explicitly aware of the access control implementation so queries perform as users expect.
2. Re-implements access control logic in multiple places. For example, access level resolution logic is implemented on both the frontend and backend and need to be kept in sync.

### Multiple access control systems

When our access control system was first developed, it was designed specifically around Django’s data-management patterns, and under the assumption that all resources would be public by default to the owning project. For example, the following product areas have had to implement their own authorization logic, separate from "access control":

- *PostHog AI (Tasks)* - Tasks are visible to the creating user and hidden to the rest of the team by default, however our access control implementation assumes the opposite: that objects are visible to the entire team by default.
- *Billing access control* - Billing lives at the organization-level, but access controls live at the project level, so access control here is implemented by a hacky feature flag that, when enabled, requires the admin organization role to see the billing page.
- *MCP* - We cannot set separate access control policies for MCP sessions
- *Sharing* - Sharing has its own authorization system that uses its own data model and ephemeral authentication tokens.
- *members_can_\* fields* - We store a lot of one-off columns like `members_can_invite` and `members_can_use_personal_api_keys` that are used to implement authorization policies in place.
- *API key scopes* - There is a mismatch between the scopes you can select when creating an API key and what we show in the access control settings UI.

### There is no general purpose authorization library

Right now authorization is handled different ways in different contexts, but there is no cohesive throughline. For example, tenant isolation for organization data is handled completely differently than checking if a user is allowed to create new dashboards.

In the age of agents this also means we cannot easily instruct agents to follow a specific authorization pattern. Authorization is so fractured and context-dependent that we cannot reliably tell an agent how to implement authorization when building new features. This has obvious security implications.

### Auditability

Right now if a user is having trouble accessing something, we have no way of easily figuring out why. Our best bet is to dump their access control rules from the database, feed it to an agent, and ask it to figure out why it isn't working. This is because access control evaluation happens in many different places, with different behaviors, rather than a single common code path.

Additionally, we do not have an audit trail that documents access control decisions. This means we cannot retroactively determine if a user was granted or denied access to a resource, let alone understand why. As we and our customers continue to scale, this will be important to document for security incident response.

### Slow development

For all the reasons mentioned above, developing anything touching access controls is slow. Small changes become complex. As the surface area expands, it gets harder to verify access controls behave as we expect. Some access controls we don't have a good idea how to even build.

## Proposal

### Vision

A cultural shift at PostHog where authorization is a primary concern, and there is one interface that developers across the company use for authorization checks.

### Proposal

I propose we adopt relationship-based access control (ReBAC) and repackage access control as a general purpose authorization library for PostHog developers within the monorepo.

### Success criteria

1. No customer's access is impacted by the change.
2. All authorization decisions are made in a single place.
3. We have audit logs documenting authorization decisions.
4. Authorization becomes standardized.

## Context

### Customer context

Customers keep asking for new access control features that we are unable or too slow to build, because of the inflexibility of our access control system.

### Modern architecture

Modern authorization architecture is typically divided into two components: a Policy Evaluation Point (PEP), and a Policy Decision Point (PDP). The PDP is responsible for answering the question "Can the user do this action?", whereas the PEP is responsible for enforcing the resolved access for the user.

For example, a viewset which returns `Dashboards` would be a PEP. The viewset would ask the PDP if the authenticated user can perform the API action. The PDP returns a boolean response, yes or no, and if denied, the PEP is responsible for terminating the request and responding with the correct HTTP response.

This decoupled architecture improves scalability, because changes to the access control rules are contained to the PDP. The PEP treats the PDP as a black box, which reduces the blast radius of access control logic changes. Additionally, modern authorization systems rely on domain specific languages (DSLs) and specialized data structures to model access control rules. This imposes a design constraint that forces developers to model access control as concise formal logic rather than application code.

See [ABAC](https://en.wikipedia.org/wiki/Attribute-based_authorization) and [Cedar](https://docs.cedarpolicy.com/) for further reading on this architecture.

### ReBAC

ReBAC solves the problem of access control by modeling access as a graph. A user can perform an action on an object if they have a relation, direct or indirect, to that object. This is a natural fit for us, because we already model access control through relations.

For example, a user can edit a dashboard if they are a project admin, or they are an editor for dashboards, or they have edit permissions for that specific dashboard. In this example, the user's access to edit the dashboard is defined by their relationships to the dashboard and the project.

### Implementations

[Zanzibar](https://storage.googleapis.com/gweb-research2023-media/pubtools/5068.pdf) is Google's in-house ReBAC implementation. It is used across hundreds of their products, including popular products like Google Docs. Zanzibar is the primary source of inspiration for nearly all modern ReBAC implementations.

[OpenFGA](https://openfga.dev/) is a ReBAC implementation that is a CNCF incubation project, maintained by Okta and Grafana. It seems to be the most popular ReBAC implementation right now due to its cloud-native approach.

[SpiceDB](https://authzed.com/spicedb) is an authorization database with support for ReBAC configuration. It is [open source](https://github.com/authzed/spicedb), under the Apache 2.0 License, and maintained by [AuthZed](https://authzed.com) who sell authorization as a service.

[Permify](https://permify.co/) is an [open source](https://github.com/Permify/permify) ReBAC implementation that, like OpenFGA, aims to be cloud-native. It is  licensed under the APGL-3 license and maintained by [FusionAuth](https://fusionauth.io/) who sell authorization as a service.

[Cedar](https://cedarpolicy.com) is an [open source]() policy engine that supports ReBAC authorization models. Unlike the other engines, it is very much bring your own batteries, requiring you to implement core ReBAC semantics. However, it can be embedded in existing programs and you provide your application code provides relationship data.

### Pros & Cons

**Pros**

- *Centralized logic* - All logic is colocated in one place, making it easier to maintain.
- *Audit logs* - With all authorization requests routed through a single interface, we can log authorization decisions for security review.
- *Simplified SDK* - Clients are less aware of the access control implementation and can be thinner, which will help with maintainability.
- *Formalized rules* - Rules are separated from code, and written down to make it easier to reason about, develop, and maintain.
- *More authorization surfaces* - Access control can be used for more use cases such as tenant isolation, plan entitlements, sharing, contextual policies.
- *Own the rules, not the engine* - We own the rules that determine authorization, not the infrastructure code that iterates over DB rows and figures out how to turn that into an authorization decision.

**Cons**

- *More complex architecture* - Requires an additional service deployed to the cluster, with its own data store.
- *Potential for higher latency* - It is possible that there is more network or computational overhead for each authorization request.
- *ReBAC graph could become inconsistent with DB state* - We need to make sure CRUD operations to DB objects are synchronized with the ReBAC graph, and we need to develop tools to respond to potential incidents where ReBAC graph state drifts and becomes inconsistent with the DB.

## Design 
*What are the key user experience and technical design decisions / trade-offs?*

We will adopt a ReBAC framework, then expose a minimal Python SDK for clients to perform authorization checks. In this case the python SDK will act as the PDP, and clients will act as the PEP.

### Python SDK

I propose the following approximate Python SDK.

```python
from posthog import authorization

# Check if a user can perform an action on a resource
can_access = authorization.check(subject=..., resource=..., action=...)

# Same as `authorization.check(...)`, but performs multiple checks in bulk
can_access = authorization.check(
    [
        authorization.CheckRequest(subject=..., resource=..., action=...),
        authorization.CheckRequest(subject=..., resource=..., action=...),
    ]
)

# Enumerate a user's permissions for a resource
permissions = authorization.permissions(subject=..., resource=...)

# Enumerate resources a user can access
accessible_resources = authorization.list_resources(subject=..., resource=..., action=...)

# Enumerate users who can access a resource
authorized_users = authorization.list_subjects(resource=..., action=..., subject=...)

# Generic utility for upserting and deleting relationships in bulk
authorization.write(
    [
        authorization.AddRelationshipRequest(subject=..., resource=..., action=...),
        authorization.DeleteRelationshipRequest(subject=..., resource=..., action=...),
    ]
)

# Special function to inspect the graph. Returns the existence of the (subject, resource, action) tuple in the graph, NOT whether the user has access
# Should only be used by APIs that explicitly need to check for specific relations like sharing UIs, for example.
# Prefixed with `dangerous_` to avoid accidental use
authorization.dangerous_query_relation(subject=..., resource=..., action=...)

# Debug function for PostHog admins. If decision is to allow, returns context for what relations were involved in that decision being made.
authorization.dangerous_explain(authorization.CheckRequest())
```

**For example, here are what some common access control operations might look like using this SDK.**

```python
from posthog import authorization

# check if user can access dashboard
can_access = authorization.check(
    subject=authorization.Subject(user_id=user.pk), # ex: "user:{user.pk}"
    resource=authorization.Resource(type=authorization.ResourceType.Dashboard, id=dashboard.pk), # ex: "dashboard:{dashboard.pk}"
    action=authorization.Action.Edit,
)
assert can_access is False

# grant user read and write access to dashboard
authorization.write(
    [
        authorization.AddRelationshipRequest(
            subject=authorization.Subject(user_id=user.pk),
            resource=authorization.Resource(type=authorization.ResourceType.Dashboard, id=dashboard.pk),
            action=authorization.Action.View,
        ),
        authorization.AddRelationshipRequest(
            subject=authorization.Subject(user_id=user.pk),
            resource=authorization.Resource(type=authorization.ResourceType.Dashboard, id=dashboard.pk),
            action=authorization.Action.Edit,
        ),
    ]
)

# check if user can access dashboard
can_access = authorization.check(
    subject=authorization.Subject(user_id=user.pk),
    resource=authorization.Resource(type=authorization.ResourceType.Dashboard, id=dashboard.pk),
    action=authorization.Action.Edit,
)
assert can_access is True

# remove user's edit access from the dashboard
authorization.write(
    authorization.DeleteRelationshipRequest(
        subject=authorization.Subject(user_id=user.pk),
        resource=authorization.Resource(type=authorization.ResourceType.Dashboard, id=dashboard.pk),
        action=authorization.Action.Edit,
    ),
)

# let user's list all dashboards in their project
authorization.write(
    [
        authorization.AddRelationshipRequest(
            subject=authorization.Subject(user_id=user.pk),
            resource=authorization.Resource(type=authorization.ResourceType.Dashboard, project_id=project.pk), # ex: "project:{project.pk}:dashboard"
            action=authorization.Action.List,
        ),
    ]
)

# check if a user can list dashboards
can_access = authorization.check(
    subject=authorization.Subject(user_id=user.pk),
    resource=authorization.Resource(type=authorization.ResourceType.Dashboard, project_id=project.pk),
    action=authorization.Action.List,
)
assert can_access is True

# check if a user can create dashboards
can_access = authorization.check(
    subject=authorization.Subject(user_id=user.pk),
    resource=authorization.Resource(type=authorization.ResourceType.Dashboard, project_id=project.pk),
    action=authorization.Action.Create,
)
assert can_access is False

# check if a user can view a dashboard that they have not been explicitly given access to
can_access = authorization.check(
    subject=authorization.Subject(user_id=user.pk),
    resource=authorization.Resource(type=authorization.ResourceType.Dashboard, id=other_dashboard.pk),
    action=authorization.Action.View,
)
assert can_access is True

# check if a user can access a bunch of different dashboards
dashboards = [dashboard, other_dashboard_1, other_dashboard_2]
can_access = authorization.check(
    [
        authorization.CheckRequest(
            subject=authorization.Subject(user_id=user.pk),
            resource=authorization.Resource(type=authorization.ResourceType.Dashboard, id=dashboards[0].pk),
            action=authorization.Action.View,
        ),
        authorization.CheckRequest(
            subject=authorization.Subject(user_id=user.pk),
            resource=authorization.Resource(type=authorization.ResourceType.Dashboard, id=dashboards[1].pk),
            action=authorization.Action.View,
        ),
        authorization.CheckRequest(
            subject=authorization.Subject(user_id=user.pk),
            resource=authorization.Resource(type=authorization.ResourceType.Dashboard, id=dashboards[2].pk),
            action=authorization.Action.View,
        ),
    ]
)
assert can_access == [True, False, False]

# see what a user can do for a particular dashboard
permissions = authorization.permissions(
    subject=authorization.Subject(user_id=user.pk),
    resource=authorization.Resource(type=authorization.ResourceType.Dashboard, id=dashboard.pk),
)
assert permissions == [authorization.Action.View]

# see what dashboards a user can access
resources = authorization.list_resources(
    subject=authorization.Subject(user_id=user.pk),
    resource=authorization.Resource(type=authorization.ResourceType.Dashboard, project_id=project.pk),
    action=authorization.Action.View,
)
assert resources == [authorization.Resource(type=authorization.ResourceType.Dashboard, id=dashboard.pk)]

# see who can access a dashboard
subjects = authorization.list_subjects(
    subject=authorization.Subject(type=authorization.SubjectType.User),
    resource=authorization.Resource(type=authorization.ResourceType.Dashboard, id=dashboard.pk),
    action=authorization.Action.View,
)
assert subjects == [authorization.Subject(user_id=user.pk)]

# Bring together role-based access control, data warehouse table access control, and relationship based access control into one beautiful access control graph
authorization.write(
    [
        authorization.AddRelationshipRequest(
            subject=authorization.Subject(role=role, project_id=project.pk), # ex: "project:{project.pk}:role:{role.pk}"
            resource=DataWarehouseTableResource(project_id=project.pk, table="postgres.stripe_invoices"), # ex: "project:{project.pk}:data_warehouse_table:postgres.stripe_invoices"
            action=authorization.Action.View,
        ),
    ]
)
```

### Gotcha: Django-isms

The mega-brained reader will know that there is a lot of access control checks and enforcement which happens in Django-land. With the new Python SDK we will re-implement the many Django permission enforcement and utility classes to use the new Python SDK under the hood, instead of directly evaluating access control rules from the database.

For example, `AccessControlViewSetMixin` would need to be updated to use the new python SDK instead of implementing the access control checks directly.

### Gotcha: list operations
List operations introduce an additional challenge because our existing access control system enforces object-level restrictions directly in the queryset. This means unauthorized objects are omitted from list results entirely. We will need to be careful to maintain this pre-existing behavior while migrating. Some queries are easy to migrate, only filtering objects that match a list of specific ids, whereas others are more complicated, joining on access control tables.

### Why now?

As we've scaled up the access coverage over more products, the data warehouse, properties, and new AI products, the current access control model has not scaled well. It has slowed down development, polluted some business logic with authorization code, and become difficult to reason about.

### Alternatives considered

**ABAC**

We already have lots of hierarchy in our rules: project admin overrides, creator escape hatch, object-level overrides, roles, etc. that are all dependent on their relationships between each other. ABAC is better suited for flat access control structures that are dependent on inline object attributes. For us, ReBAC is a better fit since it directly models relations.

**Move existing code behind new Python SDK, but don't adopt ReBAC**

In this case, we are still spending a lot of time maintaining the code which evaluates the access control rules. Each new use case requires lots of new Python code that intermixes different concerns and is dificult to refactor.

**Build the ReBAC engine ourselves**

If we adopt a standard ReBAC engine, then code examples on the internet makes it easier for LLMs to use. Additionally,
we want to spend the bulk of our development effort on the authorization rules, not the plumbing that enables that.

### Rollout?
The python SDK will control rollout of ReBAC backed access control. Initially, we will port the existing access control code into the access control product folder, and expose it through the new python SDK. Then, once we start implementing it to be backed by ReBAC, we will set up a flag that determines whether the legacy access control code or the new ReBAC model should be used for authorization evaluation.

This flag will have multiple variants: `off`, `shadow-log`, `rebac`. In `off`, only the existing Python logic is used. In `shadow-log` the existing Python is used as the authoritative answer, but a request is made to the ReBAC service in parallel and that result is logged against the result returned by the Python implementation. In `rebac`, the ReBAC service's response is used as the authoritative answer, and the Python implementation's response is logged.

### What about the data model?

In the first iteration we will maintain the existing data model by implementing a dual write system for object permissions. However, once the migration is complete we may re-evaluate this and opt to use the ReBAC service as the canonical source of truth for authorization policies, including those that we display in the UI and allow customers to configure.

### Most specific access control?

We will block this initiative on most-specific access control being shipped. Maintaining two big access control changes at once would be too difficult.

### What authorization questions are in or out of scope?

The goal is that all authorization questions are answered by the new engine. For example, this includes:

1. Organization and project access
2. Contextual policies (i.e. timestmap, risk, etc.)
3. Subject-policies (i.e. token type, mcp user agent, non-user subject, etc.)
4. Feature entitlement

### New vendor?

No, I think we should avoid this, and remain cloud-native since this is core infra.

### Who would own this?

Platform features would be responsible for maintaining, monitoring, scaling, and on-call response.

## Sprints

1. Research ReBAC frameworks, benchmark them, and select final option
2. Implement python authorization SDK using existing access control code
3. Implement access control decision logging
4. Set up ReBAC infra in prod, update local dev stack, update self-hosted stuff
5. Implement ReBAC backed access control behind a feature flag, add ReBAC decisions to access control logging
6. Verify existing access control and ReBAC access control agree on authorization decisions
7. Begin phased rollout of ReBAC
8. Remove old access control logic and make new ReBAC approach source of truth
