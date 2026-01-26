# Technical Review: Envoy Claims in Research Sandboxing Egress Control Document

## Summary

I verified every Envoy-related technical claim in the document against the Envoy source code and public documentation. **The document is technically sound overall.** Most claims are accurate and precise. I found **one inaccuracy** (UDP support claim), **one statement needing nuance** (hot restart), and **one claim that could be more precise** (DNS filter architecture). Details below.

---

## Verified Claims (Correct)

### 1. Resource contention primitives
**Claim:** "Envoy provides primitives to address this (overload manager, per-filter-chain connection limits, upstream circuit breakers)"

**Verdict: Accurate.**

All three exist and work as described:
- **Overload manager**: Defined in `api/envoy/config/overload/v3/overload.proto`. Monitors system resources (memory, CPU, file descriptors) and triggers actions like stopping new connections, disabling keepalive, or returning 503s. Protects the Envoy process itself.
- **Per-filter-chain connection limits**: Implemented as network filter `envoy.filters.network.connection_limit` (`api/envoy/extensions/filters/network/connection_limit/v3/connection_limit.proto`). The docs explicitly state: "each filter chain has an independent connection limit."
- **Upstream circuit breakers**: Defined in `api/envoy/config/cluster/v3/circuit_breaker.proto`. Controls max connections, pending requests, concurrent requests, retries, and connection pools per upstream cluster.

### 2. Forward proxy with HTTP CONNECT
**Claim:** "Natively supports HTTP. HTTPS and arbitrary TCP streams are proxied via HTTP CONNECT tunneling."

**Verdict: Accurate.**

- Envoy supports forward proxy mode: `docs/root/intro/arch_overview/http/http_routing.rst` states "Envoy also has the ability to be configured as forward proxy."
- HTTP CONNECT tunneling is implemented in `source/extensions/transport_sockets/http_11_proxy/connect.h` with RFC 9110 compliance. The proto (`api/envoy/extensions/transport_sockets/http_11_proxy/v3/upstream_http_11_connect.proto`) documents: sends HTTP/1.1 CONNECT request, processes response, passes connection to underlying transport socket on HTTP 200.

### 3. Transparent proxy: original_dst
**Claim:** "Envoy recovers the original destination"

**Verdict: Accurate.**

- `original_dst` listener filter (`api/envoy/extensions/filters/listener/original_dst/v3/original_dst.proto`) calls `getOriginalDst(socket)` to recover the original destination after iptables REDIRECT.
- `ORIGINAL_DST` cluster type (`api/envoy/config/cluster/v3/cluster.proto`, line 77) dynamically creates upstream hosts based on recovered addresses.

### 4. tls_inspector and SNI extraction
**Claim:** "the tls_inspector listener filter extracts SNI from the TLS ClientHello without decrypting, enabling hostname-based policy without TLS termination"

**Verdict: Accurate.**

- `tls_inspector` (`source/extensions/filters/listener/tls_inspector/tls_inspector.cc`) calls `SSL_get_servername()` on the ClientHello to extract SNI and sets it via `setRequestedServerName()`. It does **not** terminate TLS — the connection continues to upstream with TLS intact.
- Stats track `sni_found` / `sni_not_found` confirming this is a metadata extraction step.

### 5. Source-IP-matched filter chains
**Claim:** "A single shared listener with source-IP-matched filter chains is the right architecture"

**Verdict: Accurate.**

- `FilterChainMatch` in `api/envoy/config/listener/v3/listener_components.proto` includes `source_prefix_ranges` (line 141) and `direct_source_prefix_ranges` (line 132) fields for CIDR-based source IP matching.
- `source/common/listener_manager/filter_chain_manager_impl.cc` implements the matching logic. A single listener can have many filter chains, each selected by source IP prefix.

### 6. Access logs and metrics
**Claim:** "Envoy produces access logs and metrics natively"

**Verdict: Accurate.**

- Access log sinks in `source/extensions/access_loggers/`: file, gRPC, Fluentd, OpenTelemetry, Wasm, stdout/stderr.
- Stats sinks in `source/extensions/stat_sinks/`: StatsD, DogStatsD, Graphite, gRPC metrics service, OpenTelemetry, Hystrix.

### 7. Cilium's embedded Envoy
**Claim:** "Cilium's own architecture proves this: it delegates L7 HTTP/HTTPS enforcement to an embedded Envoy, with a separate built-in DNS proxy for FQDN policy."

**Verdict: Accurate.**

- Cilium ships a custom Envoy build (`cilium/proxy` on GitHub) with Cilium-specific filters for L7 policy enforcement. Envoy runs as a separate process within the Cilium agent pod.
- DNS proxy is a separate Go component in the Cilium agent, not part of Envoy. It intercepts DNS responses per-endpoint to map FQDNs to IPs for L3 policy rules.

---

## Issues Found

### Issue 1: UDP support claim is inaccurate (factual error)

