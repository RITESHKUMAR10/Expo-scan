# Question 2 - The Architecture

You are migrating ExpoScan - a single-tenant FastAPI + MongoDB PWA - to a multi-tenant SaaS product. Multiple clients will use the same deployment. Their data must be completely isolated.

Answer the following:
- Pick one data isolation approach (shared collection with tenantId, or database-per-tenant) and justify it for this specific context - a trade show lead capture app expecting 50 to 200 tenants at launch.
- Write the FastAPI middleware or dependency that enforces tenant isolation on every request. Show the code.
- What is the single biggest risk in this migration and how do you mitigate it?

We are looking for: real architectural judgment, not a textbook comparison of both approaches.
