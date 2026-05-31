# Tests and Fixtures

## Test Framework

- **JUnit 4** (`junit:junit:4.13.2`) — all unit tests use JUnit 4 annotations and assertions.
- **JMH** (`jmh-core`, `jmh-generator-annprocess`) — used for the `CaptureBenchmark` microbenchmark (not a regular test).

## Running Tests

```
mvn test
```

Unit tests run in-process. No server needs to be started.

Integration tests are shell scripts in `test-integration/` and must be run manually (they require PyWb or OpenWayback to be installed separately).

## Unit Test Files

| File | What it tests |
|---|---|
| `test/outbackcdx/WebappTest.java` | HTTP-level round trips: CDX ingestion, querying (CDX/JSON/XML), field filtering, collapse, access points, aliases, fuzzy matching, POST data queries, replication sequence endpoints |
| `test/outbackcdx/IndexTest.java` | Direct index operations: closest-date query ordering, POST data URL key encoding |
| `test/outbackcdx/AccessControlTest.java` | Access control: rule storage, policy lookup, URL pattern matching, date-range filtering, access point evaluation |
| `test/outbackcdx/AccessRuleXmlTest.java` | XML serialisation/deserialisation of access rules |
| `test/outbackcdx/AccessRuleEqualityTest.java` | `equals()` and `hashCode()` correctness for `AccessRule` |
| `test/outbackcdx/CaptureTest.java` | `Capture` binary encoding round-trips (VarInt encode → decode) |
| `test/outbackcdx/FilterTest.java` | Filter predicate construction and collapse logic |
| `test/outbackcdx/HmacFieldTest.java` | HMAC field template variable substitution and signature computation |
| `test/outbackcdx/JsonTest.java` | Jackson JSON serialisation of captures and related objects |
| `test/outbackcdx/ReplicationFeaturesTest.java` | Secondary-mode write rejection (403), sequence endpoint, WAL truncation |
| `test/outbackcdx/UrlCanonicalizerTest.java` | SURT canonicalisation of various URL forms |
| `test/outbackcdx/VarIntTest.java` | VarInt encode/decode correctness |
| `test/outbackcdx/WbCdxApiTest.java` | Query wildcard expansion, field sorting, POST URL key encoding |

## Test Infrastructure

- `WebappTest` creates a real `DataStore` backed by a JUnit `TemporaryFolder`, so it exercises actual RocksDB read/write paths.
- `IndexTest` and `AccessControlTest` use `RocksMemEnv` (an in-memory RocksDB environment) for speed and isolation.
- `DummyRequest.java` is a helper that implements `Web.Request` for testing `Webapp` without an HTTP server.

## Integration Tests

Located in `test-integration/`:

| File | What it tests |
|---|---|
| `test-pywb.sh` | Loads CDX records, starts PyWb, checks playback and CDX API responses |
| `test-openwayback.sh` | Loads CDX records, starts OpenWayback, checks XML API and playback |
| `test-filter.sh` | Tests filter plugin integration |
| `common.sh` | Shared helper functions (`launch_cdx`, `check`, etc.) |
| `config.yaml` | PyWb configuration pointing at the test OutbackCDX instance |
| `wayback.xml` | OpenWayback `RemoteCollection` configuration |
| `test1.cdx`, `test2.cdx` | CDX fixture records from real web archive crawls |
| `test1.warc.gz`, `test2.warc.gz` | Corresponding WARC archives for replay tests |
| `test-resources/outbackcdx/rules.xml` | Access rule XML fixture |

## What the Tests Document

- **Ingestion:** POSTing CDX lines (including CDXJ JSON, aliases, POST data) works correctly.
- **Querying:** Exact, prefix, domain, and range queries return correct results; sort=closest orders by date proximity; collapse deduplicates; field selection (`fl`) works.
- **XML protocol:** OpenWayback XML `urlquery` and `prefixquery` return valid output with correct pagination and `<closest>` annotation.
- **Replication:** Secondaries reject writes; the `sequence` endpoint works; WAL truncation is accepted.
- **Access control:** Rules are stored and retrieved; access point filtering correctly allows and denies captures.
- **Aliases:** Alias-resolved queries return captures for the target URL.
- **HMAC fields:** Signatures are computed correctly for configured templates.
- **URL canonicalisation:** A broad set of URL variations are tested for correct SURT form.

## Coverage Gaps

- No unit tests cover the WARC replay flow (`Replay.java`).
- No unit tests cover the Keycloak or JWT authorizer in an end-to-end HTTP scenario.
- Integration tests require external tools (PyWb, OpenWayback) and are not run in CI automatically.
- `CaptureBenchmark.java` is a JMH benchmark, not a correctness test — it is not run by `mvn test`.
