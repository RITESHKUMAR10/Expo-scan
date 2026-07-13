# Answer - Question 2: The Architecture

## Data isolation approach: shared collection with `tenantId`

For ExpoScan at 50-200 tenants launch, I'd use a **shared collection with a
`tenantId` field**, not database-per-tenant.

Reasoning specific to this context:
- **Data volume per tenant is small.** Trade show lead capture generates a
  few hundred to a few thousand leads per tenant per event. There's no
  performance or scale argument for physical isolation here.
- **Ops overhead scales badly with DB-per-tenant.** 50-200 separate Mongo
  databases means 50-200 connection pools, migration runs, and backup/restore
  targets to manage. For a small team, that overhead grows linearly with
  sales success, which is the wrong direction.
- **No regulatory driver for physical isolation.** There's nothing in this
  context (trade show lead capture) that mandates hard tenant separation the
  way healthcare or finance data might. Isolation enforced correctly in
  software is sufficient at this sensitivity level.
- **Simpler CI/CD and cross-tenant tooling.** One schema to migrate, one
  place to run admin/analytics queries across tenants (useful for ExpoScan's
  own product analytics and support tooling).

The tradeoff I'm accepting: isolation now depends on the application layer
being correct on every single query, not on physical separation. I mitigate
that below.

## FastAPI dependency enforcing tenant isolation

```python
from fastapi import Depends, HTTPException, Request
from bson import ObjectId

async def get_current_tenant(request: Request) -> str:
    # Tenant is resolved from the verified JWT claim set by auth middleware,
    # never from a client-supplied header/query param/body field. Trusting
    # client input for tenant_id is the most common way this pattern gets
    # broken into.
    user = request.state.user
    if not user or "tenant_id" not in user:
        raise HTTPException(status_code=401, detail="Missing tenant context")
    return user["tenant_id"]


async def get_tenant_scoped_db(
    tenant_id: str = Depends(get_current_tenant),
):
    return TenantScopedCollection(db.leads, tenant_id)


class TenantScopedCollection:
    """Wraps a Mongo collection and forces tenantId into every operation,
    so isolation is enforced in one place instead of re-implemented per route."""

    def __init__(self, collection, tenant_id: str):
        self._collection = collection
        self._tenant_id = tenant_id

    def find(self, filter: dict = None, **kwargs):
        filter = {**(filter or {}), "tenantId": self._tenant_id}
        return self._collection.find(filter, **kwargs)

    def find_one(self, filter: dict = None, **kwargs):
        filter = {**(filter or {}), "tenantId": self._tenant_id}
        return self._collection.find_one(filter, **kwargs)

    def insert_one(self, doc: dict, **kwargs):
        doc = {**doc, "tenantId": self._tenant_id}
        return self._collection.insert_one(doc, **kwargs)

    def update_one(self, filter: dict, update: dict, **kwargs):
        filter = {**filter, "tenantId": self._tenant_id}
        # Strip tenantId from every update operator so a caller can never
        # reassign a document to another tenant via the update payload.
        update = {
            op: {k: v for k, v in fields.items() if k != "tenantId"}
            for op, fields in update.items()
        }
        return self._collection.update_one(filter, update, **kwargs)

    def delete_one(self, filter: dict, **kwargs):
        filter = {**filter, "tenantId": self._tenant_id}
        return self._collection.delete_one(filter, **kwargs)


@app.get("/leads/{lead_id}")
async def get_lead(
    lead_id: str,
    leads: TenantScopedCollection = Depends(get_tenant_scoped_db),
):
    lead = leads.find_one({"_id": ObjectId(lead_id)})
    if not lead:
        raise HTTPException(status_code=404)
    return lead
```

Design points:
- `tenant_id` comes only from the verified JWT, never from anything the
  client can set directly.
- Isolation logic lives in one wrapper class, so no route can accidentally
  forget to scope a query.
- Every collection gets a compound index starting with `tenantId` so scoped
  queries stay fast as data grows:
  `db.leads.create_index([("tenantId", 1), ("createdAt", -1)])`.

## Biggest risk and mitigation

**Biggest risk:** a raw query that bypasses the tenant-scoping wrapper — a
missing `tenantId` filter doesn't crash anything, it silently leaks one
tenant's data to another. That's the failure mode that's both most likely
(easy to introduce under deadline pressure) and most damaging (a data breach,
not a bug report).

**Mitigation:**
1. Force all data access through the `TenantScopedCollection` wrapper —
   no route should ever call `db.leads` directly.
2. Automated cross-tenant isolation tests: seed two tenants, hit every
   endpoint as tenant A using tenant B's IDs, assert 403/404 on all of them.
   Run this in CI on every PR.
3. Code review rule / lint check that flags any direct `db.<collection>`
   access outside the wrapper.
4. Structured logging that tags every query with `tenantId`, so a leak can
   be caught and audited quickly if one slips through.
