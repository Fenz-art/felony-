# Computer Networks Mini Project: HTTP Forward Proxy Server

## 1. Overview

This project implements a concurrent HTTP forward proxy in C. The proxy sits between a client and the origin web server and forwards HTTP requests while adding functionality such as request validation, destination filtering, response caching, logging, metrics collection, and timeout-based reliability.

The software demonstrates how an application-layer intermediary can inspect traffic, control access, optimize repeated responses, and expose runtime behaviour for observation. The project was designed as an educational computer networks implementation, combining concepts from socket programming, HTTP, concurrency, caching, and system design.

The implemented proxy accepts HTTP requests from clients, checks whether the destination host is allowed, looks for a cached response, and forwards the request upstream when necessary. It also records operational statistics such as cache hit rate, request success/failure rate, and average response latency.

---

## 2. Project purpose

The main purpose of the project is to study and implement how a proxy server works in practice.

A proxy server acts as an intermediary between clients and origin servers. It does not simply route traffic blindly; it can:
- validate requests
- filter destinations
- cache responses
- reduce repeated upstream traffic
- inspect requests and responses
- collect usage and performance data

This project demonstrates these features in a compact, approachable C-based implementation.

---

## 3. Why this project matters

This project is useful because it combines several important ideas from computer networks and systems programming:
- TCP socket communication
- HTTP protocol parsing
- connection management
- concurrency and thread synchronization
- cache design and expiry
- access control policies
- metrics and telemetry
- timeout and failure handling
- service reliability under load

These concepts are foundational in modern networking systems, application gateways, caching layers, and HTTP middleboxes.

---

## 4. System model

The project uses a forward proxy model.

In the simplest model, a client connects directly to a web server:

Client → Origin Server

In the proxy model:

Client → Proxy Server → Origin Server → Proxy Server → Client

The proxy acts as an intermediary. It can decide whether to:
- allow the request
- deny the request
- serve a cached response
- fetch from the upstream server
- log all traffic and compute metrics

This is a common pattern in enterprise networks, content delivery, filtering, and traffic inspection.

---

## 5. Scope of the implementation

The project intentionally supports a limited but realistic subset of HTTP functionality.

Supported features:
- HTTP/1.0 and HTTP/1.1
- GET requests
- absolute-form and origin-form request targets
- host-based access control
- response caching for successful GET responses
- thread-based concurrent request handling
- timeouts for reads, writes, and origin connections
- request and response logging
- metrics output and CSV export

Not supported or intentionally restricted:
- HTTPS tunneling
- POST, PUT, DELETE, PATCH, and other methods
- request bodies
- IPv6 authority parsing
- full RFC-compliant HTTP cache semantics
- user authentication or client identity checks
- public exposure without additional network hardening

This scope is appropriate for a mini-project and keeps the implementation understandable while still demonstrating major concepts.

---

## 6. Core networking concepts used

### 6.1 TCP sockets

TCP is used for communication between:
- client and proxy
- proxy and origin server

TCP provides reliable, ordered byte-stream communication over a connection. This is ideal for client-server protocol communication such as HTTP.

Why it was used:
- HTTP works over TCP
- it provides reliable transmission and error handling
- it allows persistent connection-style behaviour and controlled timeouts

In the implementation, sockets are created, bound, listened on, accepted, connected, and closed through helper functions and low-level socket APIs.

### 6.2 HTTP request parsing

The proxy reads raw HTTP request bytes and interprets them into meaningful fields such as:
- method
- host
- path
- port
- protocol version

This is handled by the HTTP parsing layer in the helper module.

Why it was used:
- the proxy must understand what the client is asking for
- it must validate whether the request is legal
- it must reconstruct an upstream request for the origin server

Parsing is essential because the proxy is not just passing bytes through; it is reading and processing application-level data.

### 6.3 HTTP methods and request validation

The implementation supports only GET requests and rejects unsupported methods such as POST or PUT. It also rejects malformed requests, empty targets, invalid Host headers, and unsupported bodies.

Why it was used:
- to keep the service focused and reliable
- to reduce complexity in the parser and upstream forwarding logic
- to align with the project scope

This makes the implementation safer and easier to validate in tests.

### 6.4 Timeouts and socket reliability

The proxy uses time-sensitive operations to avoid hanging forever on slow clients or unresponsive origins.

Examples include:
- maximum client header wait time
- read/write deadlines
- origin connection timeout
- upstream response timeout

Why it was used:
- network services must not block indefinitely
- slow or malicious clients must not stall the server
- origin failures should be handled gracefully

Timeouts improve the reliability and robustness of the proxy.

---

## 7. Concepts used in the implementation

### 7.1 Concurrency

