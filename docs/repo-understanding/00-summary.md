# OutbackCDX — Repository Summary

## Purpose

OutbackCDX is a **CDX index server** for web archives.

It stores metadata records about archived web pages (URLs, timestamps, WARC file locations, checksums) and serves them to web archive replay tools.
Clients such as OpenWayback and PyWb query it to find out *which* WARC file and *where* in that file a captured copy of a URL lives, so they can serve it to users.

It is used in production at the National Library of Australia and the British Library, with indexes of 8–9 billion records.

## Technology Stack

| Layer | Technology |
|---|---|
| Language | Java 11+ |
| Build | Maven (`mvn package`) |
| Storage | RocksDB (embedded, via `rocksdbjni`) |
| HTTP server | JDK built-in `HttpServer` (default) or Undertow (optional) |
| JSON | Jackson |
| WARC handling | jwarc |
| URL canonicalization | urlcanon + custom rules |
| JWT auth | Nimbus JOSE+JWT |
| Replication HTTP client | Apache HttpComponents |
| Fuzzy match config | SnakeYAML |
| Frontend | Vue 2 (served as static assets from the JAR) |

## Main Runtime Model

OutbackCDX runs as a **single Java process**.

- RocksDB is embedded — no separate database server.
- Each named collection is a separate RocksDB database in a subdirectory of the data directory.
- Collections are created automatically on first write.
- HTTP is handled by a fixed thread pool (default: one thread per CPU core).
- In **secondary mode**, a background thread polls an upstream primary for changes and applies them locally.

## Main External Services or Dependencies

- **WARC files** — read directly via HTTP range requests or local filesystem when replay is enabled.
- **Keycloak** or a **JWKS endpoint** — for authorization (optional; unauthenticated by default).
- **OpenWayback / PyWb / Heritrix** — the clients that query OutbackCDX.

## Main Entry Points

| Entry point | Purpose |
|---|---|
| `src/outbackcdx/Main.java` | CLI entry point; parses arguments and starts the HTTP server |
| `POST /<collection>` | Ingest CDX records into a named collection |
| `GET /<collection>?url=…` | Query the index (CDX/JSON/XML output) |
| `GET /<collection>?q=type:urlquery+url:…` | OpenWayback XML protocol query |
| `GET /<collection>/changes` | Replication change-feed endpoint (primary → secondary) |

## Architecture Summary

The HTTP layer (`Web.java`, `Webapp.java`) routes requests to handler methods.

All persistent state lives in RocksDB through `DataStore` (collection manager) and `Index` (per-collection query interface).
`Capture.java` defines how CDX records are encoded into RocksDB keys and values — keys are the canonicalised URL (SURT) + timestamp, values are VarInt-packed fields (or CBOR for index version 5).

Two query protocols are supported:

- **WbCdxApi** — CDX/JSON output, used by PyWb and compatible clients.
- **XmlQuery** — OpenWayback XML output, used by OpenWayback's `RemoteResourceIndex`.

URL canonicalization converts raw URLs to SURT form before storage and before queries.
Fuzzy matching (optional YAML config) strips session IDs and other noise, enabling playback of non-identical but equivalent URLs.

Optional features include access control (rules + policies stored in separate RocksDB column families), HMAC-signed field generation for WARC access authentication, and WARC replay (serving the actual archived content directly).

## Read These First

| File / directory | What it explains |
|---|---|
| `src/outbackcdx/Main.java` | Startup, all CLI options, server wiring |
| `src/outbackcdx/Webapp.java` | All HTTP routes and request handling |
| `src/outbackcdx/DataStore.java` | Collection management; how RocksDB databases are opened |
| `src/outbackcdx/Index.java` | Query interface; how records are read and written |
| `src/outbackcdx/Capture.java` | CDX record model and binary encoding |
| `src/outbackcdx/WbCdxApi.java` | CDX/JSON query output (PyWb protocol) |
| `src/outbackcdx/XmlQuery.java` | OpenWayback XML query output |
| `src/outbackcdx/UrlCanonicalizer.java` | URL → SURT conversion and fuzzy matching |
| `src/outbackcdx/ChangePollingThread.java` | Replication (secondary → primary polling) |
| `src/outbackcdx/FeatureFlags.java` | Feature toggle mechanism |