**Claim:** "Raw UDP and clients that do not support HTTP_PROXY remain unsupported"

**Problem:** Envoy now supports UDP tunneling, making "unsupported" incorrect.

- **CONNECT-UDP** (RFC 9298) is implemented in `docs/root/intro/arch_overview/http/upgrades.rst` (lines 209-224), allowing UDP tunneling through HTTP proxy. Status: **alpha**.
- **Raw UDP tunneling** via the UDP Proxy filter (`api/envoy/extensions/filters/udp/udp_proxy/v3/udp_proxy.proto`) supports tunneling UDP over HTTP CONNECT or HTTP POST. Requires HTTP/2 minimum.

**Suggested fix:** Change to something like: "Raw UDP is not natively supported through HTTP_PROXY. Envoy has experimental UDP tunneling (CONNECT-UDP per RFC 9298), but this is alpha-status and requires HTTP/2, making it impractical for the forward proxy use case where clients set HTTP_PROXY. Clients that do not support HTTP_PROXY also remain unsupported."

This is a minor point — the practical conclusion (forward proxy doesn't solve UDP) is still correct, but the blanket "unsupported" statement is technically wrong.

### Issue 2: Hot restart — correct but could be more precise (nuance)

**Claim:** "Envoy's hot restart mechanism also enables zero-downtime upgrades"

**Nuance:** The hot restart mechanism (`source/server/hot_restart_impl.h`, `docs/root/intro/arch_overview/operations/hot_restart.rst`) works by:
1. Starting a new Envoy process
2. Passing listening sockets from old to new process via Unix domain sockets
3. **Draining** existing connections on the old process (configurable via `--drain-time-s`)
4. Shutting down the old process after drain

The docs explicitly state: "existing connections are not transferred to the new envoy process: they must complete during the drain process or be terminated." This means:
- New connections immediately go to the new process: **zero downtime for new connections**.
- Existing long-lived connections are gracefully drained, not seamlessly transferred.

**Suggested fix:** The claim is not wrong — "zero-downtime upgrades" is the standard way to describe hot restart — but adding "(existing connections are drained, not transferred)" would be more precise for an audience making infrastructure decisions. For a node-level proxy serving many containers, the drain behavior matters.

### Issue 3: DNS filter architecture — correct but imprecise mechanism (clarity)

**Claim:** "Envoy's DNS filter is configured once per listener with no source-IP branching"

**Verdict: Correct conclusion, but the mechanism deserves clarification.**

The DNS filter (`source/extensions/filters/udp/dns_filter/`) is a **UDP listener filter**, not a network filter placed in filter chains. The reason it lacks source-IP branching is architectural:
- TCP listener filters support `ListenerFilterChainMatchPredicate` for conditional application.
- **UDP listener filters have no matcher support at all** — the `UdpListenerFilterManager::addReadFilter()` interface takes no matcher parameter.
- Since UDP listeners don't use the TCP-style filter chain matching infrastructure, per-source-IP configuration is structurally impossible.

The doc's statement is correct, but a reader might wonder *why* the DNS filter can't branch on source IP when TCP filter chains can. The answer is that the DNS filter lives in a fundamentally different part of the listener architecture (UDP listener filters vs. TCP filter chains).

**Suggested fix:** No change required — the conclusion and its implications are correct. But if more precision is desired: "Envoy's DNS filter is a UDP listener filter with no per-source-IP configuration capability (UDP listener filters lack the matcher infrastructure available to TCP filter chains), so per-container DNS policy requires an external policy DNS server."

---

## Comparison Table Review

The comparison table is accurate. One minor clarification:

| Cell | Comment |
|------|---------|
| "All TCP; UDP less mature" (Transparent Proxy protocol support) | Accurate characterization. UDP support exists but is alpha-status. |
| "HTTP natively; HTTPS and TCP via HTTP CONNECT" (Forward Proxy) | Accurate. See Issue 1 above re: UDP. |

---

## Diagram Review

Both diagrams accurately represent the described architectures:
- **Forward Proxy diagram**: Correctly shows HTTP_PROXY flow, JWT validation, DNS resolution at proxy, and iptables DROP for bypass prevention.
- **Transparent Proxy diagram**: Correctly shows iptables REDIRECT, original_dst + tls_inspector in Envoy, xDS control plane pushing config, and external policy DNS server.
- The `original_dst` and `tls_inspector` labels in the transparent proxy diagram match actual Envoy extension names.

---

## Overall Assessment

The document demonstrates strong understanding of Envoy's architecture. The one factual error (UDP blanket "unsupported") is minor and doesn't affect the recommendation. The hot restart nuance and DNS filter mechanism clarity are optional improvements. All other Envoy claims — resource management primitives, forward proxy mode, HTTP CONNECT tunneling, transparent proxy components, filter chain matching, observability, and the Cilium/Envoy relationship — are verified correct against the source code.
