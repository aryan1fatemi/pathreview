## Week 7 — Issue selection

**Issue link:** https://github.com/jamjamgobambam/pathreview/issues/70

**Issue title:** Add rate limiting per IP address in addition to per user

**Tier:** [x] Tier 2

**Problem summary:**
The existing `RateLimiter` in `safety/rate_limiter.py` supports rate limiting by any identifier, but it's only ever invoked with an authenticated user's ID. Requests that don't carry a user identity — like calls to public/unauthenticated endpoints — currently pass through with no rate limiting at all, leaving them open to abuse. This issue adds per-IP rate limiting as a secondary layer, applied via middleware in `api/middleware/`, so every request is bounded by client IP regardless of authentication state. A successful fix means unauthenticated traffic is now capped per IP using the same rolling-window Redis-backed limiter already used for per-user limits.

**Branch name:** feat/70-per-ip-rate-limiting

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [ ] Issue added to cohort ledger
