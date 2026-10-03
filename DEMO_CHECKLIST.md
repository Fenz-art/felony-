# Demo Checklist

This checklist reflects the actual project commands and the validated local proxy workflow.

## 1. Start the local origin

Build the local fixture if needed, then start it in one terminal:

```bash
make
./build/local_origin_server
```

The origin prints the port it bound to on stdout. Record that port as `<origin_port>`.

## 2. Start the proxy

In a second terminal, run:

```bash
./build/proxy --port 8888 --cache-size 32 --cache-max-response 1048576 --cache-ttl 30 --access-control config/access_control.conf
```

This is the same configuration used in the project’s local regression checks.

## 3. Normal HTTP GET

Use the proxy to fetch a normal origin response:

```bash
curl --proxy http://127.0.0.1:8888 -i http://127.0.0.1:<origin_port>/hello
```

The response should be HTTP 200 and should include the body from the local origin.

## 4. Demonstrate CACHE MISS

The first proxied request to a fresh key is a cache miss. The response should reflect origin delivery, and the request count will increment on the origin.

```bash
curl --proxy http://127.0.0.1:8888 -sS http://127.0.0.1:<origin_port>/hello
curl --proxy http://127.0.0.1:8888 -sS http://127.0.0.1:<origin_port>/_count?path=/hello
```

The origin counter should increase for the first access.

## 5. Repeat the same request to demonstrate CACHE HIT

```bash
curl --proxy http://127.0.0.1:8888 -sS http://127.0.0.1:<origin_port>/hello
curl --proxy http://127.0.0.1:8888 -sS http://127.0.0.1:<origin_port>/_count?path=/hello
```

The second request should be served from cache; the origin counter should remain unchanged.

## 6. Demonstrate blocked domain

The sample ACL blocks `blocked.test`.

```bash
curl --proxy http://127.0.0.1:8888 -i http://blocked.test/
```

Expected result: HTTP 403.

Check that the origin count is unchanged:

```bash
curl --proxy http://127.0.0.1:8888 -sS http://127.0.0.1:<origin_port>/_count?path=/hello
```

## 7. Demonstrate malformed request -> 400

Use a Python socket to send an invalid request that never completes a valid header block:

```bash
python3 - <<'PY'
import socket
s = socket.create_connection(('127.0.0.1', 8888), timeout=5)
s.sendall(b'GET / HTTP/1.1\r\nHost:\r\n\r\n')
print(s.recv(4096).decode('latin1', 'replace'))
s.close()
PY
```

Expected result: HTTP 400.

## 8. Demonstrate unreachable origin -> 502

Use a closed local port, such as a port not bound by the local origin fixture:

```bash
curl --proxy http://127.0.0.1:8888 -i http://127.0.0.1:1/hello
```

Expected result: HTTP 502.

## 9. Demonstrate timeout -> appropriate failure

The local origin includes a `/delay` route that stalls for 9 seconds, exceeding the proxy’s 8-second origin-response deadline.

```bash
curl --proxy http://127.0.0.1:8888 -i http://127.0.0.1:<origin_port>/delay
```

Expected result: HTTP 504.

## 10. Demonstrate concurrent clients

Run several concurrent requests in parallel:

```bash
for i in $(seq 1 25); do
  curl --silent --show-error --proxy http://127.0.0.1:8888 http://127.0.0.1:<origin_port>/hello >/dev/null &
done
wait
```

The local validation run handled this burst successfully, with `CONCURRENCY_COUNT=250` and `ERR_COUNT=0` when scaled in the custom stress check.

## 11. Demonstrate logs

After some traffic, check the proxy log:

```bash
tail -n 20 proxy.log
```

The log records timestamp, client, method, host, path, status, cache state, size, and latency.

## 12. Show metrics

When the proxy exits cleanly, it prints the metrics summary. Stop the proxy with Ctrl+C in its terminal, and observe the metrics output.

## 13. Run the direct-vs-proxy benchmark

Use the real benchmark binary from the project root:

```bash
./build/performance_test direct 127.0.0.1 <origin_port> / 5
./build/performance_test proxy 127.0.0.1 8888 127.0.0.1 <origin_port> / 5
```

This compares the same local origin and request path with proxy caching disabled.

## 14. Final repo-level regression command

```bash
make clean && make && make test
```

The validated result was:

```text
Failures: 0
Integration failures: 0
```
