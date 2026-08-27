# 📌 Problem Statement — Rate Limiter

## Summary

Design a system that limits how many requests a single source (a user, an
IP address, or an API client) can make to a backend service within a given
period of time — and gracefully rejects any request beyond that limit.

## The Problem

Backend services are exposed to the public internet, which means anyone —
not just legitimate users — can send requests to them. Without any
restriction on request volume, a service is vulnerable to:

- **Brute-force attacks** — e.g., an attacker rapidly guessing passwords or
  API keys by sending thousands of attempts per second.
- **Resource exhaustion / rising costs** — e.g., unlimited uploads or
  writes consuming storage and compute resources without bound, directly
  increasing operating costs.
- **Service degradation for real users** — a flood of requests from one
  bad actor can slow down or crash the service for everyone else.

There is currently **no mechanism** to detect or restrict excessive request
volume from a single source, leaving the system exposed to all of the
above.

## Goal

Design and build a **rate limiter** that:

1. Tracks how many requests each source (user/IP) has made within a
   configurable time window.
2. **Blocks** further requests once that source exceeds its allowed limit,
   returning a clear, standard error (`HTTP 429 Too Many Requests`) —
   ideally with information on when they can try again.
3. Works correctly even when the backend service runs across **multiple
   server instances**, not just one.
4. Adds **minimal latency** to every request — the check must be fast
   enough to feel invisible to legitimate traffic.
5. Scales to support a very large user base (on the order of **1 billion
   users**) without requiring excessive storage or infrastructure.
6. Has a clearly defined, deliberate behavior for what happens if the
   rate limiter itself becomes unavailable (fail-open vs. fail-closed).

## Out of Scope (for this iteration)

- Rate limiting logic that is specific to a single API only — the design
  should be **general-purpose**, usable across many services.
- Client-side/frontend-only enforcement — this is a UX enhancement, not a
  substitute for server-side enforcement, since it can be trivially
  bypassed.

## Success Criteria

- A legitimate user operating within normal limits should **never notice**
  the rate limiter exists.
- A user/source exceeding the defined limit should be **reliably and
  quickly blocked**, with a clear, correct HTTP response.
- The system should continue to function predictably — by design, not by
  accident — during partial failures (e.g., the rate-limiting store going
  down).

---

_See [`rate-limiter.md`](./rate-limiter.md) for the full design write-up,
including requirements, architecture, algorithms, and trade-offs._
