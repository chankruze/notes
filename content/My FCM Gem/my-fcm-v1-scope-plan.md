---
title: Scope and plan for Firebase Cloud Messaging gem
tags:
  - firebase
  - ruby
created: 2026-03-03
updated:
status: draft
---
Finally after looking into both, I have decided the scope of work for v1.

1. Auth: service-account JSON + OAuth access token minting with cache.
2. Core API: `send`, `send_multicast`, `send_all`.
3. Fanout concurrency: thread pool via `concurrent-ruby`.
4. Retry/backoff: transient HTTP + transient FCM statuses.
5. Rate limiting: in-memory token bucket (Redis adapter as optional extension point).
6. Structured errors: normalized error classes + invalid-token detection.
7. Observability: structured logs + optional `ActiveSupport::Notifications`.
8. README with your 3 important notes (multicast reality, invalid token cleanup, topics for large fanout).

**Key changes to my draft**
1. Add a cache abstraction now:
   - `TokenStore` interface: `read/write/delete` with `MemoryStore` default.
   - `RedisStore` optional (extra dependency, optional group).
2. Avoid `Marshal` deep dup in hot path:
   - use `Oj.dump` + `Oj.load(symbol_keys: true)` cloning or manual merge.
3. Retry policy should inspect both:
   - HTTP transient codes (`429, 5xx, 408`).
   - FCM error status (`UNAVAILABLE`, `INTERNAL`) inside response body.
4. Explicit invalid-token classifier:
   - map `UNREGISTERED`, `NOT_FOUND`, `NOT_REGISTERED`, `INVALID_ARGUMENT`(token-specific cases) to `InvalidTokenError`.
5. Add rate limiter before dispatch:
   - per-process limiter in v1, pluggable interface for Redis later.
6. Keep `send` API for Admin-SDK parity, but document Ruby `Object#send` caveat.

**Repo plan**
1. Create a new repo: `fcm_http_v1` .
2. Initial structure:
   - `lib/.../client.rb`, `auth.rb`, `config.rb`, `result.rb`, `errors.rb`, `rate_limiter.rb`, `token_store/*.rb`, `messaging.rb`.
3. Add CI:
   - Ruby 3.2/3.3 matrix, rubocop, rspec, mutation-safe retry tests.
4. Add test strategy:
   - Unit tests with WebMock/VCR-free stubs for HTTP.
   - Concurrency tests for `send_multicast`/`send_all`.
   - Token cache expiry and refresh race tests.
5. Release plan:
   - `0.1.0` internal alpha.
   - Integrate in your backend behind feature flag.
   - `0.2.0` add Redis store + instrumentation polish.

**Acceptance criteria for v1**
1. `send` works for token/topic/condition payloads.
2. `send_multicast` and `send_all` return per-message structured results.
3. Invalid tokens are surfaced in a dedicated list for DB cleanup.
4. Retries happen only for retryable errors, with jittered backoff.
5. Token minting is cached and thread-safe.
6. Throughput and error metrics are visible via logs/events.