The proxy handles multiple clients simultaneously by creating worker threads for each accepted request.

Why it was used:
- a proxy should be able to serve more than one client at a time
- networking services are naturally concurrent
- a single-threaded accept loop would block on slow clients

This is a core concept in computer networks and server design.

Implementation details:
- worker threads are created when a client connects
- a semaphore limits the maximum number of concurrent workers
- worker counts are tracked for clean shutdown
- mutexes protect shared structures such as cache, metrics, and worker count tracking

### 7.2 Thread synchronization

Since multiple threads can access shared memory at the same time, synchronization is required.

Concepts used:
- pthread mutexes
- semaphores
- condition variables

Why they were used:
- to protect shared state such as the cache and metrics
- to avoid data races during concurrent access
- to allow controlled shutdown when active workers remain

Without synchronization, cache content, counters, and request state could become inconsistent or corrupted.

### 7.3 Semaphores for worker caps

A semaphore is used to limit the number of active workers.

Why it was used:
- the server should not allow unlimited growth in threads
- a worker cap helps prevent system overload
- if the server is under heavy load, excess clients can be rejected with HTTP 503

This is a practical example of admission control in a server.

### 7.4 Signal handling

The server listens for termination signals such as SIGINT and SIGTERM.

Why it was used:
- a server should shut down cleanly
- active worker threads must finish or be stopped safely
- resources should be cleaned up without leaving the process in a broken state

This is a standard feature in robust server design.

### 7.5 Caching

A cache stores prior responses so that repeated requests can be served faster without contacting the origin again. The proxy uses an in-memory cache keyed by destination host, port, and path.

Why it was used:
- repeated requests are common in real-world traffic
- caching reduces latency and upstream load
- it demonstrates the core idea of optimization at the network intermediary layer

The cache also stores a TTL (time-to-live), controlling how long a response remains valid.

### 7.6 TTL and expiry

Every cache entry has an expiry timestamp based on the configured TTL.

Why it was used:
- cached responses should not remain valid forever
- stale content can become incorrect or outdated
- TTL keeps the cache consistent with real-world assumptions

On lookup, expired entries are removed before being served.

### 7.7 Cache eviction

When the cache reaches its capacity, old entries are removed.

Why it was used:
- memory is limited
- a proxy cannot keep growing without bound
- eviction demonstrates how cache policies manage finite memory

The implementation uses a simple oldest-insertion eviction pattern, which is straightforward and suitable for a mini-project.

### 7.8 Access control

The proxy uses a hostname allow/block policy to restrict access.

Why it was used:
- a proxy often acts as a gatekeeper or policy layer
- it can prevent access to certain domains or destinations
- it demonstrates control over upstream traffic

The policy is loaded from a file and checked before the request is forwarded upstream.

### 7.9 Logging

The proxy logs information about every request.

Why it was used:
- network services need observability
- logs support troubleshooting and debugging
- logs also help analyze traffic patterns and errors

The log includes request metadata such as client IP, method, destination host, status code, cache result, and latency.

### 7.10 Metrics collection

The metrics module records request counts and response statistics.

Why it was used:
- to evaluate proxy performance
- to distinguish successful and failed requests
- to measure cache effectiveness
- to estimate latency and throughput

The project tracks:
- total requests
- successful requests
- failed requests
- cache hits
- cache misses
- total bytes transferred
- overall latency

This makes the system measurable and analyzable.

### 7.11 Data capture and raw response storage

The proxy captures the raw HTTP response when it is cacheable so that it can later be served from memory.

Why it was used:
- the cache works at the HTTP response level
- the proxy can replay the original response bytes accurately
- this is simpler and closer to real proxy behavior than re-encoding or modifying the response body

---

## 8. Detailed explanation of important features used

### 8.1 HTTP request parsing feature

The parser identifies:
- request method
- URL target
- protocol version
- host and port
- required Host header for HTTP/1.1

Why it is important:
- without parsing, the proxy cannot decide what to do with the request
- it is the key distinction between a raw socket listener and an HTTP-aware intermediary

### 8.2 URL normalization and cache key creation

The cache key is derived from the destination host, port, and path. This allows the proxy to identify repeated requests for the same resource.

Why used:
- repeated identical URL requests can be served without re-fetching from the origin server
- this reduces upstream traffic and response delay

### 8.3 Header extraction and boundary detection

The proxy reads until it detects the end of HTTP headers, which is the boundary between request headers and the message body.

Why used:
- request parsing requires header termination detection
- the proxy rejects unsupported bodies and malformed requests

### 8.4 Response parsing

The response parser reads the status line and checks headers such as Content-Length and Transfer-Encoding.

