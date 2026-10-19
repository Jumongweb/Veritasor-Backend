Closes #1181, #1197, #1208

## Summary

Fix shutdown lifecycle ordering, one-time export download tokens, and SSRF exposure in webhook subscriptions and delivery.

## Changes

### Graceful shutdown (#1181)

- Replace the invalid `pool` import with the `PgClient` returned by `getPgClient()`.
- Stop background workers before the server and PostgreSQL pool are closed, so cleanup tasks can finish their database work.
- Close the attestation WebSocket server and terminate its connected clients as part of shutdown.
- Close idle HTTP keep-alive connections after initiating `server.close()`, allowing active requests to drain without idle sockets extending shutdown to the deadline.
- Document the shutdown order and add a shared call-log assertion for worker stop → server close → pool end.

### One-time export downloads (#1197)

- Consume tokens with one Redis Lua operation, preventing concurrent requests from both claiming the same token.
- Preserve the existing token expiry with `KEEPTTL` and execute the Redis operation through the circuit breaker.
- Peek at a token and check the export status before consuming it, so a pending export can be downloaded later with the same token.
- Add coverage for concurrent consumption, TTL preservation, and retrying a download after the export completes.

### Webhook SSRF protection (#1208)

- Require public HTTPS endpoints in production; allow HTTP only for localhost in non-production environments.
- Reject loopback, private, link-local, carrier-grade NAT, reserved, multicast, and IPv6 local addresses, along with localhost and `.internal` hostnames.
- Apply the shared URL validator to both subscription schemas and legacy business webhook registration.
- Resolve every destination address at dispatch time, fail closed on DNS errors or restricted results, and pin the HTTPS connection to the vetted addresses to prevent DNS rebinding.
- Do not follow redirects, so a webhook cannot redirect delivery to an internal address.
- Add schema and dispatcher regression tests for restricted URLs, private DNS results, and redirects.

## Verification

- `git diff --check` passes.
- Focused Vitest execution was attempted but could not start because this checkout has no installed `vitest` executable. Typechecking could not be run because `tsc` is also unavailable.
- The real-process SIGTERM integration scenario with idle HTTP keep-alive and an open WebSocket still needs execution-level coverage.
