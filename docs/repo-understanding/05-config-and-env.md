# Configuration and Environment

## Command-Line Options

OutbackCDX has no configuration file — all settings are passed as CLI arguments at startup.

| Option | Default | Required | Effect |
|---|---|---|---|
| `-p <port>` | `8080` | No | TCP port to listen on |
| `-b <bindaddr>` | `localhost` | No | IP address to bind to |
| `-c <url-prefix>` | (none) | No | URL context path (must start with `/`) |
| `-d <datadir>` | `data` | No | Directory for RocksDB collection subdirectories |
| `-t <count>` | CPU core count | No | HTTP thread pool size |
| `-m <max-open-files>` | auto-calculated | No | Max open RocksDB SST files (controls memory use) |
| `-r <count>` | unlimited | No | Max RocksDB records scanned per request |
| `-v` | off | No | Verbose request logging to stdout |
| `-x` | off | No | Output CDX14 fields by default |
| `-y <file>` | (none) | No | Path to fuzzy-match YAML configuration |
| `-i` | off | No | Inherit server socket from stdin (systemd/inetd) |
| `-j <jwks-url> <perm-path>` | (none) | No | Enable JWT authorization |
| `-k <url> <realm> <clientid>` | (none) | No | Enable Keycloak authorization |
| `--hmac-field <name> …` | (none) | No | Define a computed HMAC/digest field |
| `--warc-base-url <url>` | (none) | No | Enable WARC replay with this file base URL |
| `--service-worker <file>` | (none) | No | Custom JavaScript service worker for replay |
| `--max-num-results <N>` | `10000` | No | Max records scanned for `numresults` in XML protocol |
| `--omit-self-redirects` | off | No | Omit self-redirects from query results by default |
| `--replication-window <secs>` | (none / no WAL retention) | No | Retain WAL for this many seconds (primary mode) |
| `--primary <collection-url>` | (none) | No | Upstream collection URL to poll (enables secondary mode) |
| `--update-interval <secs>` | `10` | No | Polling frequency for primary (secondary mode) |
| `--accept-writes` | off | No | Allow writes on a secondary node |
| `--batch-size <bytes>` | `10485760` (10 MB) | No | Max bytes per replication batch |
| `--index-version <N>` | `3` | No | Index key/value encoding version (3–5; experimental) |

## Environment Variables

| Variable | Default | Effect |
|---|---|---|
| `EXPERIMENTAL_ACCESS_CONTROL` | `0` | Set to `1` to enable access control rules and policies |
| `CDX14` | `0` | Set to `1` to output CDX14 fields by default (same as `-x`) |
| `FILTER_PLUGINS` | `0` | Set to `1` to load `FilterPlugin` implementations via Java `ServiceLoader` |
| `PANDORA_HACKS` | `0` | Set to `1` to enable NLA PANDORA-specific workarounds |
| `CDX_PLUS_WORKAROUND` | `0` | Set to `1` to retry queries replacing `%20` with `+` when no results are found |

## Memory Tuning

RocksDB keeps an in-memory index (binary search table, Bloom filter) for each open SST file.
By default OutbackCDX calculates a safe limit:

```
max_open_files = (totalRAM / 2 - maxJvmHeap) / 10 MB
```

This is capped by the OS `ulimit -n` file descriptor limit minus a safety margin.
Override with `-m <N>` (`-m -1` disables the limit for maximum query performance when RAM is plentiful).

The JVM heap should be limited explicitly, e.g. `-Xmx512m`, because Java defaults to half of physical RAM.

## Fuzzy Match Configuration

The optional `-y` YAML file defines URL rewriting rules that strip session IDs, tracking parameters, and other noise from URLs before they are stored or queried.
See `src/outbackcdx/UrlCanonicalizer.java` for the rule application logic and `test/outbackcdx/WebappTest.java` `setUp()` for an example YAML snippet.

## HMAC Field Configuration

HMAC fields are defined at startup with multiple arguments:

```
--hmac-field <name> <algorithm> <message-template> <value-template> <secret-key> <expiry-secs>
```

Template variables include `$filename`, `$offset`, `$expires`, `$hmac_base64`, `$now`, and others.
Multiple HMAC fields can be defined simultaneously provided they have unique names.
The secret key is a startup argument and is never stored persistently.

## Authorization

By default the server is unauthenticated — all writes are accepted without credentials.
Two optional authorization modes are available:

- **JWT (`-j`):** Validates a bearer token against a JWKS endpoint. The permission path argument specifies where in the JWT payload to find the list of granted permissions.
- **Keycloak (`-k`):** Configures the server to validate tokens issued by a Keycloak realm. Client roles `index_edit`, `rules_edit`, and `policies_edit` control write access.

When neither is configured, all requests are treated as fully permitted (`NullAuthorizer`).

## Replication WAL Retention

The `--replication-window` option sets RocksDB's WAL TTL (`setWalTtlSeconds`).
Without this option, the WAL is not retained beyond the immediate need, and secondary nodes cannot catch up after a gap.
WAL entries older than the window may be automatically deleted; entries can also be manually purged by POSTing to `/<collection>/truncate_replication`.
