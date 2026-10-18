# t4 Refined Specification

## 1. Purpose

Build `t4`, a self-contained C17 HTTP server. Store objects in one flat namespace. Use URL-encoded path keys. Do not implement buckets, authentication, multipart uploads, or versioning.

## 2. Required Repository Layout

Create these files and directories:

```text
t4/
├── Makefile
├── README.md
├── .gitignore
├── Dockerfile
├── docker-compose.yml
├── src/
│   ├── t4.c
│   ├── http.h
│   ├── http.c
│   ├── store.h
│   ├── store.c
│   ├── etag.h
│   ├── etag.c
│   ├── route.h
│   └── route.c
├── tests/unit/
│   ├── test.h
│   ├── test_http.c
│   ├── test_store.c
│   ├── test_etag.c
│   └── test_route.c
├── tests/integration/blackbox.sh
└── tests/e2e/
    ├── Dockerfile
    └── run_tests.sh
```

Keep every source file below 500 lines. Do not use third-party C libraries. Use C17 and POSIX sockets and file APIs.

## 3. Build and CLI Contract

Run `make build` to create `bin/t4` with exactly:

```text
gcc -Wall -Wextra -Werror -pedantic -std=c17 -O2 -Isrc -o bin/t4 src/*.c
```

Invoke the server as:

```text
./bin/t4 [--data-dir DIR] [--host HOST] [--port PORT]
```

Use these defaults:

```text
--data-dir ./data
--host 0.0.0.0
--port 8080
```

Parse `--port` as an integer from 1 through 65535. For every invalid port, write exactly `t4: invalid port` to stderr and exit 1. Create the data directory recursively. For directory or bind failure, write `t4: <reason>` to stderr and exit 1. Write normal logs to stdout. Prefix every server error with `t4: `. Handle SIGINT and SIGTERM, close resources, and exit 0.

## 4. HTTP Response Contract

Use HTTP/1.0 or HTTP/1.1. Treat methods and paths as case-sensitive. Require uppercase methods. Include these headers in every response:

```text
Server: t4
Content-Length: <decimal byte count>
Connection: close
```

Send one request per connection, then close the connection. End every error and listing body with a newline. Encode error and listing bodies as JSON. Escape JSON string characters correctly.

Return only these error reasons:

```json
{"error":"no such key"}
{"error":"invalid key"}
{"error":"invalid range"}
{"error":"range not satisfiable"}
{"error":"body too large"}
{"error":"bad request"}
{"error":"method not allowed"}
```

Use status 400 for invalid key, invalid range, and bad request. Use status 404 for no such key. Use status 405 for method not allowed. Use status 413 for body too large. Use status 416 for range not satisfiable.

## 5. Endpoint Matrix

### 5.1 PUT `/<key>`

Decode the path key before validation. Read exactly `Content-Length` bytes. Treat a missing `Content-Length` as a zero-byte body. Accept binary bytes. Reject a body larger than 1048576 bytes with status 413, the `body too large` JSON error, and connection close. Store the object with an atomic temporary-file, fsync, and rename sequence.

Return status 201 for a new key and status 200 for an existing key. Return the following header:

```text
ETag: "<decimal-size>-<16 lowercase hexadecimal FNV-1a-64 hash>"
```

Return an empty body unless an error occurs.

### 5.2 GET `/<key>`

Return status 200 and the exact stored bytes. Set `Content-Type: application/octet-stream`. Set `Content-Length` to the selected body length. Return the deterministic ETag.

Support one `Range` header with these forms:

```text
Range: bytes=<start>-<end>
Range: bytes=<start>-
```

Use zero-based inclusive indexes. Return status 206 and this header for a valid range:

```text
Content-Range: bytes <start>-<end>/<total>
```

Return status 400 with `invalid range` for malformed syntax, non-numeric values, multiple ranges, or `start > end`. Return status 416 with `Content-Range: bytes */<total>` and `range not satisfiable` when `start >= total`. Clamp an end value greater than the last byte to the last byte. Return 404 for a missing key.

