# Answer - Question 3: The Debugging

## First three things I'd check, in order

**1. MongoDB server-side health, before app logs.**
Run `mongostat` / `mongotop` (or check Atlas metrics if hosted): connection
count vs the driver's pool limit, opcounters, and replication lag. Also run
`rs.status()` to check replica set state — a primary election/failover
produces exactly this symptom: a burst of timeouts with zero app-side cause.
Check disk I/O and CPU on the Mongo host — if the collection's working set
has grown past available RAM over the last 3 months, queries that used to be
served from cache start hitting disk and creep past the timeout threshold.
This is the most common "nothing changed but it broke" cause for a
collection that's been growing quietly.

**2. Query-level diagnosis.**
Run `db.currentOp()` to see what's actually executing/blocked right now.
Run `.explain("executionStats")` on the exact query the failing endpoint
issues — check `totalDocsExamined` vs `nReturned`, and whether it's hitting
an index (`IXSCAN`) or scanning the whole collection (`COLLSCAN`). A
collection that grew significantly over 3 months can silently cross the
point where a missing or wrong index turns a fast query into a slow one —
no deploy required, the data just grew into the problem. I'd also check the
slow query log / enable the profiler for the window the errors started:
`db.setProfilingLevel(1, {slowms: 100})`.

**3. App-side connection handling.**
Check the Mongo client config (`maxPoolSize`, `serverSelectionTimeoutMS`,
`socketTimeoutMS`) — under sustained load, a too-small connection pool
causes queueing that looks like random timeouts. Check
`db.serverStatus().connections` (current vs available) for a slow
connection leak that could take months to exhaust the pool. Cross-reference
the exact timestamp of the first 500 against any infra event: Atlas
maintenance window, autoscaling event, or a cron job/second service newly
hitting the same cluster.

## Most likely cause

Given the constraints in the prompt — three months stable, no deploy, no
schema change, normal traffic — the strongest candidate is: **the collection
outgrew its index or its working set.** Steady organic data growth pushed
either a previously-irrelevant missing index into relevance (a query that
did a light scan over a small collection is now scanning hundreds of
thousands of documents), or pushed the collection's hot data past available
RAM, so queries that were served from cache now hit disk under normal load.

Second candidate: a Mongo-side infra event (replica set failover,
maintenance window, resource cap on a shared/managed tier) causing transient
timeouts unrelated to the app at all.

## How I'd confirm it

- For the growth/index theory: run `.explain()` on the exact query behind
  the failing endpoint, check `docsExamined` and index usage, and compare
  collection + index size against available RAM via `db.collection.stats()`
  and `db.serverStatus().mem`.
- For the infra-event theory: check Mongo Atlas/ops logs for election
  events, maintenance windows, or resource threshold alerts around the exact
  timestamp of the first 500 error.

I'd confirm one before assuming the other — the two have different fixes
(add an index / upgrade tier vs. nothing to fix, just failover behavior to
tolerate with retries).
