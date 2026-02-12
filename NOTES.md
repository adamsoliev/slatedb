https://materializedview.io/p/cloud-storage-triad-latency-cost-durability

thesis is object stores to converge on low latency reads and writes with atomicity

PUTs are $0.005 per 1k requests (eg ~$130k per month for 10k requests a second service), there’s no choice but to batch writes. Limiting writes to every 10ms leads to max 100 request per second, which costs ~$1300.

This leads to cost, latency and durability trade off.
⬆️ cost ⬇️ latency ⬆️ durability = sync writes
⬇️ cost ⬆️ latency ⬆️ durability = sync batch writes
⬇️ cost ⬇️ latency ⬇️ durability = async batch writes

SlateDB, by contrast, writes everything (including the WAL) to object storage

SlateDB, by contrast, caches recent writes in memory

Keys are limited to a maximum of 65 KiB (65,535 bytes). Values are limited to a maximum of 4 GiB (4,294,967,295 bytes).

API
put(key, value): Insert a key-value pair. 
get(key): Retrieve a key-value pair.
delete(key): Delete a key-value pair.
scan(range): Scan a range of keys.
flush(): Flush the in-memory data to disk.

Write-ahead log (WAL) - an append-only persistent log
MemTables - a sorted in-memory map; mutable one receives writes while the froze MemTable is flushed in the background
SSTables - a sorted on-disk map;
Compaction - merging of multiple SSTables into a new sequence of range partitioned SSTables
Manifest - metadata

Trade offs

```mermaid
sequenceDiagram
  box Memory
    participant C as Client
    participant MT as MemTable (mutable)
    participant IMT as MemTable (immutable)
  end
  box Disk
    participant L0 as L0 SSTable
    participant SR as SR (sorted run)
  end

  C->>SR: get()
  MT-->>C: key-value pair (if found)
  IMT-->>C: key-value pair (if found)
  L0-->>C: key-value pair (if found)
  SR-->>C: key-value pair (if found)
```

```mermaid
sequenceDiagram
  box Memory
    participant C as Client
    participant MT as MemTable (mutable)
    participant IMT as MemTable (immutable)
  end
  box Object Storage
    participant SST as SSTable
    participant SR as Sorted Run
  end

  C->>MT: put()
  MT-->>C: update OK
  MT-->>IMT: freeze
  IMT-->>SST: write
  SST-->>SR: compact
```
