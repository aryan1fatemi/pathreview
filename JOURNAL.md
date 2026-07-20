## Week 7 — Issue selection

**Issue link:** https://github.com/jamjamgobambam/pathreview/issues/70

**Issue title:** Add rate limiting per IP address in addition to per user

**Tier:** [x] Tier 2

**Problem summary:**
The existing `RateLimiter` in `safety/rate_limiter.py` supports rate limiting by any identifier, but it's only ever invoked with an authenticated user's ID. Requests that don't carry a user identity — like calls to public/unauthenticated endpoints — currently pass through with no rate limiting at all, leaving them open to abuse. This issue adds per-IP rate limiting as a secondary layer, applied via middleware in `api/middleware/`, so every request is bounded by client IP regardless of authentication state. A successful fix means unauthenticated traffic is now capped per IP using the same rolling-window Redis-backed limiter already used for per-user limits.

**Branch name:** feat/70-per-ip-rate-limiting

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [ ] Issue added to cohort ledger

## Week 8 — Reproduction & solution planning

**Reproduction commit link:** __REPRO_COMMIT_URL__

**Reproduction summary:**
Against the baseline (`main`) request path I sent 65 unauthenticated `GET /` requests
and every one returned `200` with zero `429`s — confirming unauthenticated traffic is
not rate limited at all. Driving the existing `safety/rate_limiter.py` limiter directly
with an `ip:` identifier (limit 5) denied the 6th request, proving the machinery works
and only the per-IP wiring into the request path is missing.

**PLAN.md link:** https://github.com/aryan1fatemi/pathreview/blob/feat/70-per-ip-rate-limiting/PLAN.md

**Walkthrough video (recommended):** _(not recorded)_

**Blockers or open questions:**
Client-IP resolution behind a proxy/load balancer is the main open question — whether to
trust `X-Forwarded-For`, and how to avoid collapsing all clients behind one proxy into a
single bucket. Also undecided: whether to exempt `GET /health` from the per-IP limit.

### Reproduction details

Reproduced self-contained (no Docker), Python 3.11 venv with `fastapi`, `httpx`, `redis`,
`structlog`. Two observations plus static evidence:

1. **Static evidence (baseline gap).** On `main`,
   `git grep -n check_rate_limit -- ':!tests'` returns only the definition in
   `safety/rate_limiter.py` — no call sites under `api/`. The limiter is never invoked in
   the request path, so no endpoint (authenticated or not) is throttled.

2. **Dynamic — gap.** A FastAPI app mirroring `main`'s public `GET /` (no IP middleware),
   hit 65× via `TestClient`:
   ```
   Sent 65 unauthenticated GET / requests (limit would be 60/min).
     200 responses: 65
     429 responses: 0
     RESULT: GAP REPRODUCED — no request was ever throttled
   ```

3. **Dynamic — root cause.** The real `RateLimiter` (from `safety/rate_limiter.py`) keyed
   by `ip:203.0.113.7` with `limit=5`:
   ```
   req #1: allowed=True  remaining=4
   ...
   req #5: allowed=True  remaining=0
   req #6: allowed=False remaining=0
   RESULT: limiter denies at request #6 when keyed by IP
   ```
   The limiter enforces limits correctly per IP; the fix is purely to invoke it per request
   in `api/middleware/`.