### 5.3 GET `/`

List all stored keys. Return status 200, `Content-Type: application/json`, and this body shape:

```json
{"keys":["<key1>","<key2>"]}
```

Sort decoded keys by byte-wise ascending order. Escape each key as a JSON string. End the body with a newline. Return an empty list as `{"keys":[]}\n`.

### 5.4 HEAD `/<key>`

Return the same status and headers as GET for an existing key, including `Content-Type`, `Content-Length`, and ETag. Send no body. Return 404 for a missing key. Apply the same range validation and response headers as GET, but send no body.

### 5.5 DELETE `/<key>`

Unlink an existing object. Return status 204 with an empty body and `Content-Length: 0`. Return 404 with `no such key` for a missing key.

### 5.6 Unsupported Methods

Return status 405 with `{"error":"method not allowed"}\n` for an unsupported method on a valid route. Do not implement buckets, nested paths, or additional endpoint behavior.

## 6. Key Rules

Define the key as the path after the leading slash, after percent decoding each `%XX` byte. Treat `+` as a literal plus byte. Reject an empty key. Reject `/` and NUL after decoding. Reject `.`, `..`, and any decoded segment containing `/`. Reject malformed percent encoding with `invalid key`. Store data under `<data-dir>/<key>`. Ensure keys cannot address internal names or escape the data directory.

## 7. ETag Algorithm

Hash raw object bytes with FNV-1a 64. Use offset basis `14695981039346656037` and prime `1099511628211`. Format the ETag as quoted decimal size, a hyphen, and 16 lowercase hexadecimal hash digits. The empty body ETag must equal:

```text
"0-cbf29ce484222325"
```

## 8. Module Responsibilities

Keep `t4.c` responsible for CLI parsing, startup, signals, socket creation, accept handling, and composition. Keep `http.c` responsible for HTTP parsing and response writing. Keep `store.c` responsible for file-backed PUT, GET, DELETE, and LIST operations. Keep `etag.c` responsible for ETag computation. Keep `route.c` responsible for method and key dispatch. Declare public interfaces in the matching headers. Pass store state through parameters. Do not use global mutable state.

## 9. Required Test Gates

Run `make test` to build and run all unit and integration tests. Fail if any test fails or if zero tests run. Test URL decoding, request parsing, range parsing, real temporary-directory storage, atomic overwrite behavior, ETag determinism, route validation, status codes, headers, binary bodies, and all error bodies.

Run `make lint` with exactly these commands:

```text
clang-tidy -checks='clang-diagnostic-*,clang-analyzer-*' -Isrc src/*.c
gcc -Wall -Wextra -Werror -pedantic -std=c17 -fsyntax-only -Isrc src/*.c
```

Require zero findings and zero warnings.

Run `make format` with:

```text
clang-format -i src/*.c src/*.h
```

Make formatting idempotent. Run `make clean` to remove `bin/`, object files, and `data/`.

Run `make e2e` with:

```text
docker compose up --build --exit-code-from e2e
```

Use the real server binary and a separate test runner. Poll for socket readiness. Set a maximum two-second timeout for every HTTP operation. Test each PUT, GET, HEAD, DELETE, listing, range, invalid key, invalid range, missing key, body limit, ETag, and CLI behavior through the public interface.

## 10. Container Contract

Define services `t4` and `e2e` in `docker-compose.yml`. Build the server from the repository. Build the test runner from `tests/e2e`. Set `T4_URL` to `http://t4:8080`. Use the `t4-data` named volume. Use Alpine, curl, and bash in the E2E image.

## 11. Documentation

Document build, run flags, endpoint behavior, status codes, headers, JSON shapes, key rules, ETag format, `make test`, `make lint`, `make format`, `make e2e`, and `make clean` in README.md.

## 12. Forbidden Practices

Do not use third-party C libraries, buckets, SigV4, multipart uploads, versioning, threads, global mutable state, shell failure masking, placeholders, tautological tests, or unasserted test results. Use real parsers, real temporary files, real byte buffers, and real sockets at integration and E2E boundaries.