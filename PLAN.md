# analyzeIT Implementation Plan

This document outlines the implementation phases, deliverables, and acceptance
criteria for the analyzeIT platform.

## Phase 1: Repository Setup and Baseline Infrastructure

- [x] Project documentation (`README.md`, `ARCHITECTURE.md`, `AGENTS.md`)
- [x] Agent instructions and symlinks (`CLAUDE.md`, `GEMINI.md`, `COPILOT.md`)
- [x] Security controls, `.gitignore`, `.env.example`, and pre-commit hook
- [x] GitHub security (Push Protection, branch protection ruleset)
- [x] GitHub Actions CI audit workflow (`security.yml`)

## Phase 2: Core Ingestion Pipeline and Network Probing

- [ ] Ingestion Service in Go (`net/http` handler on Unix Domain Socket)
- [ ] Transparent 1x1 tracking pixel endpoint (`/pixel.gif` zero-allocation)
- [ ] In-memory site token validation
- [ ] GeoIP resolution engine (country, region, city, ASN/ISP lookup)
- [ ] Configurable IP storage mode (`full`, `mask_1_byte`, `mask_2_bytes`, `hash`)
- [ ] Dual-stack DNS endpoint routing (`ipv4.` and `ipv6.` via shared socket)
- [ ] Tracking JS with non-blocking dual-stack IP probe (`sendBeacon`/`fetch`)
- [ ] Dual-stack WebRTC STUN probe in tracking JS (IPv4 & IPv6 candidates)
- [ ] Lightweight UDP STUN service integration in Go (port 3478)
- [ ] NATS JetStream client publisher (`analytics.raw.hit`)
- [ ] Unit tests and throughput benchmark harness

## Phase 3: Message Broker and Batch Worker

- [ ] NATS JetStream stream definitions and buffer policies
- [ ] Batch Worker in Go consuming events in micro-batches
- [ ] Databend client integration
- [ ] Bulk insert pipeline with session-level IPv4/IPv6 merging
- [ ] Threat classification engine (VPN leak, Tor exit, and NAT type tagging)

## Phase 4: Analytical Storage and Query Layer

- [ ] Databend deployment configuration (`fs` storage backend)
- [ ] Columnar table schemas (Apache Parquet storage)
- [ ] Aggregation query definitions for standard analytics dashboards

## Phase 5: Web & API Service with Wazero Sandbox

- [ ] Web and REST API service in Go
- [ ] Embedded SQLite state database with WAL mode
- [ ] Wazero WebAssembly runtime host with 64 MB memory and CPU limits
- [ ] On-demand plugin execution runner for custom analytics modules

## Phase 6: Wasm Marketplace and Bitcoin/Lightning Licensing

- [ ] Lightning Network invoice generator (BOLT11 / LNURL REST client)
- [ ] Automated revenue splitting (Lightning splits and multi-output routing)
- [ ] Cryptographic license generator (Ed25519 token issuance)
- [ ] Local offline license verification engine in SQLite / API
- [ ] Marketplace catalog browser in web dashboard

## Phase 7: Orchestration and Hardening

- [ ] Multi-container `docker-compose.yml` with Unix Domain Socket mounts
- [ ] Zero exposed container TCP ports (UDP 3478 only for optional STUN)
- [ ] Caddy reverse proxy configuration for dual-stack hostnames
- [ ] Systemd timer templates for automated routines (Zero Cron)
- [ ] End-to-end integration and load testing
