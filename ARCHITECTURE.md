# analyzeIT Architecture

This document details the internal design and core principles of analyzeIT.
The primary objectives are decoupling latency-critical ingestion from
analytical workloads, enforcing memory safety (Rust and Go over C++), providing
granular and configurable IP retention, and safely executing third-party code.

## 🏗️ Topology and Data Flow

```text
[ Tracking Client: JS (/hit) or Pixel (/pixel.gif) ]
            │ (HTTPS POST / GET over dual-stack, IPv4, or IPv6)
            ▼
[ Caddy Reverse Proxy ]
            │ (Unix Domain Socket /var/run/analyzeit.sock)
            ▼
1. 📥 INGESTION SERVICE (Go)
            │ (HTTP 204 or 43-byte GIF, GeoIP & IP mode processing)
            │ Publish("analytics.raw.hit")
            ▼
2. 📨 MESSAGE BROKER (NATS JetStream)
            │ In-memory buffer absorbing traffic spikes
            ▼
3. ⚙️ BATCH WORKER (Go)
            │ Fetch 5,000 msgs / 1,000 ms flush interval
            │ Merges dual-stack IPv4/IPv6 sessions, evaluates leaks
            │ Bulk Insert
            ▼
4. 🗄️ DATABEND (Rust Data Warehouse)
            │ ├── databend-meta (Cluster metadata)
            │ └── databend-query (Compute and Parquet storage on disk)
            ▲
            │ (SQL queries over aggregated session data)
            │
5. 💻 WEB & API SERVICE (Go + SQLite)
            │
            ├──► ⚡ LIGHTNING MARKETPLACE (Bitcoin / LN Payments)
            │    Issues offline Ed25519 cryptographic license tokens.
            │
            └──► 🧩 WASM RUNTIME (Wazero)
                 On-demand execution of sandboxed Wasm plugins over
                 aggregated datasets from Databend.
```

## 🔒 Network Isolation and Security

The architecture avoids Docker's default behavior of exposing container
ports to the host via `iptables`. It is designed for hardened Linux
environments:

1. **Unix Sockets:** Application containers (Ingestion, API) communicate with
   the reverse proxy strictly via bind-mounted Unix Domain Sockets. This removes
   TCP/IP stack overhead and prevents containers from exposing network ports.
1. **nftables:** Host and bridge network interfaces are protected natively via
   `nftables`. Containers communicate strictly on internal Docker bridge
   networks with no external exposure, cannot reach arbitrary targets, and
   cannot initiate outbound internet connections.
1. **Memory Safety (Rust and Go):** Complete elimination of C++ database
   components (using Databend instead of ClickHouse) eliminates buffer overflow
   risks and memory corruption bugs in data processing layers.

## 🧠 Component Roles

### Ingestion Service (Go, net/http)

Specialized for low latency and high throughput:

- **Dual Ingestion Endpoints**: Responds with `204 No Content` for standard
  REST hits (`/hit`), or streams a zero-allocation 43-byte transparent GIF
  (`/pixel.gif`) for no-script and email tracking.
- **Immediate GeoIP Resolution**: Performs in-memory lookup of Country, Region,
  City, and ASN/ISP before any IP transformation occurs.
- **Configurable IP Retention**: Governed by `IP_STORAGE_MODE`: `full` retains
  complete addresses (Matomo-style), `mask_1_byte` or `mask_2_bytes` masks
  lower subnets, and `hash` uses a daily-rotated salt for strict GDPR.
- **Dual-Stack DNS Routing**: Receives traffic from dual-stack, `ipv4.`, and
  `ipv6.` hostnames over a single shared Unix Domain Socket.

### NATS JetStream (Message Broker)

Functions as an in-memory buffer and resilience shock-absorber. If the storage
layer is paused for maintenance or updates, the Ingestion Service continues
buffering incoming hits uninterrupted, preventing data loss during traffic
spikes.

### Batch Worker (Go)

Columnar analytical databases perform best with batched writes rather than
single-row inserts:

- Consumes raw hits from NATS in micro-batches (e.g., 5,000 records).
- Merges complementary IPv4 and IPv6 hits belonging to the same `visitor_id`.
- Evaluates threat intelligence heuristics: flags IPv6 VPN leaks, identifies
  Tor exit nodes, and classifies DS-Lite / CGNAT connection profiles.
- Executes efficient bulk inserts into Databend.

### Databend (Storage Layer)

A cloud-native Data Warehouse written in Rust that decouples compute
(`databend-query`) from metadata (`databend-meta`). In this self-hosted
architecture, it stores highly compressed Apache Parquet files directly on local
storage via its built-in `fs` storage backend, delivering extreme query
performance without memory unsafety.

### Web & API Service (Go + SQLite)

Serves the web dashboard, manages administrative APIs, and orchestrates site
configurations:

- **Zero-Overhead State Database:** Eliminates external database services
  (e.g., Postgres). Tenancy, user accounts, API keys, and plugin licenses are
  persisted in an embedded SQLite database running in WAL mode.
- **Bitcoin & Lightning Marketplace:** Commercial modules are unlocked via
  Lightning Network invoices (BOLT11 / LNURL). The server issues Ed25519-signed
  license tokens that are verified locally offline without phone-home tracking.
- **Wasm Runtime (Wazero):** Integrated WebAssembly sandbox kept out of the
  critical ingestion path. Execution occurs strictly on-demand when users
  request complex analytical reports. The Wasm module receives a query dataset
  from Databend, runs bounded computation (limited to 64 MB RAM and CPU quotas),
  and returns results safely.

## 🕒 Periodic Tasks and Maintenance (Zero Cron)

Containers run **no background cron daemons**. Maintenance tasks (data
rotation, notifications, scheduled reports) are handled by internal Go loops
(`time.Ticker` with graceful shutdown handlers) or host-level `systemd.timer`
units. This prevents orphaned processes and silent failures inside Docker.
