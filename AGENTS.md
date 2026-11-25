# PingWAF development requirements

## Certificate management

Implement reusable provider and API behavior on the feature branch. Keep customer deployment settings in examples and operations documentation. Target Amazon Linux 2023, the workspace Rust version and the supported OpenSSL or rustls feature combination. Check release size, cold startup, RSS and credential-provider activity on small agents when adding SDKs. Initialize AWS clients on first use and reuse them across renewals.

Trace certificate ownership through the mode entry point, ACME state, database and the authenticated gRPC site bundle. Validate initial issuance, renewal, persistence, disconnected service, new-agent enrollment and protection of the previous valid PEM. Use one designated issuer per certificate. Verify the public certificate fingerprint at each consumer before relying on issuer-only operation.

## Reverse proxy and challenge regressions

Preserve the 123Greetings reverse proxy opt-in on site fallback and explicit routes. Exercise trusted HAProxy and direct connections, repeated and malformed forwarding fields, TLS termination, original host and Apache client-IP interpretation. Verify the same client identity in access logs, WAF rules, rate limits, GeoIP and challenge verification. Trust forwarded identity only at an explicitly configured proxy boundary.

Preserve the customized JavaScript and CAPTCHA pages, logo, CSS, required DOM IDs, injected verification script, clearance signing secret, cookie parsing and host/IP binding. Verify the configured template using a disposable site with the approved branding fixture. Keep customer assets and secrets out of generic upstream changes.

## Operations

The 123Greetings agent unit requires `PINGWAF_ENABLE_REVERSE_PROXY_HEADERS=true`, `MIMALLOC_ARENA_RESERVE=67108864` and `MIMALLOC_ARENA_EAGER_COMMIT=0`. Isolated staging uses `PINGAP_ACME_STAGING=true`. Certificate consumers use `PINGAP_DISABLE_ACME=true` after certificate distribution has been verified.

Use separate staging databases, agent registrations, state directories, DNS permissions and test hostnames. Protect caches and backups as private-key material. On the AL2023 build host, source `/etc/profile.d/pingwaf-build.sh` before invoking Cargo. Build versioned artifacts separately from deployed executables. Retain working binaries, certificate state and ACME accounts for rollback.

On this project, work on `feature/acme-route53` and keep provider and forwarding changes in separate commits. Pushing, merging, production credential access, actual DNS mutations, certificate orders, service changes and deployment require explicit authorization.
