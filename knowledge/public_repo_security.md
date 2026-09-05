# Public Repository Security and Leak Prevention

**analyzeIT** is developed in a public GitHub repository. Storing secrets,
operational credentials, sensitive customer data, or tool caches in git is
strictly forbidden.

## 1. Prohibited Items (Never Commit)

- **Secrets and Configs**: `.env`, `.env.*` (only `.env.example` is tracked),
  API tokens, plaintext passwords, rotated salts, certificates (`.pem`,
  `.crt`, `.pfx`), and private keys (`.key`, `id_rsa*`, `id_ed25519*`).
- **Databases and State Data**: SQLite databases (`*.db`, `*.sqlite*`, `*-wal`,
  `*-shm`), Databend Parquet chunks, and storage volumes (`data/`, `storage/`,
  `minio_data/`).
- **Runtime Sockets**: Unix Domain Sockets (`*.sock`, `run/`).
- **Tool and Build Caches**: Go (`.gocache/`, `pkg/mod/`, `bin/`, `dist/`,
  `vendor/`), Rust/Wasm (`target/`, `*.wasm`, `*.wasm.cache`, `.wazero/`),
  Node.js / Python (`node_modules/`, `__pycache__/`, `*.pyc`).

## 2. Local Defense Layers

1. **Strict `.gitignore`**: Ensures untracked runtime artifacts and caches are
   excluded from `git status` and standard staging commands.
1. **Pre-commit Hook (`.githooks/pre-commit`)**: Evaluates staged paths and
   inspects diffs for private keys, AWS access keys, and GitHub tokens before
   allowing a commit to finalize.
1. **Configured Hook Path**: Automatically active in repository clones via:
   `git config core.hooksPath .githooks`.

## 3. GitHub Server-Side Protection

1. **Secret Scanning and Push Protection**: Native GitHub security (enabled).
   GitHub analyzes commits server-side and blocks pushes containing recognized
   secrets.
1. **Branch Protection Ruleset (`Branch Protection (main)`)**: Active ruleset on
   `main` preventing branch deletion (`deletion`) and forced pushes
   (`non_fast_forward`).
1. **CI Security Audit (`security.yml`)**: GitHub Actions workflow executing
   Gitleaks scans and verifying that no prohibited files or caches are tracked.
1. **Dependabot**: Automated security alerts and patch pull requests for
   vulnerable dependencies.
1. **Push Rules Gotcha**: Rulesets with `target: push` (such as
   `file_path_restriction`) are restricted to GitHub Organizations. For
   personal accounts, file path exclusion is enforced by the local pre-commit
   hook and the CI audit workflow.

## 4. Development Setup

Developers instantiate local environments exclusively from the template:

```bash
cp .env.example .env
```

The resulting `.env` file remains local on the host and is ignored by Git.
