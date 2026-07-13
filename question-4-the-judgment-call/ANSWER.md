# Answer - Question 4: The Judgment Call

## Fix order

**1. Idempotency-Key race condition — fixed first.**
This is the only one of the three actively corrupting data right now, on
every concurrent request. A trade show scanning app on flaky wifi retries
constantly — two near-simultaneous retries can both pass a non-atomic check
and both write, producing duplicate leads and any side effects tied to lead
creation (duplicate emails, duplicate CRM syncs). It's also the hardest to
clean up retroactively once duplicates are mixed into real data, so it has
the highest cost of delay.

**2. Missing indexes on `leads` — fixed same day, in parallel.**
80,000 documents and growing isn't an emergency yet, but it's the kind of
thing that becomes a sudden production incident with zero warning as the
collection keeps growing (see Question 3). The fix itself is cheap — add the
index, verify with `.explain()` — so there's no reason not to ship it
immediately alongside the race condition fix rather than waiting.

**3. Client-set `created_at` — fixed third.**
Real problem — client clocks can skew or be spoofed, and anything relying on
`created_at` for ordering, analytics, or SLA tracking is unreliable — but it
isn't actively corrupting data integrity the way the race condition is, and
it isn't going to cause an outage the way the missing index eventually
would. Fix it, but it doesn't compete for the top slot.

## The fix: atomic idempotency check

The bug is a classic check-then-act race: checking whether a key exists and
then inserting are two separate operations, so two concurrent requests can
both pass the check before either has written.

```python
# BROKEN: race condition. Two concurrent requests can both pass find_one()
# before either has inserted, so both proceed to insert.
async def create_lead_broken(payload: dict, idempotency_key: str):
    existing = await db.leads.find_one({"idempotencyKey": idempotency_key})
    if existing:
        return existing
    doc = {**payload, "idempotencyKey": idempotency_key, "createdAt": ...}
    result = await db.leads.insert_one(doc)
    return doc
```

```python
# FIXED: the check-and-write becomes a single atomic operation, enforced by
# a unique index rather than application logic.
#
# One-time migration:
#   db.leads.create_index("idempotencyKey", unique=True)

from datetime import datetime, timezone
from pymongo.errors import DuplicateKeyError

async def create_lead(payload: dict, idempotency_key: str):
    # payload spread FIRST, then server-trusted fields override it --
    # otherwise a client-supplied "idempotencyKey" or "createdAt" in the
    # payload would silently win and defeat the whole fix.
    doc = {
        **payload,
        "idempotencyKey": idempotency_key,
        "createdAt": datetime.now(timezone.utc),  # server-set, also fixes issue #3
    }
    try:
        result = await db.leads.insert_one(doc)
        doc["_id"] = result.inserted_id
        return doc
    except DuplicateKeyError:
        # Another concurrent request (or a genuine client retry) already
        # inserted this key. Return the existing doc instead of erroring --
        # that's the correct idempotent behavior.
        return await db.leads.find_one({"idempotencyKey": idempotency_key})
```

The unique index is what actually prevents the race — MongoDB guarantees
only one document per `idempotencyKey` can ever exist, even across multiple
app instances and arbitrary concurrency. The `try/except` is just handling
the outcome gracefully; it isn't the enforcement mechanism.

## Communicating this to a non-technical founder

"During my first-week review I found a bug where some leads can get
duplicated if the scanner retries on bad wifi at an event — which could mean
double emails to the same attendee or inflated lead counts. It's a common
class of bug in fast-moving products, I've already got a fix for it, and
I'm shipping it today along with a quick database optimization to keep the
app fast as we collect more leads. I also found a smaller data-quality issue
with timestamps that I'll clean up this sprint — low risk, just on my list.
No downtime needed for any of this."

This leads with business impact in plain language, signals the issue is
found-and-handled rather than an open crisis, gives a concrete timeline, and
avoids dumping all three issues as an undifferentiated wall of risk.
