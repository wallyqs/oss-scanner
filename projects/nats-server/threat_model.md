# NATS Server threat model

NATS Server (`nats-server`) is a high-performance messaging server written in Go.
It provides core publish/subscribe and request/reply, multi-tenant accounts,
clustering (routes), superclusters (gateways), leaf nodes, WebSocket and MQTT
front ends, and JetStream persistence (streams, consumers, key-value and object
stores) replicated with its own Raft implementation.

## Where untrusted input enters

- Client connections on the client port (default 4222): the text protocol parser
  (`server/parser.go`, `server/client.go`), `CONNECT` options, subjects, headers,
  and payloads. Treat every byte from an unauthenticated or authenticated client
  as untrusted.
  
- WebSocket (`server/websocket.go`) and MQTT (`server/mqtt.go`) listeners,
  including frame/packet decoding, compression, and session state.
  
- Authentication material: user/password, tokens, NKEY signatures, JWTs
  (operator/account/user, `github.com/nats-io/jwt`), TLS client certificates and
  OCSP, and auth callout responses.
  
- JetStream API requests (`$JS.API.>`) sent by clients: stream/consumer
  configuration, direct get, batching, purge, backups/restores and snapshots.
  
- The HTTP monitoring endpoint (`server/monitor.go`) when exposed.

- Leaf node connections from remote servers, which may belong to a different
  operator or be controlled by a less-trusted party.

Route and gateway peers inside one cluster/supercluster are operated by the same
administrator and are considered trusted, but a single compromised or malformed
peer must not be able to corrupt other servers' memory. Configuration files,
the operator, and the system account are trusted.

## Security properties that matter most

- Account isolation: a client must not publish to or receive messages from
  subjects in another account except through configured exports/imports, and
  must not escape its publish/subscribe permissions (including via wildcards,
  queue subscriptions, reply subjects, service imports, subject mappings, or
  JetStream APIs).

- Authentication must not be bypassable; expired, revoked, or wrongly-signed
  JWTs and NKEYs must be rejected; account and user limits must be enforced.

- JetStream must enforce per-account resource limits and API permissions, and
  must not let one account read or modify another account's streams, consumers,
  or stored data. Filestore and Raft state must not be corruptible by client
  input.

- Leaf nodes must stay within the permissions and account binding configured on
  the hub side.

- The server must not crash, deadlock, or exhaust memory/CPU from input an
  unprivileged client can send (a single remote client taking down the server is
  a real issue for a shared broker).

## Components

In scope: everything under `server/`, `conf/` (config parser), `logger/`, and
`internal/`. `server/stree`, `server/gsl`, `server/avl`, `server/pse`, and
`server/sublist.go` are core data structures on the hot path.

Out of scope: `test/` and `*_test.go` helpers, `scripts/`, `docker/`,
Windows-service-specific code unless reachable remotely, and third-party
dependencies unless a vulnerability is reachable through nats-server.

## How to exercise it

The checkout is at `/src` and the server binary is `/usr/local/bin/nats-server`.
The image keeps Go, module dependencies and build caches; `GOPROXY=off` and
`GOTOOLCHAIN=local` prevent downloads during the audit. Test binaries for all
packages (and a `-race` build of `./server`) are precompiled.

Prefer Go tests in `server/` as reproducers. The test helpers
`RunServer`/`RunBasicJetStreamServer`/`createJetStreamClusterExplicit` start
in-process servers, and `github.com/nats-io/nats.go` is available as a client.
A raw TCP connection to the client port is often the simplest reproducer for
protocol bugs.

The full suite is large and split by build tags (see
`scripts/runTestsOnTravis.sh`); run focused tests with `-run`, for example:

```sh
cd /src
go test -count=1 -run '^TestJetStreamConsumer' ./server
go test -count=1 -race -run '^TestNoRace' ./server
```

Image setup runs only a small baseline and logs failures without aborting the
build. Some cluster tests are timing-sensitive; a single flaky failure is not
evidence of a vulnerability.

## How we rate severity

- Critical: authentication bypass, cross-account data access, or remote code
  execution reachable by an unauthenticated client.

- High: an authenticated client escaping its permissions or account; memory
  corruption; a crash or unbounded resource use reachable by an unauthenticated
  client; persisted JetStream data corruption triggerable by a client.

- Medium: denial of service requiring an authenticated client within its
  configured permissions/limits; issues requiring a malicious leaf node.

- Low: issues requiring a trusted peer (route/gateway), the system account, or
  operator-controlled configuration; information leaks in monitoring endpoints
  that are meant to be protected by network access controls.

Our published advisory policy at https://advisories.nats.io/advisory-policy
describes further cases we do or do not consider security issues.

## Reports

Include the scanned commit, required configuration, attacker capabilities, a
deterministic reproducer (preferably a Go test), and a minimal proposed patch
with a regression test. Deduplicate findings sharing a root cause.

## Anything to leave alone

- Resource exhaustion that only occurs when the operator configured no limits
  (e.g. unlimited `max_payload`, `max_pending`, or JetStream account limits)
  unless the documented defaults are themselves unsafe.

- Behaviour of the monitoring endpoint being unauthenticated by design.

- Default behavior when there are no permissions set which allows all communication.

- Data races or panics confined to test code.
