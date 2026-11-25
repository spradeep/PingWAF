# Route 53 production readiness review

Proceed to isolated integration testing. The implementation can issue a combined apex and wildcard certificate. For a deployment with a control-plane-only server and a separate issuer agent, certificate publication remains a release blocker. A production cutover also requires evidence from the actual TLS listeners and proxy path. No external certificate order or DNS mutation was part of this review.

## Provider and control plane

| Review area | Evidence and outcome |
| --- | --- |
| Provider selection | The shared normalizer accepts case-insensitive Route53, Cloudflare/cf, Aliyun/ali, Tencent/dnspod and Huawei. The dispatcher constructs a Route53DnsTask for Route 53. Managed API paths reject unknown DNS providers, manual DNS-01 and credential or endpoint fields for Route 53. Standalone manual mode remains available. The Huawei converter derives its endpoint from the region only for Huawei. |
| AWS client | The lockfile pins aws-config 1.13.3 and aws-sdk-route53 1.138.0 with Tokio and the default HTTPS transport. Client construction fixes the commercial endpoint and signing region us-east-1. The SDK credential chain supports EC2 roles. The installed SDK's IMDS client acquires IMDSv2 tokens. Role acquisition and actual policy effectiveness require an isolated EC2 test. |
| Zone selection | An explicit ID skips listing. Discovery paginates public zones, selects the closest DNS-label suffix and rejects duplicate best matches. Both paths load the zone and compare its delegation set with DNS NS answers. A closer delegation or CNAME is rejected. This relies on the issuer's resolver and its current DNS view. |
| Shared TXT record | The apex and wildcard share one challenge owner name. The provider reads the current set and submits an exact DELETE plus CREATE in one batch for replacement. A competing edit invalidates the read snapshot. On recognized conflicts the writer rereads before retrying. TTL and unrelated TXT values survive, and cleanup only removes a value this task added. Complex routing sets are rejected. Unit tests cover batch construction and selection. Concurrent AWS writers and propagation require staging. |
| Partial failure | Zone resolution and SDK requests have deadlines. Automatic SDK retries are disabled to avoid replaying uncertain writes. Conflicts receive bounded application retries. Process loss or an ambiguous network result can leave a TXT value. Compare against pre-order state before any authorized cleanup. |
| Reporting and secrets | Attributed issuance failures travel through agent events to the control plane. The dashboard can mark a matching pending, PEM-less row failed and retain an existing certificate. Route 53 returns bounded messages without request bodies, and provider URL query strings are redacted from failure events. Merged-config and importer parse failures omit raw TOML. Database rows, config bundles, ACME account storage, caches and backups contain secrets. Live logs and production credentials were outside the review. |

## Distributed certificate lifecycle

| Question | Source-traced behavior |
| --- | --- |
| Where does issuance run? | Standalone Pingap runs the ACME service against its configured storage. Managed all-in-one runs it in the embedded data plane. An independent agent can run its own issuer. A control-plane-only server runs the importer and gRPC service and has no issuer loop. |
| How does material reach the database? | The server polls its local ACME state file every 30 seconds. Import requires a site-owned managed row, unambiguous hostname set, an ACME-marked state entry, matching SANs and a usable key pair. An independent agent reports events, but there is no issuer-to-server gRPC call carrying the resulting PEM and key. |
| Who receives the key? | The server builds site config from selected database material, including the private key. A config notification prompts refresh. A newly registered assigned agent gets the current material on initial sync. The issuer site must be assigned and its certificate selected. Verify assignment and fingerprint at every listener. |
| Disconnection, restart and reload | An existing proxy continues serving loaded material. The agent loads its persistent site cache before its background connection loop. Startup can build a data plane from cached sites and merge local ACME state. An empty cache needs an initial site bundle. A fresh incoming PEM takes precedence on reload. A stale database PEM arriving between local renewal and import could supersede the new local copy. These are source findings; there was no distributed outage test. |
| Consumer-only mode | PINGAP_DISABLE_ACME=true skips the managed issuance service and HTTP challenge setup while the proxy can load distributed TLS material. It is process-wide. There is no per-certificate election. Confirm distribution and restart behavior before using it in production. |
| Staging separation | Account records have separate staging and production names. Served certificate slots and site database rows can be shared accidentally. A staging toggle alone can leave an untrusted certificate in production. Use completely independent state, database, domains and registrations. |
| Renewal | The manual API marks a row pending and sends a configuration notification. The issuer still decides when to order from expiration, hostnames and backoff. Issuance failure does not clear an existing valid key pair. A successful Route 53 renewal retains its configuration for future cycles after sync. An uninterrupted switch on every listener remains a staging acceptance test. |

Source entry points are [managed mode and reload](../src/pingwaf.rs), [standalone](../src/main.rs), [ACME service](../pingap-acme/src/lets_encrypt.rs), [state import](../pingwaf-server/src/acme_state_sync.rs), [gRPC](../pingwaf-server/src/grpc/control_plane.rs), [bundle conversion](../pingwaf-server/src/grpc/config.rs) and [agent cache](../pingwaf-agent/src/cache/mod.rs).

