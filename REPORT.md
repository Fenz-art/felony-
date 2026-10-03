# Project Report: HTTP Forward Proxy Server

**Course:** Computer Networks  
**Project:** Project 1, Designing a Proxy Server  
**Submission date:** 11 October 2026

## Team

| Member | Roll number | Responsibility |
|---|---:|---|
| Kumar Kushang | 2405734 | Proxy core, concurrency, cache, integration, performance integration |
| Tonisha Ghosh | 24051896 | TCP listener, HTTP parsing, origin connections, relay |
| Ahona Misra | 24051078 | Host access policy, configuration, request rejection |
| Aastha Ray | 24051975 | Logging, metrics, regression testing, performance measurement |

## Problem and Objectives

A forward proxy intermediates HTTP requests between clients and origin servers. The project implements a local, concurrent HTTP proxy that parses and forwards requests, applies a hostname policy, caches eligible responses, logs traffic, records metrics, handles tested failure cases, and compares direct and proxied request performance.

## Architecture and Request Lifecycle

The production program is divided into `main.c`, `proxy_server.c`, `cache.c`, `access_control.c`, and the existing HTTP handler, logger, and metrics modules. `main.c` parses configuration and initializes/destroys shared modules. The proxy module owns listening and request orchestration; cache and access-control modules remain independent of socket handling.

A request is read into a bounded header buffer, parsed by the HTTP handler, checked against the ACL, and looked up in the cache. A hit is returned without an origin connection. A miss connects to the origin, constructs an origin-form GET, forwards the complete response, and stores it only if eligible. The worker records the request and releases its sockets and worker slot.

## Concurrency and Resource Controls

The listener binds to IPv4 loopback by default. At most 64 detached workers run at once; accepted clients above capacity receive 503. Shutdown stops accepting and waits for active workers before `main` destroys metrics, logger, ACL, and cache state. The cache uses a mutex only while accessing its in-memory entries. Metrics updates and logger writes are synchronized.

The client-header deadline is 5 seconds, origin-response forwarding has an 8-second total deadline, and client/origin writes have 5-second deadlines. Socket inactivity timeouts remain backstops. Origin TCP connection establishment defaults to 5 seconds. System DNS resolution has no independent deadline.

## HTTP Support

The implemented subset is HTTP/1.0 and HTTP/1.1 GET in absolute-form or origin-form. The proxy forwards origin-form upstream and preserves the query. Headers are limited to 8 KiB. Unsupported methods, request bodies, HTTPS/CONNECT, IPv6 authorities, and malformed inputs are rejected. Responses with Content-Length are forwarded; chunked responses are passed through unchanged but never cached.

## Cache

The in-memory cache defaults to 32 entries, a 30-second TTL, and a 1 MiB maximum response. Keys contain lowercased `host:port/path?query`; capacity eviction removes the oldest inserted entry. Only successful GET responses with a valid Content-Length and within the size limit are cached. Expired entries miss and refetch. This is an educational cache, not an RFC-complete implementation: it does not process Cache-Control, Vary, or validators.

Integration tests prove a cache HIT by comparing the response and confirming that the origin request counter remains unchanged. Separate tests prove authority, port, path, and query isolation, expiry/refetch, non-success ineligibility, size ineligibility, and chunked non-caching.

## Access Control

The ACL loads simple `allow hostname` and `block hostname` rules. Matching is exact and case-insensitive; one terminal DNS root dot is normalized. Explicit BLOCK wins over ALLOW, and the default policy allows unlisted hosts. The sample configuration blocks `blocked.test`. The proxy returns 403 before cache lookup or origin connection. Integration tests verify the origin counter is unchanged for both a blocked name and its terminal-dot form.

## Logging, Metrics, and Error Handling

The existing logger records timestamp, client, method, destination, resource, status, cache result, response bytes, and latency. The existing synchronized metrics module records total/success/failure requests, cache hits/misses, bytes, and latency. Integration verifies real HIT, MISS, BLOCK, malformed, gateway, timeout, overload, and successful traffic; metrics satisfy success plus failure equals total and hits plus misses equals total for this request-accounting policy.

