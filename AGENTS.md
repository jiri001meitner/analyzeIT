# AGENTS.md

This document is the primary and single source of truth for AI agents and
developers working within the **analyzeIT** repository.

Client files:

- `CLAUDE.md`
- `GEMINI.md`
- `COPILOT.md`

must be strictly relative symlinks to this file (`AGENTS.md`). All clients and
agents share a single source of truth.

---

## 1. Instruction and Context Priority

Always follow this hierarchy when working on the project:

1. User instructions and prompt rules.
1. This file (`AGENTS.md`).
1. `README.md` and `ARCHITECTURE.md` in the root directory.
1. Knowledge base (`knowledge/`).
1. Configuration files (`docker-compose.yml`, `.env.example`).
1. General model knowledge.

Always load context from `AGENTS.md` and `README.md` first and understand the
current state before making changes.

---

## 2. Language and Communication

1. **User Communication (Czech)**: Always communicate with the user in the
   Czech language. This instruction has the highest priority.
1. **Repository Language (English)**: All code, identifiers, function and
   variable names, comments, documentation, commit messages, knowledge base
   articles (`knowledge/`), and skills (`skills/`) must be in English.
1. **Communication Style**: Keep responses technical, concise, accurate, and
   direct. Avoid unnecessary fluff.
1. **Uncertainty**: When unsure or missing requirements, do not guess; state
   the uncertainty explicitly or ask for clarification.
1. **File References**: When referencing files, use markdown links in the
   format `[filename](file:///path/to/file)`.

---

## 3. Architecture and Technology Stack

**analyzeIT** is a high-performance, self-hosted web analytics platform
engineered for high throughput, memory safety, and cloud-native operation:

1. **Ingestion Service (Go)**:
   - Ultra-fast HTTP event collection (`net/http`) via `204 No Content`
     for API hits and zero-allocation 43-byte 1x1 GIF (`/pixel.gif`).
   - Real-time GeoIP resolution (country, city, ASN/ISP).
   - Configurable IP retention (`full`, `mask_1_byte`, `mask_2_bytes`, `hash`).
   - Dual-stack DNS routing (`ipv4.` and `ipv6.`) over shared Unix socket.
   - Offload payload directly to NATS without blocking on disk I/O.
1. **Message Broker (NATS JetStream)**:
   - In-memory buffer absorbing traffic spikes and isolating outages.
1. **Batch Worker (Go)**:
   - Micro-batch consumer from NATS (e.g., 5,000 events / 1s interval).
   - Session-level IPv4/IPv6 merging and threat classification (VPN leak,
     Tor exit node, and DS-Lite / CGNAT detection).
   - Efficient bulk insert into Databend analytical storage.
1. **Analytical Storage (Databend / Rust)**:
   - Cloud-native Data Warehouse written in Rust (eliminates C++ memory leaks).
   - Columnar Apache Parquet storage on local SSD or S3 backend.
   - Decoupled compute (`databend-query`) and metadata (`databend-meta`).
1. **Web & API Service (Go + SQLite)**:
   - Zero-overhead state database in SQLite with Write-Ahead Logging (WAL)
     for users, site settings, and plugin configurations.
   - REST/JSON API and web interface.
1. **Wasm Marketplace Runtime (Wazero)**:
   - Sandboxed WebAssembly runtime executing analytics modules on-demand.
   - Strict boundaries: 64 MB RAM limit and CPU execution quotas.
   - Bitcoin & Lightning Network payments with offline Ed25519 licensing.
1. **Hardened Network Security**:
   - Zero exposed TCP container ports on host interfaces.
   - Reverse proxy communication (Caddy/Nginx) exclusively over Unix Domain
     Sockets (bind-mounted shared socket).
   - Container network isolation via host `nftables`.
1. **Zero Cron**:
   - No cron daemons running inside containers.
   - Periodic routines handled by internal Go tickers or host `systemd.timer`
     units.

---

## 4. Development Rules and Code Quality

1. **Zero Warning Policy**:
   - Strictly prohibit proactive warning suppression (warnings/deprecations).
   - Never silence linter errors in config; always fix root causes.
1. **No Unrequested Refactoring**:
   - Never perform unprompted refactoring on functional code unrelated to the
     task. Every change must be justified and approved.
1. **Go Standards**:
   - Strict formatting with `gofmt` and `goimports`.
   - Idiomatic error handling (`if err != nil`), never swallow errors.
   - Use `context.Context` for timeouts and graceful shutdown.
1. **Public Repository Security**:
   - This is a public repository. Never commit credentials, tokens, API keys,
     private keys, certificates, or rotated salts.
   - Local configs belong strictly in `.env` (ignored in `.gitignore`). Only
     the `.env.example` template is tracked.
   - Prohibit committing SQLite databases, unix sockets, and storage volumes.
   - Prevent all tool caches (Go `.gocache/`, `vendor/`, Rust `target/`,
     Node.js `node_modules/`, Python `__pycache__/`) from being tracked.
   - An active pre-commit hook (`.githooks/pre-commit`) blocks prohibited
     files and secret patterns.
1. **Documentation Integrity**:
   - Store all project documentation directly in the repository (e.g.,
     `README.md`, `ARCHITECTURE.md`, `knowledge/`), never in temporary artifact
     folders.
   - Markdown files must pass `mdl` validation (line length <= 80 chars,
     consistent indentation, blank lines around headings and code blocks).

---

## 5. Knowledge Base (`knowledge/`)

Operational knowledge, non-trivial architecture decisions, and diagnostics
are recorded in `knowledge/`:

1. **One topic = one file**: Short, focused English documents.
1. **Index**: Every document must be listed in `knowledge/INDEX.md` with a
   one-line summary.
1. **Targeted Reading**: Search knowledge with `rg <topic> knowledge/` and
   read only matched files. Never dump the entire directory into context.

---

## 6. Agent Skills (`skills/`)

The `skills/` directory holds reusable, specialized procedures and guidelines
for AI agents:

1. Each skill has its own subdirectory with a `SKILL.md` file.
1. Skills extend agent capabilities for specific tasks (e.g., Wasm compilation,
   ingestion load benchmarks, Parquet dataset generation).