**Smallest proposed change for a remote issuer.** Persist an issuer agent ID per managed certificate and convey issuance authority only to that agent. Add one authenticated publication RPC carrying certificate ID, config generation, chain and key. Check agent identity, ownership, assignment, SANs, key match and expected generation before atomically promoting the result. Duplicate submissions should acknowledge the existing generation, and stale results should preserve the current certificate. Reuse the existing authorized site-config fanout. Have the issuer retain an unpublished result for retry across outages and reloads. Test failover, stale publication, unauthorized publication, failed renewal, disconnected consumers and cold registration. This proposal requires design approval.

## Forwarded identity and challenge pages

The earlier reverse-proxy change in b8c3dcf5 enabled default headers on generated routes. Commit e18c2a4e also calculates identity at the configured trust boundary, strips untrusted chain prefixes and replaces duplicate origin-bound fields. Managed site fallback and explicit routes use the same opt-in. Standalone locations use their standard header setting and configured proxy ranges. A direct untrusted peer supplies the socket address. At a trusted peer the scheme accepts a single HTTP or HTTPS value, and host comes from the request authority. Correct scheme depends on the ingress sanitizing its input. WAF, challenge/rate and GeoIP request paths take client identity from the trusted resolver or request context. Access logs use the context. Disabled presets retain existing forwarding and legacy inbound identity behavior.

The custom 503 challenge scope renders the configured template with its required markup and injected verification script. Existing clearance-cookie handling, including the cookie splitting fix present in the base, remains in place. The changes do not edit challenge templates, signing or cookie parsing. The deployed 123Greetings logo, CSS, JS/CAPTCHA completion and clearance across HAProxy and Apache need the approved template and a staging browser. There is no verified production rendering in this review.

## Local verification

The checks ran on Amazon Linux 2023 aarch64 with Rust 1.100.0 and the repository's default OpenSSL feature selection. The build commands use the host profile at `/etc/profile.d/pingwaf-build.sh`.

| Check | Result |
| --- | --- |
| Formatting | Cargo fmt passed for the root and all affected crates. |
| Dashboard | Locked install and production build passed with Node 26.12.1, npm 11.20.0 and a 1 GiB Node heap. Vite reported large chunks. |
| DNS providers and lifecycle | All 15 pingap-acme library tests passed, including Route 53 zone selection and TXT changes, Huawei and Aliyun helpers, failure attribution, retries, staging defaults and account separation. |
| Config | The Route 53 service URL and hosted-zone configuration test passed. |
| Forwarding | All 26 shared header tests and the origin-header regression passed on the final source. |
| Rust compilation | Both default-feature binaries passed `cargo check --offline --locked -p pingwaf --bins`. The server passed its `full` feature check, and the agent passed its `tracing,edge` feature check. The affected library test targets also passed `cargo check --tests`. |
| Clippy | Passed for the six affected library crates and the main binary. Cargo emitted existing manifest warnings. |
| Server tests | The server test build was stopped before linking after extended swapping. The API and gRPC tests type-checked. Production certificate fanout, HAProxy/Apache compatibility and the deployed challenge page require isolated integration testing. |

## Footprint and rollout

The SDK is linked into issuer-capable builds, including the edge agent. A process-wide lazy client reuses SDK configuration across Route 53 orders. Agents without a Route 53 order skip this initialization and have no new Route 53 polling task. The existing ACME scheduler still runs where enabled. Release-size delta, peak compilation RSS, steady-state RSS, cold start and handshake latency require matched base and candidate release builds. A development check is insufficient to size a small EC2 host.

The [AL2023 deployment guide](route53-deployment.md) includes the exact server, frontend and edge build commands, IAM policy, certificate JSON, issuer and consumer systemd settings, staging isolation, renewal and restart checks, curl procedure and rollback. Rebuild all three artifacts. Deployment, upstream push and production certificate operations require separate approval.

## Changed files and commit boundaries

- 5c4f83f5 adds the provider, lockfile entries, config normalization, ACME task integration, server validation/import, certificate conversion, dashboard choices and locales, API docs and provider README. It changes 19 files, with 1,181 insertions and 59 deletions.
- e18c2a4e changes request context and identity resolution in pingap-core, generated location headers in pingap-location, request handling and outgoing headers in pingap-proxy, and managed route conversion in src/pingwaf.rs. It changes six files, with 296 insertions and 32 deletions.
- Commit a4aabbc1 tightens managed DNS-01 validation and state import, sanitizes config diagnostics and updates the dashboard and API reference. It changes eight files, with 53 insertions and 24 deletions. The documentation commit contains the project instructions, this review and the deployment guide. To enumerate every changed path against the requested base, run `git diff --stat 7efcbfcd..HEAD` and `git diff --name-status 7efcbfcd..HEAD`.
