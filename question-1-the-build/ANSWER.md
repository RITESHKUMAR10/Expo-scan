# Answer - Question 1: The Build

## Problem it solved

Progcap's ProgCommerce (progcomm.ai) is a B2B fintech platform serving
multiple enterprise clients, with two customer-facing surfaces: an internal
Admin Portal and a client-facing Storefront (web + app). These needed a
consistent design system, performant UI across devices, and secure
integration with the backend's auth and data APIs for enterprise clients
handling financial data.

## What I built + key technical decisions

Built and owned end-to-end: the Admin Portal (React.js, from scratch —
component architecture, state management, folder structure) and the
Storefront web + app frontend (consistent design system, responsiveness,
performance across devices), integrated with backend REST APIs/microservices
including JWT/OAuth auth flows.

Three decisions I'd stand behind on a call:

1. **Feature-based folder structure over type-based.** Early on the codebase
   was heading toward the typical `components/`, `hooks/`, `services/`
   split, which looks clean at first but means every change to one feature
   (say, lead/client management in the Admin Portal) touches four unrelated
   top-level folders. I moved to a feature-based structure — each feature
   owns its components, hooks, and API calls together — because the team was
   growing and I wanted new engineers to be able to work inside one folder
   without stepping on other features. Trade-off I accepted: some
   duplication of small UI bits across features instead of a single shared
   pool, which I mitigated with a thin `common/` layer for truly generic
   primitives (buttons, inputs, layout).

2. **Server state via API layer + lightweight local state, not a global
   Redux store.** Most of the Admin Portal and Storefront's state is really
   server data (leads, clients, orders) that's fetched, cached, and
   invalidated — not client-owned application state. Rather than putting
   everything through Redux, I kept genuinely local UI state (form state,
   modals, filters) in component/Context state and let API calls own the
   server-derived data with their own caching/invalidation. This cut a lot
   of boilerplate and avoided a whole class of "store is stale vs server"
   bugs. The trade-off: less centralized visibility into what's loading/
   erroring app-wide, which I addressed with a small shared convention for
   loading/error states rather than a heavier library.

3. **Route-level guarding tied to JWT claims, not just token presence.**
   For an enterprise/fintech admin surface, "logged in" isn't enough —
   different roles needed to see different parts of the Admin Portal. I
   built route guarding that checks decoded JWT claims (role/permissions),
   not just whether a token exists, so an authenticated-but-unauthorized
   user gets redirected rather than seeing a broken page or a flash of
   data they shouldn't see. Trade-off: token payload had to stay in sync
   with backend permission changes, so I made sure token refresh actually
   re-fetched fresh claims rather than reusing a cached decode.

## Multi-tenant? Data isolation?

Yes. ProgCommerce serves multiple enterprise clients on shared
infrastructure, so isolation is enforced backend/API-side — every request
carries a tenant/client claim from the JWT, and the backend scopes data
access to that claim. My responsibility on the frontend was making sure the
UI never leaked state across tenants: the API client is initialized per
authenticated session (not a shared singleton reused across logins), and any
cached data (React Query/SWR-style caches, local component state) gets fully
cleared on logout or account switch rather than trusted to naturally
overwrite — a stale cache silently showing the previous client's data is an
easy, embarrassing bug in a multi-tenant admin UI, so I treated cache
invalidation on identity change as a hard rule, not an afterthought.

## What broke in production

We used Redis as a job queue to process bulk lead imports (clients
uploading CSVs of hundreds/thousands of leads at once) instead of
processing the whole file synchronously in the request. During a worker
restart/deploy while an import was mid-flight, a batch of jobs got picked up
twice — once by the worker that was shutting down and hadn't acknowledged
completion yet, and again by a new worker that started polling the same
queue — resulting in duplicate lead records for that client's import.

**Detection:** Noticed via a spike in lead count for one client that didn't
match their uploaded file's row count; confirmed by checking for duplicate
records sharing the same source-file identifier.

**Root cause:** Job completion wasn't acknowledged atomically with the
lead-creation write — a worker would create the lead, then separately mark
the Redis job as done. If the worker died in that gap (deploy, crash,
restart), the job stayed in a state where another worker would pick it up
again and reprocess it, since there was no idempotency check on the
import-row level.

**Fix:** Made each import row idempotent by keying inserts on a deterministic
hash of (import batch ID + row content) with a unique index, so a
reprocessed job upserts instead of duplicating — same pattern as the
idempotency-key fix in Question 4. Also moved job acknowledgment to happen
only after the DB write fully committed, closing the gap where a crash
could leave a job "in limbo."

**Prevention:** Added a reconciliation check after each bulk import
(uploaded row count vs. rows actually created) that flags a mismatch, and
made worker deploys drain in-flight jobs before shutting down instead of
killing them mid-batch.
