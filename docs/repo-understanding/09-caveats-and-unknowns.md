# Caveats and Unknowns

## Missing Context

- **PANDORA_HACKS (`PANDORA_HACKS=1`):** The NLA PANDORA-specific workarounds guarded by this flag are not documented in the repository. The code exists in `FeatureFlags.java` and is referenced elsewhere, but the actual behaviour it enables is not visible without additional context from NLA developers.

- **`CDX_PLUS_WORKAROUND`:** The environment variable is described in comments as a workaround for bad WARC files that use `+` instead of `%20` in URLs. Whether this workaround is still needed or is being phased out is not clear from the codebase alone.

- **`bin/` directory:** Listed in the repository root but not explored in detail. Its contents and purpose are not confirmed.

- **`nla-deploy.sh`:** Exists at the repository root. Its contents and the deployment environment it targets have not been reviewed.

## Ambiguous Naming

- **"access point"** is used both in the access control model (a named query context like `public` or `staff`) and loosely in HTTP URL paths (`/ap/<accesspoint>`). The two usages are consistent but may cause confusion when reading code.

- **`compressedoffset` and `length`** in `Capture.java` refer to the byte offset and length of the WARC record, not to any compression-related measurement. The naming reflects legacy CDX terminology.

- **`originalLength`, `originalCompressedoffset`, `originalFile`** (CDX14 fields) — their relationship to the non-`original` fields is not fully explained in code comments. They appear to represent the location of an original uncompressed or deduplicated record.

## External Systems Not in the Repo

- **Keycloak server:** The Keycloak integration requires an externally running Keycloak instance. No local Keycloak setup or mock is included.

- **OpenWayback / PyWb:** Integration test scripts depend on these tools being installed locally. They are not bundled.

- **WARC storage:** The replay feature assumes WARC files are accessible at a configured base URL (file system or HTTP). The WARC storage infrastructure itself is external.

- **Heritrix:** Mentioned in the README as a consumer. The integration classes are in the `ukwa-heritrix` project, not here.

## Behaviour Inferred from Tests but Not Fully Visible in Implementation

- The exact semantics of `collapseToLast` are implemented as a wrapping iterator (`Filter.collapseToLast`) and tested indirectly, but the precise field(s) used for collapse comparison when applied to `collapseToLast` (as opposed to `collapseToFirst`) are not immediately obvious from the test coverage alone.

- The `DummyRequest.java` helper implements `Web.Request` for testing but does not cover all HTTP edge cases (e.g., chunked transfer encoding, keep-alive). Production request handling behaviour under those conditions is not tested in the unit test suite.

## Behaviour Visible in Code but Not Covered by Tests

- **`Replay.java` (WARC replay):** No unit tests exercise `Replay.java`. The replay flow is tested only via the integration test scripts, which require external tools.

- **Undertow backend (`UWeb.java`):** There are no tests for the Undertow HTTP server path (`-u` flag). The default JDK `HttpServer` path is exercised by all unit tests.

- **`cube` endpoint:** The `/<collection>/cube` endpoint returns a per-host/mimetype/status/year aggregate. No tests verify its output format or correctness.

- **Background compaction and upgrade (`compact`, `upgrade` endpoints):** These trigger asynchronous RocksDB operations. No tests verify that the background threads complete correctly or that the compacted/upgraded index remains queryable.

- **`--hmac-field` with multiple fields defined simultaneously:** `HmacFieldTest.java` tests individual field computation, but the interaction between multiple HMAC fields defined at the same time is not tested.

## Questions for Maintainers

- Is the `bin/` directory used? If so, what scripts does it contain and what are they for?
- Is `PANDORA_HACKS` mode still in active use? What behaviour does it enable?
- Is index version 4 considered stable enough to use in production, or only version 3?
- Is index version 5 (CBOR / CDXJ) intended for general use, or is it still experimental?
- What is the intended upgrade path from index version 3 to version 4 when both old and new records exist?
- Is the Undertow backend (`-u`) maintained for any specific production use case, or is it an alternative that may be removed?
- Are there plans to make access control non-experimental (i.e., enabled by default)?
