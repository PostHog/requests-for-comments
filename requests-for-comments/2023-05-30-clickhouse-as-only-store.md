# Request for comments: Clickhouse as our only data store

## Problem statement

- Having two separate places where we store essentially the same data (person/person_distinct_id/groups) means we often have inconsistencies
- Our plans with the data warehouse are to allow customers to model all of their entities (persons/groups) in SQL in Clickhouse. For this to work we need to be able to read everything everywhere from clickhouse
- When postgres goes down due to load from this data, it takes down the entire app

## Success criteria
*How do we know if this is successful (i.e. metrics, customer feedback), what's out of scope, whats makes this ambitious?*
We no longer store any person/groups data on postgres

## Context
I think we could revisit some of our assumptions on what our system needs to be able to do this. For example, I think an hour or days delay for analytics, as long as we still have a real time view of events coming in, even if they aren't enriched with person data.

## Design 

There are a couple of places we use postgres

**Ingestion**
Options
- Read person properties/groups/distinct ids from Clickhouse, with some kind of cache to avoid overloading
- Materialized view that joins the person properties onto the table, perhaps at some interval like an hour or day.

**Decide/feature flags**
This will be hardest, as decide does a ton of queries and they all need to be <100ms, whereas Clickhouse, even for smallest queryies takes up to a second or so.

Probably need to do aggressive caching or precalculation here.

**App**
- To display lists of groups in the interface (this will be trivial to swap out)


## Sprints
*How do we break this into discrete and valuable chunks for the folks shipping it? How do we ensure it's high quality and fast?*
