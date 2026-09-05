# Architecture and Data Flow

This document details the architectural topology, data ingestion pipeline,
and security model of analyzeIT.

## 1. Data Ingestion Pipeline

1. The tracking client (JavaScript `/hit` or transparent `/pixel.gif`) transmits
   an analytics event to the Caddy reverse proxy across dual-stack, `ipv4.`, or
   `ipv6.` endpoints.
1. Caddy forwards the request across a local Unix Domain Socket directly to the
   Ingestion Service (Go).
1. The Ingestion Service performs real-time GeoIP resolution (country, city,
   ASN/ISP) and processes client IP persistence according to `IP_STORAGE_MODE`
   (`full`, `mask_1_byte`, `mask_2_bytes`, or daily-rotated `hash`).
1. The Ingestion Service immediately responds with `204 No Content` (or a
   zero-allocation 43-byte GIF for `/pixel.gif`) and publishes the raw hit to
   NATS JetStream (`analytics.raw.hit`).
1. The Batch Worker (Go) consumes events in micro-batches (e.g., 5,000 hits per
   1-second flush interval), merges complementary IPv4/IPv6 sessions, evaluates
   threat heuristics (VPN leaks, Tor exit nodes, DS-Lite), and performs
   bulk inserts into Databend.
1. Databend (Rust Data Warehouse) compresses and writes columnar data as
   Apache Parquet files onto local NVMe storage or S3.

## 2. Application Layer, Marketplace, and WebAssembly Runtime

- **Web & API Service (Go)**: Handles user requests, dashboard rendering, and
  REST/JSON endpoints.
- **State Database**: Embedded SQLite database operating in WAL (Write-Ahead
  Logging) mode for tenancy, user credentials, and plugin configurations.
- **Bitcoin & Lightning Marketplace**: Commercial Wasm modules are unlocked via
  Lightning Network invoices (BOLT11 / LNURL) and verified locally through
  cryptographic Ed25519 signatures without external phone-home telemetry.
- **Wazero Runtime**: Sandboxed WebAssembly execution engine invoked strictly
  on-demand for computation-heavy reports or marketplace analytics modules.
  Enforces a strict 64 MB RAM limit and CPU execution budgets.

## 3. Infrastructure Security and Isolation

- **Unix Domain Sockets**: Applications do not bind or expose TCP ports to the
  host. All communication with the reverse proxy traverses shared sockets.
- **nftables**: Strict host-level firewall rules isolating container traffic.
  Containers cannot initiate arbitrary egress connections.
- **Zero Cron**: No hidden background cron daemons run inside containers.
  Scheduled tasks use in-process Go tickers or host `systemd.timer` units.
