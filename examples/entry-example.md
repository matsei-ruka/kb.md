---
title: Request Rate Limiter
section: subsystem
status: active
last_verified: 2026-04-20
verified_by: a-maintainer
code_anchors:
  - path: src/middleware/rateLimiter.ts
    symbol: rateLimiter
  - path: src/middleware/rateLimiter.ts
    symbol: RATE_LIMIT_TIERS
  - path: config/redis.ts
    symbol: redisClient
related:
  - subsystem/auth.md
  - contract/public-api.md
source_of_truth:
  tier_definitions: "src/middleware/rateLimiter.ts :: RATE_LIMIT_TIERS"
  redis_keyspace: "docs/ops/redis-keys.md"
---

# Request Rate Limiter

## What it is

Token-bucket rate limiter in front of every public HTTP route. Buckets live in Redis, keyed by `api_key` for authenticated traffic and by the forwarded client IP for unauthenticated traffic. Tier limits are defined once in `RATE_LIMIT_TIERS` and applied via the `rateLimiter` middleware.

## Key invariants

1. **One bucket per API key, not per route.** Tier limits are global for a key; per-route throttling lives in the handler, not here.
2. **Unauthenticated traffic is keyed by `x-forwarded-for`, not socket IP.** The reverse proxy always sets the header; direct hits are rejected upstream before they reach the middleware.
3. **Redis outage fails open, not closed.** The middleware logs and skips limiting. Losing Redis briefly should not take down the API.
4. **Tier changes require a rolling restart.** Buckets are initialized from `RATE_LIMIT_TIERS` at boot; changing the const without restart silently keeps old limits.

## Architecture

```
client ──► rateLimiter middleware ──► route handler
              │
              ▼
         Redis (INCR + EXPIRE on key)
```

Keyspace: `rl:<tier>:<identity>:<window>`. Windows rotate hourly from the Redis server clock.

## Mini recipe

Add a new tier:

```
1. Add row to RATE_LIMIT_TIERS in src/middleware/rateLimiter.ts
2. Add the tier name to the API-key schema migration
3. Rolling-restart all API workers
4. Verify with scripts/check-rate-limit.ts <key>
```

## Gotchas

- **Clock skew**: windows come from the Redis server clock, not the app clock. Don't assume they align with wall clock on the app host.
- **First-request spike**: a cold bucket allows a full burst immediately. If that matters for a specific tier, pre-warm with a scripted ping on deploy.

## Open questions / known stale

- **2026-03-11**: We think `x-forwarded-for` trust is configured at the load balancer, but the LB config isn't in this repo. Confirm with platform team before changing trust rules here.

## See also

- `subsystem/auth.md` — API key issuance and revocation; rate limits are keyed off the same identity.
- `contract/public-api.md` — tier limits are part of the public contract and documented there too.