Why it is important:
- response parsing determines if a response can be cached
- it validates whether the response is well-formed
- it decides whether the body should be captured and stored

### 8.5 Body capture and caching rules

The proxy only caches successful responses with valid Content-Length and no chunked transfer encoding.

Why used:
- this keeps the cache logic simpler and reliable
- chunked and bodyless responses are harder to reconstruct consistently
- it avoids unpredictable cache entries for unsupported cases

---

## 9. Member-wise contribution and learning responsibilities

The project was divided according to theme and responsibility. Each member was expected not only to contribute to that area but also to understand the whole system, because all parts are integrated into a single proxy service.

### 9.1 Kumar Kushang — Architecture, concurrency, and caching

Main responsibilities:
- overall system design
- concurrency model
- worker management
- cache architecture
- proxy execution flow

Concepts to study:
- multithreading and synchronization
- shared state protection
- cache TTL and eviction logic
- worker limits and server architecture
- request lifecycle of a proxy

Why this role matters:
- a proxy is fundamentally a concurrent server, and its correctness depends on safe concurrency and data flow
- the cache is central to performance improvement

### 9.2 Tonisha Ghosh — TCP and HTTP Layer

Main responsibilities:
- TCP connection handling
- HTTP request parsing
- origin connection creation
- client/origin socket operations

Concepts to study:
- sockets and transport-layer basics
- address resolution and connection establishment
- HTTP message structure
- request validation and error detection

Why this role matters:
- without clean socket management and parsing, the proxy cannot correctly communicate with clients or upstream servers

### 9.3 Ahona Misra — Access control, configuration, and error handling

Main responsibilities:
- access control rules
- configuration file loading
- default allow/block logic
- rejection and error handling paths

Concepts to study:
- policy design
- validation and filtering
- handling malformed requests and unsupported traffic
- security-minded filtering logic

Why this role matters:
- a proxy is often used to restrict traffic, and the project demonstrates why policy enforcement is essential

### 9.4 Aastha Ray — Logging, metrics, and performance evaluation

Main responsibilities:
- request logging
- metrics collection
- output analysis
- performance benchmarking
- result interpretation

Concepts to study:
- instrumentation and observability
- logging design
- throughput and latency measurement
- benchmarking under concurrent load

Why this role matters:
- network services are only useful if they can be understood and measured
- logs and metrics provide evidence of correctness and performance

---

## 10. Shared and integrated parts of the project

The shared and integration-based elements are what bind every individual contribution into a single working service.

### 10.1 Shared files

The project uses several shared headers and interfaces to connect modules:
- [proxy_server.h](proxy_server.h)
- [http_tcp_handler.h](http_tcp_handler.h)
- [cache.h](cache.h)
- [access_control.h](access_control.h)
- [metrics.h](metrics.h)
- [logger.h](logger.h)

These files define the contracts used across modules. They enable:
- configuration sharing
- module integration
- dependency control
- consistent interfaces

### 10.2 Integration flow

The system is assembled in [main.c](main.c). This file creates and initializes components and passes them into the proxy runtime.

The integration flow is:
- create cache
- create access control object
- load policy file if provided
- initialize metrics and logger
- build proxy configuration
- launch proxy server
- worker threads use all integrated modules while the server runs

This is the essential shared part of the architecture: all separate contributions are connected through one startup flow and one runtime configuration object.

### 10.3 Why integration matters

A proxy is not a set of isolated features; it is one system. To work correctly, all components must agree on:
- how requests are parsed
- how hosts are filtered
- how cache keys are built
- how errors are represented
- how logs and metrics are recorded

A single bug in one layer can affect the whole system. That is why shared interfaces and integration testing are critical.

---

## 11. Testing documentation

The project includes several testing layers to validate correctness and behavior.

### 11.1 Unit tests

The unit tests validate the individual logic of each major subsystem.

#### Access-control tests
File: [tests/unit/test_access_control.c](tests/unit/test_access_control.c)

These tests verify:
- allowed hosts are accepted
- blocked hosts are rejected
- default allow/block logic works correctly
- rules are normalized correctly
- invalid rules are rejected properly

#### Cache tests
File: [tests/unit/test_cache.c](tests/unit/test_cache.c)

These tests verify:
- cache insert works correctly
- cache lookup works correctly
- expired entries are cleaned up
- the cache evicts when full
- repeated data is handled consistently
- the cache behaves correctly under concurrent access

Why these tests matter:
- they validate the most important data structures and logic in isolation
- they catch failures before the full system is tested

### 11.2 Integration tests

File: [tests/integration/test_proxy.c](tests/integration/test_proxy.c)

This tests the real end-to-end proxy flow by running the proxy and a local origin server together.

