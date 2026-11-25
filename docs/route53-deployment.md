# Route 53 deployment on Amazon Linux 2023

## Choose the issuer

The currently supported central issuer is the data plane embedded in `pingwaf all-in-one`. Assign it the site and select its ACME certificate in the site's TLS settings. The server imports the embedded issuer's local state and distributes the database PEM and key to authorized agents. Use one site-owned certificate row with the exact apex and wildcard names. Where sites share a certificate, use the issued material and assign renewal to its owner.

A control-plane-only server plus an independent issuer agent needs an additional certificate publication mechanism. Keep that topology in preproduction until the mechanism described in the readiness report is implemented and tested. Network access to a common Pingap config store does not implement publication to the PingWAF database.

## Build artifacts

Use the target architecture and the AL2023 native build prerequisites from the repository build guide and Makefile. The Rust workspace requires Rust 1.96 or later. The dashboard package specifies Node 26.12.1 or later and npm 11.20.0. Build the frontend before the control plane so rust-embed includes its assets. The UI, control-plane validation, issuer and consuming agent header logic all changed.

```sh
. /etc/profile.d/pingwaf-build.sh
(cd web && npm ci && NODE_OPTIONS=--max-old-space-size=1024 npm run build)
cargo build --locked --release --bin pingwaf --features full
cargo build --locked --release --bin pingwaf-agent --features tracing,edge
sha256sum "$CARGO_TARGET_DIR/release/pingwaf" "$CARGO_TARGET_DIR/release/pingwaf-agent"
```

The build profile sets RUSTUP_HOME to `/opt/build/rustup`, CARGO_HOME to `/opt/build/cargo` and CARGO_TARGET_DIR to `/opt/build/target`. It adds Cargo to PATH, uses one build job, disables incremental compilation and release LTO, and uses four release codegen units. Use the same target, allocator, TLS backend, linker and features for baseline measurements. Record stripped binary bytes with `stat -c %s`, peak build RSS with `/usr/bin/time -v` around each clean build, and the time from process start to a successful local TLS handshake. Compare idle and active issuer RSS with the kernel's `VmRSS` or systemd cgroup metrics. Use a separate build host if paging dominates. A development-mode Cargo check does not provide release or runtime measurements.

## IAM and certificate settings

Attach the policy below to the designated issuer's EC2 instance role. Require IMDSv2 on that instance. Replace the example zone ID and record name. The exact-zone configuration avoids account-wide discovery permission. For discovery add a separate Allow statement for `route53:ListHostedZones` on `*`. The issuer's resolver must see the public delegation. [Route 53 IAM condition keys](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/specifying-conditions-route53.html) · [SDK instance metadata support](https://docs.aws.amazon.com/sdkref/latest/guide/feature-imds-credentials.html)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["route53:GetHostedZone", "route53:ListResourceRecordSets"],
      "Resource": "arn:aws:route53:::hostedzone/ZEXAMPLE"
    },
    {
      "Effect": "Allow",
      "Action": "route53:ChangeResourceRecordSets",
      "Resource": "arn:aws:route53:::hostedzone/ZEXAMPLE",
      "Condition": {
        "ForAllValues:StringEquals": {
          "route53:ChangeResourceRecordSetsNormalizedRecordNames": ["_acme-challenge.123greetings.com"],
          "route53:ChangeResourceRecordSetsRecordTypes": ["TXT"],
          "route53:ChangeResourceRecordSetsActions": ["CREATE", "DELETE"]
        }
      }
    }
  ]
}
```

Set both hostnames on the site. In the certificate API use the following fields, supply the actual owner site ID and select or activate the certificate. The dashboard exposes the same settings. Credentials come from the issuer's AWS SDK credential chain.

```json
{
  "domain": "123greetings.com,*.123greetings.com",
  "acme_email": "tls-operations@example.org",
  "auto_renew": true,
  "activate": true,
  "acme_challenge_type": "dns-01",
  "acme_dns_provider": "route53",
  "acme_dns_config": {"hosted_zone_id": "ZEXAMPLE"}
}
```

## Sample systemd drop-ins

Layer these settings on the repository's installed units. Keep existing database, gRPC CA, API-key file, service user and bind configuration. The all-in-one service uses `pingwaf all-in-one` and a persistent private cache directory. Point its state importer at the embedded agent's actual state file. A pure `pingwaf server` process only serves the control plane.

```ini
# Embedded issuer service
[Service]
UMask=0077
Environment="PINGWAF_CACHE_DIR=/var/lib/pingwaf/issuer-cache"
Environment="PINGWAF_ACME_STATE=/var/lib/pingwaf/issuer-cache/acme_state.toml"
Environment="PINGWAF_ENABLE_REVERSE_PROXY_HEADERS=true"
Environment="MIMALLOC_ARENA_RESERVE=67108864"
Environment="MIMALLOC_ARENA_EAGER_COMMIT=0"
```

Verify effective CLI and TOML paths agree with the service settings. The agent supports `pingwaf-agent --server-url https://control.example.org:9090 --server-ca-cert /etc/pingwaf/grpc-ca.pem --api-key-file /etc/pingwaf/agent.key --cache-dir /var/lib/pingwaf/cache`. Consumer agents need gRPC access and a persistent private cache. Their instance roles can omit all Route 53 privileges.

```ini
# Agent service
[Service]
UMask=0077
Environment="PINGWAF_ENABLE_REVERSE_PROXY_HEADERS=true"
Environment="MIMALLOC_ARENA_RESERVE=67108864"
Environment="MIMALLOC_ARENA_EAGER_COMMIT=0"
```

