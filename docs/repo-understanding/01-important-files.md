# Important Files

## Entrypoints

| File / directory | Role | Why it matters |
|---|---|---|
| `src/outbackcdx/Main.java` | CLI entry point | Parses all command-line arguments and wires together the server, data store, authorizer, and replication threads. Starting here gives the full picture of startup. |
| `src/outbackcdx/Webapp.java` | HTTP request router and handler | Defines every HTTP route and delegates to the correct handler method. The single most important file for understanding what the server does. |

## Core Application Logic

| File / directory | Role | Why it matters |
|---|---|---|
| `src/outbackcdx/DataStore.java` | Collection manager | Maintains a map of open RocksDB instances (one per named collection). Controls how databases are opened, configured, and listed. |
| `src/outbackcdx/Index.java` | Per-collection query interface | Wraps a single RocksDB database; exposes typed query methods (exact, prefix, domain, range, closest). Also manages batch writes, aliases, replication log access, and background compaction. |
| `src/outbackcdx/Capture.java` | CDX record model + binary codec | Defines the fields of a CDX record and the binary encoding scheme (SURT key + VarInt-packed or CBOR value) used by RocksDB. Central to understanding storage format. |
| `src/outbackcdx/Query.java` | Query parameter parsing and execution | Parses HTTP query parameters, resolves wildcards and match types, builds the SURT key, and dispatches to the right `Index` method. |
| `src/outbackcdx/WbCdxApi.java` | CDX/JSON API (PyWb protocol) | Implements the CDX query response in text, JSON array, JSON dict, and CDXJ formats. |
| `src/outbackcdx/XmlQuery.java` | OpenWayback XML API | Implements the Wayback `RemoteCollection` XML protocol (`urlquery` and `prefixquery`). |
| `src/outbackcdx/UrlCanonicalizer.java` | URL canonicalization | Converts raw URLs to SURT form. Also implements fuzzy matching via YAML-configurable rules. |
| `src/outbackcdx/ChangePollingThread.java` | Replication client (secondary mode) | Background thread that polls an upstream primary's `/changes` endpoint and applies write batches locally. |
| `src/outbackcdx/FeatureFlags.java` | Runtime feature toggles | Centralises all feature-flag checks; flags are set by environment variables or CLI arguments at startup. |

## Configuration

| File / directory | Role | Why it matters |
|---|---|---|
| `pom.xml` | Maven build descriptor | Defines dependencies (RocksDB, jwarc, urlcanon, Jackson, Nimbus JWT, etc.), Java version (11), and the shade plugin that builds the fat JAR. |

## Access Control

| File / directory | Role | Why it matters |
|---|---|---|
| `src/outbackcdx/AccessControl.java` | Access control engine | Stores rules and policies in RocksDB; evaluates them at query time using an in-memory inverted radix tree. Only active when `EXPERIMENTAL_ACCESS_CONTROL=1`. |
| `src/outbackcdx/auth/` | Authorizer implementations | Contains `JwtAuthorizer`, `KeycloakConfig`, `NullAuthorizer`, `Permission`, and `Permit`. Determines whether a request is allowed to write to the index. |
| `docs/access-control.md` | Access control model documentation | Explains the access point / policy / rule model. Useful before touching access control code. |

## UI / Static Assets

| File / directory | Role | Why it matters |
|---|---|---|
| `resources/outbackcdx/` | Web UI and API assets | Served directly from the JAR. Includes `dashboard.html` (Vue.js SPA), `api.html` (Swagger/ReDoc API docs), `replay.js` (service worker for replay), and SVG icons. |

## Tests and Fixtures

| File / directory | Role | Why it matters |
|---|---|---|
| `test/outbackcdx/WebappTest.java` | Integration-style unit tests | Tests HTTP round-trips against a real RocksDB (temp folder). Covers CDX ingestion, querying, XML protocol, access control, and aliases. |
| `test/outbackcdx/IndexTest.java` | Low-level index tests | Tests query types (closest, prefix, alias resolution) directly against an in-memory RocksDB. |
| `test/outbackcdx/AccessControlTest.java` | Access rule/policy tests | Verifies that rules are stored, loaded, and correctly filter query results. |
| `test-integration/` | Shell-based integration tests | Launches a real server, loads CDX records, and checks end-to-end responses via `curl`. Includes PyWb and OpenWayback integration scenarios. |
| `test-integration/test1.cdx`, `test2.cdx` | Sample CDX fixture data | Real CDX records used by the integration tests. |
| `test-integration/test1.warc.gz`, `test2.warc.gz` | Sample WARC archives | Used by replay integration tests. |
