# Request for comments: Adopt relationship-based access control (ReBAC)

**Author:** Reece Jones
**Date:** 2026-08-13

## TLDR

Our access control model no longer scales with our access control needs. As we add access control in different shapes to more places, we are increasingly relying on one-off implementations that take lots of effort to build, verify, and ship. Additionally, auditing existing access control logic has become increasingly diffuclt as the rules are not centralized and are spread throughout different parts of the codebase. This RFC proposes we adopt Relationship-Based Access Control (ReBAC) as a general purpose access control engine to consolidate access control rule evaluation, abstract access control logic away from consumers, and increase auditiability.

## Problem statement

The current access control implementaiton has several problems that make it hard to develop for, fragile, and difficult to audit.

### Access control logic spread across different places

The logic that evaluates access control rules is spread across many places in the codebase. For example:
- *Django viewsets and serializers*
- *HogQL AST parser and printer*
- *Filesystem API*
- *Frontend frontend access checks*
- *One-off database queries*

This has the unfortunate side-effect of:
1. Coupling access control logic to other logic. For example, access control has become highly coupled to the HogQL implementation. This means that HogQL contributors must be explicitly aware of the access control implementation so queries perform as users expect.
2. Access control is re-implemented in multiple places. For example, access control resolution logic is implemented on both the frontend and backend and need to be kept in sync.

### Our access control needs have outpaced our access control model

When access control was first developed it was made specifically for the Django pattern of managing data, and under the assumption that all resources would have the same resolution logic. For example, the following product areas have had to implement their own access control logic or have skipped implementing access control entirely:

- *PostHog AI*
- *Billing access control*
- *Guest-mode project access*
- *MCP*
- *Sharing*

For example, in PostHog AI `Tasks` are visible to the creating user and hidden to the rest of the team by default, however our access control implementation assumes the opposite: that objects are visible to the entire team by default.

### Multiple access control systems

As our access control model has been unable to adapt to new use-cases, multiple systems have taken shape. For example, the previously mentioned PostHog AI `Tasks` have received their own access control implementation. However, the issue is more systemic

Sharing has its own system, which has grown increasingly complex and nearly impossible to work for large product surfaces like notebooks.

Access to many objects isn't covered by our access control implementation, but rather by static analysis using semgrep IDOR rules.

There are also many small access control checks that are stored as one-off database columnms and evaluated in one-off implementations. For example, at the organization level `members_can_invite`, `members_can_see_org_members`, `members_can_use_personal_api_keys`, etc. But at the project level access control rules are stored in their own data model. Meaning that the evaluation of access control rules at the organization and project level are drastically different.

### Auditability

Right now if a user is having trouble accessing something, we have no way of easily figuring out why. Our best bet is to dump their acces control rules from the database, feed it to an agent, and ask it to figure out why it isn't working. This is because we do not have a centralized rule engine, because access control evaluation happens in many different places. So it can be very difficult to reason about the live access control logic.

Additionally, we do not have an audit trail that documents access control decisions. This means we cannot retroactively determine if a user was granted or denied access to a resource, let alone understand why. As we and our customers continue to scale, this will be important to document for security incident response.

## Success criteria
*How do we know if this is successful (i.e. metrics, customer feedback), what's out of scope, whats makes this ambitious?*

1. No customer's access is impacted by the change (they don't notice we changed the engine while they were driving)
2. All access control evaluation flows through a single engine. There are no divergent paths with one-off access control implementations or hard-coded rules.
3. We have audit logs documenting access control decisions
4. We can ship new access control rules faster and it is easier for agents to work with

## Context
*What are our competitors doing, what are the technical constraints, what are customers asking for, what does the data tell us, are there external motivations (e.g. launch week, enterprise contract)?*

### Customer context

Customers keep asking for new access control features that we are unable or too slow to build, because of the inflexibility of our access control system.

### Modern architecture

Modern authorization architecture is typically divided into two components: a Policy Evaluation Point (PEP), and a Policy Decision Point (PDP). The PEP is responsible for answer the question "Can the user do this action?", whereas the PDP is responsible for enforcing the resolved access for the user.

For example, a viewset which returns `Dashboards` would be a PDP. The viewset would ask the PEP if the authenticated user can perform the API action. The PEP returns a boolean response, yes or no, and if denied the PDP is responsible for responding with the correct HTTP response.

This decoupled architecture improves scalability, because changes to the access control rules are contained to the PEP. The PDP treats the PEP as a black box, which reduces the blast radius of access control logic changes. Additionally, modern authorization systems rely on domain specific languages (DSLs) and specialized data structures to model access control rules. This imposes a design constraint that forces developers to model access control as formal logic rather than application code.

