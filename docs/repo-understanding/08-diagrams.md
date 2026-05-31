# Diagrams

## 1. Runtime Flow

```mermaid
flowchart TD
    CLI["java -jar outbackcdx.jar [options]"] --> Main
    Main --> UrlCanonicalizer
    Main --> DataStore
    Main --> Webapp
    Main --> HttpServer["JDK HttpServer / Undertow"]
    Main --> CPT["ChangePollingThread (secondary mode only)"]

    HttpServer -->|request| Auth["Authorizer (JWT/Keycloak/Null)"]
    Auth -->|permitted| Router["Webapp Router"]
    Router -->|GET /<collection>?url=...| WbCdxApi
    Router -->|GET /<collection>?q=...| XmlQuery
    Router -->|POST /<collection>| Ingest["Ingest Handler"]
    Router -->|GET /<collection>/changes| ChangeFeed["Change Feed Handler"]
    Router -->|GET /<collection>/<date>/<url>| Replay

    WbCdxApi --> Index
    XmlQuery --> Index
    Ingest --> Index
    ChangeFeed --> Index
    Replay --> Index
    Replay --> WARC["WARC file (file:// or http://)"]

    Index --> RocksDB[("RocksDB\n(per collection)")]
    CPT -->|poll primary| ChangeFeed
    CPT --> RocksDB

    DataStore --> RocksDB
```

---

## 2. Module / File Interaction

```mermaid
graph TD
    Main --> Webapp
    Main --> DataStore
    Webapp --> WbCdxApi
    Webapp --> XmlQuery
    Webapp --> AccessControl
    Webapp --> Replay
    Webapp --> DataStore
    DataStore --> Index
    Index --> RocksDB[("RocksDB")]
    Index --> UrlCanonicalizer
    WbCdxApi --> Query
    WbCdxApi --> Capture
    XmlQuery --> Capture
    Query --> UrlCanonicalizer
    Query --> Index
    Capture --> VarInt
    AccessControl --> RocksDB
    ChangePollingThread --> Index
    Web["Web.java\n(HTTP framework)"] --> Webapp
    Auth["auth/\n(Authorizer)"] --> Web
    HmacField --> Capture
    Filter --> Capture
```

---

## 3. Sequence Diagram — CDX Query (PyWb Protocol)

```mermaid
sequenceDiagram
    participant PyWb as PyWb / Client
    participant Webapp
    participant Query
    participant UrlCanonicalizer
    participant Index
    participant RocksDB

    PyWb->>Webapp: GET /<collection>?url=http://example.org/&output=json
    Webapp->>Query: new Query(params)
    Query->>UrlCanonicalizer: surtCanonicalize(url)
    UrlCanonicalizer-->>Query: "org,example)/"
    Query->>Index: execute(query)
    Index->>RocksDB: seek("org,example)/")
    loop while captures match & limit not reached
        RocksDB-->>Index: Capture key+value
        Index-->>Query: Capture object
        Query-->>Webapp: Capture (after filter)
        Webapp-->>PyWb: JSON row (streamed)
    end
    RocksDB-->>Index: end of matching range
    Webapp-->>PyWb: end of JSON array
```

---

## 4. Replication Flow (Primary → Secondary)

```mermaid
sequenceDiagram
    participant Secondary as Secondary Node\n(ChangePollingThread)
    participant Primary as Primary Node\n(Webapp)
    participant SecDB as Secondary RocksDB
    participant PrimDB as Primary RocksDB

    loop every --update-interval seconds
        Secondary->>SecDB: read #ReplicationSequence
        SecDB-->>Secondary: lastSeqNo
        Secondary->>Primary: GET /<collection>/changes?since=lastSeqNo&size=batchSize
        Primary->>PrimDB: getUpdatesSince(lastSeqNo)
        loop WAL batches available & size < batchSize
            PrimDB-->>Primary: WriteBatch
            Primary-->>Secondary: {sequenceNumber, writeBatch (base64)}
        end
        loop for each received batch
            Secondary->>SecDB: apply WriteBatch + update #ReplicationSequence
        end
    end
```

---

## 5. Access Control Evaluation

```mermaid
flowchart LR
    Q["Query /<collection>/ap/public?url=..."]
    --> AP["Extract access point: 'public'"]
    --> AC["AccessControl.filter('public', now)"]
    --> Pred["Predicate<Capture>"]
    --> Iter["RocksDB iterator over matching keys"]
    --> Check{Rule match\nfor this capture?}
    Check -- no rule matched --> Allow["Include in results"]
    Check -- rule matched --> PolicyCheck{Policy allows\n'public' access point?}
    PolicyCheck -- yes --> Allow
    PolicyCheck -- no --> Deny["Omit from results"]
```
