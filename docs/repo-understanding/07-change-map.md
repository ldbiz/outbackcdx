# Change Map

## If I Need to Change Startup or Runtime Behaviour

| Files | Why |
|---|---|
| `src/outbackcdx/Main.java` | Add, remove, or modify CLI options; change defaults; alter server wiring |
| `src/outbackcdx/FeatureFlags.java` | Add a new feature flag or change how existing flags are initialised |
| `src/outbackcdx/DataStore.java` | Change how RocksDB is configured or how collections are opened |

**Caveats:** RocksDB configuration changes (compaction style, write buffer size, compression) are applied when a database is first opened. Changes will not affect already-open databases until the server restarts.

---

## If I Need to Change Configuration

| Files | Why |
|---|---|
| `src/outbackcdx/Main.java` | Add a new CLI option and wire it through to the relevant component |
| `src/outbackcdx/FeatureFlags.java` | Add a new environment-variable-controlled flag |
| `src/outbackcdx/QueryConfig.java` | Add a per-query default that can be overridden globally at startup |

**Caveats:** There is no config file — all configuration is via CLI arguments and environment variables. Persistent settings must be passed every time the server starts.

---

## If I Need to Change Request Handling or Core Query Behaviour

| Files | Why |
|---|---|
| `src/outbackcdx/Webapp.java` | Add or modify an HTTP route; change how a request is pre-processed or authorised |
| `src/outbackcdx/Query.java` | Change how query parameters are parsed or how match types/sort modes work |
| `src/outbackcdx/WbCdxApi.java` | Change the CDX/JSON output format or add a new output format |
| `src/outbackcdx/XmlQuery.java` | Change the OpenWayback XML output format |
| `src/outbackcdx/Index.java` | Change how captures are iterated or add a new query type |
| `src/outbackcdx/Capture.java` | Change CDX record fields or binary encoding |
| `src/outbackcdx/Filter.java` | Change filter predicate logic or collapse behaviour |

**Caveats:** `Capture.java` changes that affect the binary encoding must be versioned carefully. The index version controls which decode/encode path is used; changing the encoding without bumping the version will silently corrupt existing records.

---

## If I Need to Change URL Canonicalization

| Files | Why |
|---|---|
| `src/outbackcdx/UrlCanonicalizer.java` | Change built-in SURT rules, session ID patterns, or fuzzy matching logic |
| YAML config file (passed with `-y`) | Add fuzzy match rules without code changes |

**Caveats:** Changes to canonicalization affect how URLs are stored. Records already in the index were stored with the old canonicalization. Modifying built-in rules can cause queries using the new rules to miss records indexed under the old form.

---

## If I Need to Change Access Control

| Files | Why |
|---|---|
| `src/outbackcdx/AccessControl.java` | Change rule evaluation logic, policy lookup, or default policy creation |
| `src/outbackcdx/AccessRule.java`, `AccessPolicy.java` | Change the rule/policy data model |
| `src/outbackcdx/Webapp.java` | Add or modify access control HTTP endpoints |
| `src/outbackcdx/auth/` | Change how write operations are authorised (JWT, Keycloak, null) |
| `docs/access-control.md` | Update documentation of the access control model |

**Caveats:** Access control must be enabled with `EXPERIMENTAL_ACCESS_CONTROL=1`. Enabling it adds new RocksDB column families; once enabled on a database, those families exist permanently.

---

## If I Need to Change Replication

| Files | Why |
|---|---|
| `src/outbackcdx/ChangePollingThread.java` | Change secondary polling behaviour, batch handling, or error recovery |
| `src/outbackcdx/Webapp.java` (methods `changeFeed`, `flushWal`, `sequence`) | Change the primary-side change feed endpoint |

**Caveats:** The change feed uses RocksDB WAL batches serialised as base64. The format is an internal RocksDB binary format and is not version-stable across major RocksDB releases.

---

## If I Need to Change the Storage Format

| Files | Why |
|---|---|
| `src/outbackcdx/Capture.java` | Key and value encoding for all index versions |
| `src/outbackcdx/VarInt.java` | VarInt codec used by index versions 3/4 |
| `src/outbackcdx/FeatureFlags.java` | Index version selection |
| `src/outbackcdx/Index.java` (methods `upgradeInBackground`) | Background schema migration |

**Caveats:** Index version upgrades (v3 → v4) are explicitly noted as unsupported in existing production databases. Any storage format change should be accompanied by a new index version number and a migration path.

---

## If I Need to Change the Web UI or API Documentation

| Files | Why |
|---|---|
| `resources/outbackcdx/dashboard.html` | Vue.js SPA source |
| `resources/outbackcdx/api.html` | Swagger/ReDoc API documentation page |
| `docs/resources/swagger.json` | OpenAPI spec (served as `/swagger.json`) |
| `resources/outbackcdx/replay.js` | Replay service worker |

**Caveats:** UI assets are bundled into the JAR at build time. Changes require a rebuild.

---

## If I Need to Change Tests or Fixtures

| Files | Why |
|---|---|
| `test/outbackcdx/WebappTest.java` | Add/modify HTTP-level tests |
| `test/outbackcdx/IndexTest.java` | Add/modify index query tests |
| `test/outbackcdx/AccessControlTest.java` | Add/modify access control tests |
| `test/outbackcdx/DummyRequest.java` | Update the test HTTP request helper if `Web.Request` changes |
| `test-integration/*.sh` | Add/modify integration scenarios |
| `test-integration/test1.cdx`, `test2.cdx` | Update CDX fixtures |