It checks:
- cache hits and misses
- request forwarding correctness
- origin-server communication
- blocked destination handling
- metrics behavior under realistic use

Why it matters:
- source-level correctness is not enough; the full service path must be validated
- integration tests check that all modules work together correctly

### 11.3 Local origin server fixture

File: [tests/fixtures/local_origin_server.c](tests/fixtures/local_origin_server.c)

This is a lightweight local origin server used to simulate a real upstream server.

Why it matters:
- it allows the proxy to be tested in a controlled environment
- it avoids relying on the public internet during tests
- it ensures stable, repeatable validation

### 11.4 Regression tests

Files:
- [regression_tests/contract_tests.c](regression_tests/contract_tests.c)
- [regression_tests/timeout_test.c](regression_tests/timeout_test.c)
- [regression_tests/http_fixture.c](regression_tests/http_fixture.c)
- [regression_tests/run.sh](regression_tests/run.sh)

These tests cover:
- protocol contract correctness
- timeout behavior
- malformed and edge-case requests
- repeated testing of known bug patterns

Why they matter:
- regular unit tests often miss edge conditions
- regressions are common in networking code and must be checked repeatedly

### 11.5 Performance and confidence tests

File: [performance_test.c](performance_test.c)

This file evaluates:
- request throughput
- latency
- stress behavior
- comparison between direct origin access and proxied access
- effect of cache usage on subsequent requests

Why it is included:
- a proxy project is not only about correctness
- the system should also be evaluated for its performance and robustness under concurrent requests

---

## 12. Build process

The project is built using the [Makefile](Makefile).

Typical build commands:
- `make` to compile the project
- `make test` to run the test suite
- `make clean` to remove build artifacts

The project produces executables in the [build](build) directory.

This keeps the build process simple and reproducible.

---

## 13. Running the project

The proxy binary can be started with options such as:
- port
- cache size
- maximum cached response size
- cache TTL
- access-control file

Typical usage:
- `build/proxy --port 8080`
- `build/proxy --help`

This allows the developer to tune the proxy for different use cases while keeping the runtime model consistent.

---

## 14. Metrics and output

The project prints metrics at shutdown and saves them to [metrics.csv](metrics.csv).

Metrics include:
- total requests handled
- success and failure counts
- cache hit and miss totals
- total request bytes
- average latency

This is useful both for debugging and for performance evaluation.

---

## 15. Why these features were chosen

The selected features are not arbitrary. They were chosen because they represent the minimum set needed to build a meaningful and demonstrative proxy server.

Features used and why:
- sockets → because HTTP runs over TCP
- concurrent threads → because real proxies must handle multiple clients
- access control → because proxies often act as security or filtering layers
- cache → because reducing repeated network traffic is one of the main reasons to use a proxy
- logging → because proxy behaviour must be observable and debuggable
- metrics → because performance and reliability must be measured
- timeouts → because network systems must handle failure gracefully
- tests → because correctness is essential and must be repeated under different conditions

Each feature contributes directly to the purpose of the project.

---

## 16. Final architectural interpretation

This project is best understood as a small but realistic model of a real-world HTTP proxy server built in C. It combines fundamental networking ideas with application-level logic and demonstrates the core tasks a proxy performs in the real world.

The proxy is designed not merely to pass data, but to:
- inspect requests
- enforce access policy
- reuse prior results
- measure performance
- recover from failures gracefully
- maintain operational logs

This is why it is a strong computer networks project: it shows how a simple network intermediary can become a resourceful service with control, optimization, and visibility.

---

## 17. Conclusion

The project demonstrates the practical implementation of a forward proxy using standard C networking libraries and POSIX threading primitives. It successfully combines several major concepts:
- TCP networking
- HTTP parsing
- concurrency
- access control
- caching
- timeout management
- logging
- metrics collection
- testing and benchmarking

By dividing responsibilities among team members and integrating all modules at the end, the system becomes a coherent service. This is exactly what makes it a valuable project in systems and networks: it is not only a set of isolated code files, but a working software system in which each part contributes to the complete behavior of the proxy.

---

## 18. Summary of responsibilities

- Kumar Kushang: architecture, concurrency, caching
- Tonisha Ghosh: TCP and HTTP handling
- Ahona Misra: access control, configuration, error handling
- Aastha Ray: logging, metrics, benchmarking

These responsibilities are individual specializations, but the whole team must understand the complete system because all modules are integrated in the final proxy.

The integration and shared parts are described in a final stage of the design because they tie together the entire project. The shared components include the common headers, the runtime configuration, and the startup assembly in [main.c](main.c). This is where individual tasks become a single working proxy service.
