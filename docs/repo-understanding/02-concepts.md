# Core Concepts

## CDX Record (Capture)

A CDX record is a single metadata entry describing one archived copy of a web page.

- **Fields:** canonicalised URL (urlkey), timestamp, original URL, MIME type, HTTP status, digest, WARC filename, byte offset, and record length.
- **In the code:** represented by `Capture.java`.
- **In storage:** the RocksDB key is the SURT-canonicalised URL concatenated with the timestamp as a big-endian 64-bit integer; the value is VarInt-packed fields (index version 3/4) or CBOR (version 5).
- CDX records are ingested by POSTing them in the standard 11-field CDX text format to `/<collection>`.

## SURT (Sort-friendly URI Reordering Transform)

SURT is a rearranged form of a URL used as the RocksDB key.

- A URL like `http://www.example.org/path` becomes `org,example)/path`.
- This puts all URLs for the same domain together in sorted order, which makes prefix and domain queries fast without secondary indexes.
- SURT conversion is handled by `UrlCanonicalizer.java`.

## Collection

A collection is a named, independently queryable CDX index.

- Each collection is a separate RocksDB database stored in a subdirectory of the data directory (e.g. `data/myindex/`).
- Collections are created automatically on first write.
- Valid collection names match `[A-Za-z0-9_-]+` (enforced in `DataStore.java`).
- `DataStore.java` manages all open collections.

## Access Point

An access point is a named context through which a query is made (e.g. `public`, `staff`).

- Access points are not user accounts; they represent *categories* of access (location, role, or use case).
- When access control is enabled, a query to `/<collection>/ap/<accesspoint>?url=…` is filtered by the rules that apply to that access point.
- Defined and evaluated in `AccessControl.java`.

## Access Policy

A policy is a named set of access points.

- Rules reference policies rather than individual access points, so changing which access points belong to a policy updates all associated rules at once.
- Stored in the `access-policy` RocksDB column family.

## Access Rule

A rule associates a URL pattern (and optional date range) with a policy.

- Criteria: URL pattern(s), optional capture date range, optional access date range.
- Rules are stored in the `access-rule` RocksDB column family and also kept in-memory in a concurrent inverted radix tree for fast lookup.
- Managed by `AccessControl.java`.

## Alias

An alias maps one canonicalised URL to another in the index.

- Allows `legacy.example.org/page-one` queries to resolve to `www.example.org/page1` captures.
- Aliases are stored in a separate `alias` RocksDB column family.
- When a query arrives, the SURT key is resolved through any applicable alias before the main index is searched.
- Ingested with `@alias <alias-url> <target-url>` lines in a CDX POST.

## Primary / Secondary Mode

OutbackCDX supports a replication topology where one **primary** node accepts writes and one or more **secondary** nodes maintain read-only copies.

- **Primary:** exposes a `/<collection>/changes` endpoint that streams RocksDB WAL (Write-Ahead Log) batches as JSON.
- **Secondary:** runs `ChangePollingThread` which polls the primary's `/changes` endpoint at a configurable interval and applies the received write batches locally.
- Secondary nodes reject writes by default (unless `--accept-writes` is set).
- Implemented in `ChangePollingThread.java` and the `changeFeed` handler in `Webapp.java`.

## Fuzzy Matching

Fuzzy matching allows queries for non-identical but semantically equivalent URLs to find captures.

- Configured via a YAML file (passed with `-y`); rules strip parameters that don't affect the server response (e.g. session IDs, analytics tokens).
- Implemented as canonicalization pre-processing in `UrlCanonicalizer.java`.
- At query time, a fuzzy-matched query becomes a SURT prefix scan followed by content filtering.

## HMAC Field

A computed field that generates a cryptographic signature (HMAC or digest) over capture metadata.

- Used to produce signed URLs for time-limited access to WARC files hosted on nginx, lighttpd, or S3.
- Configured with `--hmac-field` at startup; the field becomes available as a named output field in CDX query responses.
- Implemented in `HmacField.java`.

## Feature Flag

Runtime toggles that gate experimental or optional behaviour.

- Set via environment variables at startup (`EXPERIMENTAL_ACCESS_CONTROL`, `CDX14`, `FILTER_PLUGINS`, `PANDORA_HACKS`) or CLI arguments (`--index-version`, `--primary`).
- Centralised in `FeatureFlags.java`.
- Many branches throughout the codebase read from `FeatureFlags` to decide which code paths to take.

## Index Version

The binary encoding version used for RocksDB keys and values.

- Version 3 (default): SURT + timestamp key; VarInt-packed value.
- Version 4: extends the key to include WARC filename and offset, enabling deduplication of records with the same URL/timestamp but different storage locations.
- Version 5: uses CBOR encoding for values, enabling arbitrary CDXJ extra fields.
- Changing version on an existing index is not supported for version 3 → 4.
- Controlled by `--index-version` CLI argument and `FeatureFlags.indexVersion()`.

## VarInt Encoding

CDX record values are serialised using protobuf-style base-128 variable-length integers.

- Strings are length-prefixed; numeric fields are packed compactly.
- Combined with Snappy block compression in RocksDB, this achieves approximately 1/4 to 1/5 the on-disk size of raw CDX files.
- Implemented in `VarInt.java`.