See [ABAC](https://en.wikipedia.org/wiki/Attribute-based_access_control) and [Cedar](https://docs.cedarpolicy.com/) for further reading on this architecture.

### ReBAC

ReBAC solves the problem of access control by modeling access as a graph. A user can perform an action on an object if they have a relation, direct or indirect, to that object. This is a natural fit for us, because we already model access control in a highly relational way. Our access control model is basically a bunch of database relations, so formalizing as a graph and centralizing the logic is not a big conceptual leap.

### Existing implementations

Google uses [Zanzibar, their in-house ReBAC implementation](https://storage.googleapis.com/gweb-research2023-media/pubtools/5068.pdf) to implement access control in many of their products. Zanzibar is the source of inspiration for nearly all modern ReBAC implementations

[OpenFGA](https://openfga.dev/) is a ReBAC implementation that is a CNCF incubation project. It seems to be the most popular ReBAC implementation right now due to its cloud-native approach. It is backed by Okta, and has lots of logos on the website. There is extensive documentation and talks about OpenFGA available.

[SpiceDB](https://authzed.com/spicedb) is a authorization database with support for ReBAC configuration. It is [open source](https://github.com/authzed/spicedb), under the Apache 2.0 License, and maintained by [AuthZed](https://authzed.com) who sell authorization as a service.

[Permify](https://permify.co/) is another ReBAC implementation that is open source. Like OpenFGA, it supports arbibtrary data backends for storing the relationship graph.

## Design 
*What are the key user experience and technical design decisions / trade-offs?*

We will adopt a ReBAC framework, then expose a minimal Python API for clients to perform authorization checks. In this case the python API will act as the PEP, and clients will act as the PDP.

### Python API

I propose the following simple python API. This python API should be exposed from the `products/access_control` product folder, and be the only way that access control checks are performed by application code outside of `products/access_control`.

```python
from products.access_control.backend.facade import access_control

# Check if a user can perform an action on a resource
can_access = access_control.check(subject=..., resource=..., action=...)

# Same as `access_control.check(...), but performs multiple checks in bulk
can_access = access_control.check(
    [
        access_control.CheckRequest(subject=..., resource=..., action=...),
        access_control.CheckRequest(subject=..., resource=..., action=...),
    ]
)

# Enumerate a user's permissions for a resource
permissions = access_control.permissions(subject=..., resource=...)

# Enumerate resources a user can access
accessible_resources = access_control.list_resources(subject=..., resource_type=..., action=...)

# Enumerate users who can access a resource
authorized_users = access_control.list_subjects(resource=..., action=..., subject_type=...)

# Generic utility for upserting and deleting relationships in bulk
access_control.write(
    [
        access_control.AddRelationshipRequest(subject=..., resource=..., action=...),
        access_control.DeleteRelationshipRequest(subject=..., resource=..., action=...),
    ]
)

# Special function to inspect the graph. Returns the existence of the (subject, resource, action) tuple in the graph, NOT whether the user has access
# Should only be used by APIs that explicitly need to check for specific relations like sharing UIs, for example.
# Prefixed with `dangerous_` to avoid accidental use
access_control.dangerous_query_relation(subject=..., resource=..., action=...)

# Debug function for PostHog admins. If decision is to allow, returns context for what relations were involved in that decision being made.
access_control.dangerous_explain(access_control.CheckRequest())
```

**For example, here are what some common access control operations might look like using this API.**

```python
from products.access_control.backend.facade import access_control

# check if user can access dashboard
can_access = access_control.check(subject=f"user:{user.pk}", resource=f"dashboard:{other_dashboard.pk}", action="edit")
assert can_access is False

# grant user access read and write access to dashboard
access_control.write(
    [
        access_control.AddRelationshipRequest(subject=f"user:{user.pk}", resource=f"dashboard:{dasboard.pk}", action="view"),
        access_control.AddRelationshipRequest(subject=f"user:{user.pk}", resource=f"dashboard:{dasboard.pk}", action="edit"),
    ]
)

# check if user can access dashboard
can_access = access_control.check(subject=f"user:{user.pk}", resource=f"dashboard:{other_dashboard.pk}", action="edit")
assert can_access is True

# remove user's edit access from the dashboard
access_control.write(
    access_control.DeleteRelationshipRequest(subject=f"user:{user.pk}", resource=f"dashboard:{dasboard.pk}", action="view"),
)

# let user's list all dashboards in their project
access_control.write(
    [
        # note that `dashboards` != `dashboard`. `dashboard` is for individual objects, `dashboards` is for collection operations
        # In ReBAC we can define rules so that a user is automatically given access to `dashboard:{dashboard.pk}` if they have access to `dashboards:{project_id}`
        access_control.AddRelationshipRequest(subject=f"user:{user.pk}", resource=f"dashboards:{project_id}", action="list"),
    ]
)

# check if a user can list dashboards
can_access = access_control.check(subject=f"user:{user.pk}", resource=f"dashboards:{project_id}", action="list")
assert can_access is True

# check if a user can create dashboards
can_access = access_control.check(subject=f"user:{user.pk}", resource=f"dashboards:{project_id}", action="create")
assert can_access is False

# check if a user can view a dashboard that they have not been explicitly given access to
can_access = access_control.check(subject=f"user:{user.pk}", resource=f"dashboard:{other_dashboard.pk}", action="view")
assert can_access is True

# check if a user can access a bunch of different dashboards
dashboards = [dashboard, other_dashboard_1, other_dashboard_2]
can_access = access_control.check(
    [
        access_control.CheckRequest(subject=f"user:{user.pk}", resource=f"dashboard:{dashboards[0].pk}", action="view"),
        access_control.CheckRequest(subject=f"user:{user.pk}", resource=f"dashboard:{dashboards[1].pk}", action="view"),
        access_control.CheckRequest(subject=f"user:{user.pk}", resource=f"dashboard:{dashboards[2].pk}", action="view"),
    ]
)
assert can_access == [True, False, False]

# see what a user can do for a particular dashboard
permissiosn = access_control.permissions(subject=f"user:{user.pk}", resource=f"dashboard:{dashboards[0].pk}")
assert permissions = ["view"]

# see what dashboards a user can access
resources = access_control.accessible_resources(subject=f"user:{user.pk}", resource_type=f"dashboard", action="view")
assert resources == [f"dashboard:{dashboard.pk}"]

# see who can access a dashboard
subjects = access_control.list_subjects(suject_type="user", resource=f"dashboard:{dashboard.pk}" action="view")

# Bring together role-based access control, data warehouse table access control, and relationship based access control into one beautiful access control graph
access_control.write(
    [
        access_control.AddRelationshipRequest(subject="role:worldcom_auditors", resource=f"data_warehouse_table:{project_id}:postgres.stripe_invoices", action="view"),
        access_control.AddRelationshipRequest(subject="role:worldcom_accountants", resource=f"data_warehouse_table:{project_id}:postgres.stripe_invoices", action="edit"),
    ]
)
```

### Gotcha: Django-isms

The mega-brained reader will know that the python API is only half the battle. There is a whole load of access control enforcement which happens through Django-land. With only the new Python API, downstream code would need to implement lots of one-off authorization checks for the automatic CRUD operations that Django gives us. To solve for this, we will re-implement the many Django permission enforcement and utility classes to use the new Python API under the hood, instead of directly evaluating access control rules from the database.

For example, `TeamMemberAccessPermission` could be updated to check: `access_control.check(subject=f"user:{user.pk}", resource=f"project:{project_id}", action="view")`

### Gotcha: list operations
One of the oddities of our access control system right now is that our mixins inject access control logic into the querysets so that object-level access control restrictions are enforced at the database level. Under a typical access control system, you might expect that the list endpoint returns an enumeration of all of the objects under the collection, including denied objects, exposing only the necessary metadata. Then if you attempted to fetch a specific object that you were denied for, then the request would fail. However, our access control system strips that object from the list result entirely.

The effect is that we need to be careful to maintain this pre-existing behavior while migrating. This should be possible using some trickery with `access_control.list_resources(...)` to return the set of explicitly allowed objects when the user doesn't have access to list the resource, or to return the set of explicitly denied objects when they do have access to list the resource. This is how it is implemented now, but some extra engineering effort might need to go into this edge case.

### Rollout?
The python API will control rollout of ReBAC backed acces control. Initially, we will port the existing access control code into the access control product folder, and expose it through the new python API. Then, once we start implementing it to be backed by ReBAC, we will setup a flag that determines whether the legacy access control code or the new ReBAC model should be used for authorization evaluation.

### What about the data model?

Until we have a reason to replace the existing data model, we will avoid doing so. In any world, we need a source of truth for access control policies. All that changes is now we replicate those policies to the ReBAC engine.

### Why not use ABAC or a policy engine like Cedar?

We already have lots of heirarchy in our rules: project admin overrides, creator escape hatch, object-level overrides, roles, etc.. ABAC and Cedar are better suited for flat access control structures. ReBAC is a better fit since it directly models relations.

### Most specific access control?

We will block on most-specific access control being shipped. Maintaining two big access control changes at once would be difficult.

### New vendor?

No, I think we should avoid this, and remain cloud-native since this is core infra.

## Sprints

1. Research options, benchmark them, and select final option
2. Implement python control API using existing access control code
3. Implement access control decision logging
3. Setup ReBAC infra in prod, update local dev stack, update self-hosted stuff
4. Implement ReBAC backed access control behind a feature flag, add ReBAC decisions to access control logging
5. Verify existing access control and ReBAC access control agree on authorization decisions
6. Begin Phased rollout of ReBAC
7. Remove old access control logic and make new ReBAC approach source of truth

## Open questions

**What ReBAC engine should be choose?**

I propose either [OpenFGA](https://openfga.dev/) or [SpiceDB](https://authzed.com/spicedb). Both are mature, open source, cloud native, and have big logos using them. I think there are two blockers in making a decision:
1. We should build PoCs to verify both will work for our use case, and identity any hidden gotchas before going hog wild.
2. Because this introduces a new piece of infrastructure complexity, we should choose whichever the infrastructure team is most comfortable operating.
