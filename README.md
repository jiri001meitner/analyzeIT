# analyzeIT 📊

analyzeIT is a high-performance, self-hosted web analytics platform engineered
for extreme throughput, memory safety, and cloud-native infrastructure.

Instead of traditional C++ databases, the architecture is built strictly on a
modern, memory-safe stack: the core in **Go**, **NATS JetStream** as an
in-memory buffer, and **Databend** (a cloud-native Data Warehouse written in
Rust) storing columnar Apache Parquet files on local storage or S3.

A defining feature of analyzeIT is its **WebAssembly (Wasm) Marketplace** with
native **Bitcoin & Lightning Network** micro-licensing. Complex reports and
attribution models execute strictly on-demand in sandboxed environments.

## ✨ Key Features

- **Extreme Ingestion Throughput:** Lightweight Go collector returning
  `204 No Content` for API hits or streaming a zero-allocation 43-byte 1x1 GIF
  (`/pixel.gif`) for no-script and email tracking.
- **Configurable IP Retention & GeoIP:** Real-time GeoIP resolution (country,
  city, ASN/ISP) with configurable persistence modes: `full` (Matomo-style),
  subnet-masked (`mask_1_byte`, `mask_2_bytes`), or salted `hash` for strict
  GDPR environments.
- **Dual-Stack IPv4 & IPv6 Telemetry:** Native client probing capturing both
  IPv4 and IPv6 endpoints, unmasking VPN IPv6 leaks, identifying DS-Lite / CGNAT
  topologies, and performing optional WebRTC STUN candidate discovery.
- **Cloud-Native Analytics in Rust:** Databend handles writes and analytical
  aggregations. Compute (Query nodes) is completely decoupled from storage
  (Apache Parquet on NVMe or S3/MinIO), eliminating C++ memory leaks.
- **Extensibility & Bitcoin Licensing:** Run Wasm plugins safely in Wazero with
  strict 64 MB RAM boundaries. Commercial plugins are unlocked via Lightning
  Network invoices and validated offline via Ed25519 signatures.
- **Hardened Network Security:** No exposed container TCP ports on the host.
  Communication with reverse proxies (Caddy/Nginx) occurs exclusively via
  Unix Domain Sockets and host-level `nftables` isolation.
- **Zero Cron:** Containers run zero background cron daemons. Scheduled
  routines are managed by internal Go tickers or host `systemd.timer` units.

## 🚀 Quick Start (Self-Hosted)

The platform is orchestrated via `docker-compose`. Transactional state
(users, site configurations, plugin settings) is stored in embedded SQLite
(WAL mode), while analytical data is stored via Databend as Parquet files.

```bash
git clone https://github.com/jiri001meitner/analyzeIT.git
cd analyzeIT
docker compose up -d
```

## 🧩 Wasm Marketplace

Developers can contribute custom analytical modules compiled to `.wasm`.
While the core system is open-source, third-party marketplace modules can be
monetized globally via Bitcoin Lightning Network micro-payments.