The tested errors are 400 for malformed/unsupported input, 403 for blocked destinations, 502 for origin connection failure, 503 when worker capacity is exhausted, and 504 for timeout cases. Follow-up requests demonstrate the server remains available after these failures and client disconnects.

## Testing and Results

The project is built with `make`; `make test` runs the existing handler/logger/metrics regressions, cache and ACL unit tests, a controlled local origin, actual proxy integration tests, concurrency tests, and benchmarks. The final verified command was `make clean && make && make test`; all checks passed.

The integration suite verifies forwarding and response bodies, a 1 MiB response byte-for-byte, ordinary and conflicting-header chunked passthrough, malformed and oversized requests, origin failures, client disconnect recovery, slow-trickle deadlines, the 64-worker capacity boundary, cache/ACL origin-counter behavior, and concurrency at 1, 2, 5, 10, 25, and 50 clients.

### Hardcore Validation Results

The final hardening pass included live proxy validation beyond the repository suite. Observed results were:

- `SLOW_HEADER_RECV -> HTTP 400`
- `VALID_AFTER_SLOW -> HTTP 200`
- `CONCURRENCY_COUNT=250`
- `ERR_COUNT=0`

These observations demonstrate that malformed or stalled headers are rejected and the service recovers, and that the real proxy handled a 250-client burst without errors while remaining under its 64-worker-capacity model. This is a successful burst/recovery check rather than evidence of 250 simultaneous active workers.

GCC `-fanalyzer` completed without warnings for production and test sources. Editor diagnostics reported no errors. ASan/UBSan/TSan could not link because the required runtime libraries are missing in the environment. Valgrind is not installed; neither dynamic sanitizer analysis nor Valgrind leak analysis is claimed as passed.

## Performance Methodology and Results

The direct mode connects the benchmark clients to the local origin. Proxy mode connects them to the running proxy and uses the same origin, path, and response. Caching was disabled for the comparison. Three repetitions were run at each concurrency level. The benchmark counts successful requests after validating response status and framing; integration tests separately verify deterministic response bodies byte-for-byte.

| Clients | Direct latency | Proxy latency | Observed overhead | Direct req/s | Proxy req/s |
|---:|---:|---:|---:|---:|---:|
| 1 | 0.303 ms | 0.460 ms | 51.65% | 2,502.2 | 1,584.9 |
| 5 | 0.250 ms | 0.573 ms | 129.33% | 10,617.1 | 7,332.6 |
| 10 | 0.200 ms | 0.457 ms | 128.33% | 20,307.5 | 12,073.0 |
| 25 | 0.297 ms | 0.507 ms | 70.79% | 20,879.0 | 18,346.4 |
| 50 | 0.160 ms | 0.480 ms | 200.00% | 31,294.4 | 22,415.1 |

These are short localhost microbenchmarks and are environment-specific. Scheduling and machine load can dominate sub-millisecond measurements. The values are observations from this run, not statistically rigorous results or general claims about Internet-scale proxy performance.

## Limitations and Security

HTTPS/CONNECT, IPv6 authorities, non-GET methods, request bodies, client authentication, client-address ACLs, and RFC-complete cache semantics are outside the implemented scope. Chunked responses are forwarded but not cached. DNS resolution is not independently timed. Deadlines bound the tested header, response, and write phases but are not production-grade network QoS guarantees.

The proxy is intended for a controlled laboratory demonstration. Loopback is the default bind address. The ACL defaults to allow unlisted destinations and there is no client authentication; do not expose the service publicly or change its bind address without adding appropriate access controls.

## AI Usage Declaration

AI tools were used as supplementary aids for networking concepts, design alternatives, debugging approaches, edge-case analysis, tests, unfamiliar implementation flows, and documentation. AI-assisted suggestions were reviewed and tested by the team. The team remains responsible for understanding and verifying the submitted code and design decisions.

## Conclusion

The project demonstrates a complete local HTTP forward-proxy workflow with bounded concurrency, hostname access control, a thread-safe educational response cache, synchronized logging and metrics, deterministic integration tests, and a real direct-versus-proxy benchmark. The documented limitations define the boundary of the implementation and should be included in the project demonstration.
