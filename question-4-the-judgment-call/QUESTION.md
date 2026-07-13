# Question 4 - The Judgment Call

You join as the sole engineer on a live product. In your first week you review the codebase and find three things: (1) no indexes on the leads collection which has 80,000 documents and is growing, (2) the Idempotency-Key check is not atomic - there is a race condition, (3) created_at timestamps are being set by the client, not the server.

Answer these:
- Which do you fix first and why?
- Write the fix for the one you consider most critical.
- How do you communicate these findings to a non-technical founder without causing panic?

We are looking for: prioritisation under real constraints, not a list of everything that needs fixing.
