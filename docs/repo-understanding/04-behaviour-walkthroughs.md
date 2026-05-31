# Behaviour Walkthroughs

## 1. CDX Record Ingestion

**Trigger:** A crawler or indexing tool POSTs CDX records to `/<collection>`.

**Files involved:** `Webapp.java` (`post()`), `Index.java` (`beginUpdate()`, `Batch`), `Capture.java` (`fromCdxLine()`), `UrlCanonicalizer.java`

**Flow:**

- The request body is a newline-delimited stream of CDX-11 or CDX-14 text lines (or CDXJ JSON lines), optionally mixed with `@alias` lines.
- Each regular CDX line is parsed into a `Capture` object. During parsing, the raw URL is canonicalised to SURT form.
- Each `@alias` line is parsed and stored separately in the `alias` column family.
- All records are accumulated into a RocksDB `WriteBatch` and committed atomically.
- Secondary nodes return `403 Forbidden` unless `--accept-writes` is set.

**Inputs:** CDX text (11-field or 14-field format) in the request body.

**Outputs:** `"Added N records\n"` response. RocksDB's sequence number is advanced; if the primary has `--replication-window` set, the WAL is retained for that duration.

**Error handling:** By default, any invalid line aborts the entire batch. With `?badLines=skip`, malformed lines are logged and skipped.

**Tests:** `WebappTest.test()` exercises round-trip ingestion and querying.

---

## 2. CDX Query (PyWb / JSON Protocol)

**Trigger:** A replay tool (PyWb, etc.) sends `GET /<collection>?url=<url>&output=json`.

**Files involved:** `Webapp.java` (`query()`), `WbCdxApi.java` (`queryIndex()`), `Query.java`, `Index.java`, `Capture.java`, `UrlCanonicalizer.java`

**Flow:**

- `Query` is constructed from the HTTP parameters: URL, match type (exact/prefix/domain/range), sort order (default/closest/reverse), field list, limit, and filters.
- The URL is canonicalised to SURT form.
- `Query.execute(index)` selects the appropriate `Index` method (e.g. `query()`, `prefixQuery()`, `domainQuery()`, `closestQuery()`).
- `Index` opens a RocksDB iterator, seeks to the start key, and streams matching `Capture` objects.
- Each capture is serialised by `WbCdxApi` in the requested output format (CDX text, JSON array, JSON dict, or CDXJ).
- Filters (field matching, collapse) are applied as Java `Predicate<Capture>` during streaming; the response is streamed directly to the client without buffering all results.

**Inputs:** HTTP query parameters (`url`, `matchType`, `output`, `limit`, `filter`, `fl`, `from`, `to`, `sort`, `closest`, etc.).

**Outputs:** Streaming response body in the selected format. The `outbackcdx-urlkey` response header carries the canonicalised URL.

**External calls / side effects:** None.

**Tests:** `WebappTest`, `WbCdxApiTest`, `IndexTest`.

---

## 3. OpenWayback XML Query

**Trigger:** OpenWayback's `RemoteResourceIndex` sends `GET /<collection>?q=type:urlquery+url:http%3A%2F%2F…`.

**Files involved:** `Webapp.java` (`query()`), `XmlQuery.java`, `Index.java`, `UrlCanonicalizer.java`

**Flow:**

- When the `q` parameter is present, `Webapp.query()` detects the OpenWayback protocol and constructs an `XmlQuery` instead of a `WbCdxApi` query.
- `XmlQuery` decodes the space-delimited `key:value` query string.
- For `type:urlquery` it streams XML `<result>` elements for each matching capture, including pagination metadata and an optional `<closest>` annotation.
- For `type:prefixquery` it groups captures by URL and emits one `<result>` per unique URL with capture counts and date ranges.
- The response is a streaming XML document following the Wayback `<wayback><results>…</results><request>…</request></wayback>` schema.

**Inputs:** The `q` parameter in OpenWayback query-string format.

**Outputs:** XML response matching the Wayback `RemoteCollection` schema.

**Tests:** `WebappTest` (exercises the XML protocol including pagination and `<closest>` annotation).

---

## 4. Primary-Secondary Replication

**Trigger:** A secondary node polls the primary at a configured interval.

**Files involved:** `ChangePollingThread.java`, `Webapp.java` (`changeFeed()`, `flushWal()`), `Index.java`

**Flow:**

- The secondary's `ChangePollingThread` reads its last applied sequence number from a special RocksDB key (`#ReplicationSequence`).
- It GETs `/<collection>/changes?since=<seqNo>&size=<batchSize>` from the primary.
- The primary's `changeFeed` handler reads WAL entries from RocksDB's `getUpdatesSince()` and streams them as a JSON array, each entry containing a `sequenceNumber` and a base64-encoded `writeBatch`.
- The secondary deserialises each batch and writes it to its local RocksDB, updating `#ReplicationSequence` atomically with each batch.
- If the primary's WAL no longer contains entries for the requested sequence number (due to `--replication-window` expiry), the request fails.

**Inputs:** The sequence number the secondary last successfully applied.

**Outputs:** The secondary's local RocksDB is updated to match the primary up to the received sequence number.

**Tests:** `ReplicationFeaturesTest` tests that write endpoints return `403` on a secondary, and that the sequence endpoint and WAL truncation (`truncate_replication`) work correctly.

---

## 5. Access-Controlled Query

**Trigger:** A client queries via an access point: `GET /<collection>/ap/<accesspoint>?url=…`.

**Files involved:** `Webapp.java`, `Index.java` (`queryAP()`, `prefixQueryAP()`), `AccessControl.java`, `AccessRule.java`, `AccessPolicy.java`

**Flow:**

- Available only when `EXPERIMENTAL_ACCESS_CONTROL=1`.
- `AccessControl.filter(accessPoint, now)` returns a `Predicate<Capture>` that evaluates each capture against the in-memory radix tree of rules.
- The predicate checks whether the capture's URL matches any rule, and whether that rule's policy includes the requested access point.
- Rules can also be scoped by capture date range or access date range.
- The predicate is passed into the standard `Index.query()` or `Index.prefixQuery()` stream, filtering out disallowed captures before they reach the client.
- A separate `check` endpoint (`/<collection>/ap/<accesspoint>/check`) lets clients test access for a specific URL without running a full query.

**Inputs:** Access point name (from the URL path), query parameters.

**Outputs:** Filtered CDX results; captures denied by access rules are silently omitted.

**Tests:** `AccessControlTest`, `WebappTest` (access control enabled in `setUp()`).