After checking the certificate and key distribution with an assigned consumer, set the following on each consumer before admitting traffic or enabling a production order. Keep ACME enabled on the designated issuer.

```ini
[Service]
Environment="PINGAP_DISABLE_ACME=true"
```

## Isolated staging and production gates

1. Obtain approval for each external test. Create a disposable public test zone and matching test hostnames, separate control-plane database, CA/API-key trust domain, issuer and consumer registrations, persistent directories and listeners. Restrict the staging role to that zone. Start from empty, newly provisioned staging directories. Use this staging-only unit setting on the issuer.

   ```ini
   [Service]
   Environment="PINGAP_ACME_STAGING=true"
   ```
2. Use only the staging hostnames, site and zone ID in the example. Record the pre-test TXT set and TTL. Request one combined order. Observe the certificate event stream, agent order and public TXT answers, including coexisting apex and wildcard values. Verify cleanup restores unrelated data. An ambiguous write timeout can leave a token requiring review.
3. Check the leaf issuer, serial, expiry, SANs and SHA-256 fingerprint separately at the issuer and **each** consumer using the correct SNI. Example for a disposable listener follows. Compare only public certificate information.

   ```sh
   openssl s_client -connect "$AGENT_ADDRESS:443" -servername "$TEST_HOST" </dev/null 2>/dev/null |
     openssl x509 -noout -issuer -serial -dates -ext subjectAltName -fingerprint -sha256
   ```
4. Register a second assigned staging agent with an empty cache and no DNS write permission. Check its TLS fingerprint after full sync. With approval, isolate a consumer from the disposable control plane, repeat the handshake, restart the disposable agent with its persistent cache and repeat. Restore connectivity, change a non-TLS site setting and check again after reload. Verify a newly registered unassigned agent has no site material. Record agent-visible events and public fingerprints without dumping cache or PEM files.
5. Verify renewal with an existing certificate inside the renewal window in the isolated issuer. Record the continuously served fingerprint while a renewal is pending and during a deliberately failed staging validation. Check the certificate event and the previous valid TLS chain. On successful renewal compare the new serial/fingerprint across agents and after a fresh registration. The dashboard renewal request updates status and sends a config notification. The scheduler still checks expiry, SANs and retry backoff.
6. Promote **only the reviewed binary and configuration pattern**. Production uses distinct empty/new issuer identity where appropriate, or the existing production state and accounts. Retain the production database and caches. Confirm staging is disabled in the effective production service and every issuer has the intended production database and role. Staging and production accounts have different storage names, but certificate PEMs share configuration slots if storage is reused.
7. After production approval, verify the previous production chain at all listeners, enable the single selected production order, check a publicly trusted full chain and repeat per-agent and restart checks on a canary. Keep the previous valid path available until distribution is proven. Roll out consumers gradually.

## Forwarded-header acceptance test

For a trusted HAProxy ingress configure the site's proxy trust ranges with its actual socket addresses. Enable its real-client header and select nearest hop for a single sanitizing proxy or trusted-chain traversal for several proxies. In standalone Pingap configure `basic.trusted_proxies` and enable the location's standard reverse proxy headers. The outer trusted proxy must overwrite incoming protocol and identity headers. Apache mod_remoteip must trust only the actual WAF addresses and any approved intermediaries. Restrict access to the backend to that path.

The enabled preset replaces XFF, X-Real-IP, protocol, host and port on the origin request and clears the RFC Forwarded field. An untrusted peer supplies only its socket IP. A trusted XFF chain is read from the nearest end and stops at the first untrusted IP. X-Forwarded-Host uses the incoming request authority including its port. The forwarded scheme uses a single trusted http/https value or the local TLS state. With the preset disabled, ordinary forwarding and explicitly configured header directives apply. Treat incoming forwarded headers at the backend as untrusted in that configuration.

On a disposable origin that records received headers, compare a direct connection with one through the configured HAProxy, including an HTTPS frontend with HTTP between HAProxy and PingWAF. Example direct request follows. Use the staging CA for listener verification.

```sh
curl --cacert "$TEST_CA" --resolve "$TEST_HOST:443:$AGENT_IP" \
  -H 'X-Forwarded-For: 192.0.2.51' -H 'X-Forwarded-For: 192.0.2.52' \
  -H 'X-Forwarded-Proto: http' -H 'X-Forwarded-Host: example.invalid' \
  "https://$TEST_HOST/header-check"
```

Check for one field per generated header. Compare the origin XFF chain, Apache logical client address (`%a`), socket peer (`%{c}a`), original authority and scheme. Cross-check the WAF event, access log, rate-limit bucket, GeoIP input and challenge verification for that request. Exercise both generated locations and explicit routes and confirm the JS and CAPTCHA pages display the approved logo and CSS, complete verification and accept the resulting clearance cookie.

## Rollback

Before any approved rollout, retain the previous binaries, unit definitions, config snapshots, database backup, private issuer and agent state and challenge template/assets. Record public certificate fingerprints and expiry. Drain one consumer at a time, restore its previous binary and compatible config, and verify SNI, chain, forwarded identity and clearance before returning traffic. If the new issuer is faulty, stop further orders through the approved service procedure, keep serving the known valid certificate, and restore the prior issuer only after confirming sole ownership. Never restore a stale database or key over a newer working certificate as a routine binary rollback.
