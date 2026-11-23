---
url: https://jetpack-desktop.github.io/publishing/cli-tools.md
description: >-
  Reference for the console jetpack developer CLI (init, mcp, codegen, validate,
  install-local, package) and jetpack-daemon runtime/signing flags.
---

# CLI Tools Reference

JetPack provides two complementary binaries:

* **`jetpack`** (`crates/jetpack-cli`) — Standalone console developer CLI for scaffolding AI-ready projects, running the local MCP server (`stdio`), generating TypeScript opcode contracts, validating manifests, and driving local installs/packaging.
* **`jetpack-daemon`** (`crates/jetpack-daemon`) — Native system-tray orchestrator and catalog operator binary.

## Console Developer CLI (`jetpack`)

### `jetpack init <target_dir> [--app-id <id>] [--template <python|fullstack>]`

Scaffolds a ready-to-run JetPack project in `<target_dir>` with AI instructions (`AGENTS.md`, regular-file mirror `CLAUDE.md`, `.cursor/rules/jetpack.mdc`) and Agent Skills (`.agents/skills/create-jetpack-worker/SKILL.md`, `.agents/skills/add-binary-opcode/SKILL.md`, `.agents/skills/debug-pairing/SKILL.md`). Refuses to overwrite an existing `app.toml`.

```bash
# Python worker starter (default)
jetpack init ./workers/my-ocr-worker --template python

# Fullstack TypeScript (@jetpack/client) + Python worker starter
jetpack init ./apps/my-ai-app --app-id com.dev.my-ai-app --template fullstack
```

### `jetpack mcp`

Starts the local Model Context Protocol (MCP) server (`rmcp`) over `stdio` transport (`stdin`/`stdout` JSON-RPC, all `tracing` diagnostics directed to `stderr`). Bridges AI coding agents with embedded/local documentation (`JETPACK_DOCS_DIR` override supported) and token-authenticated `jetpack-daemon` introspection (`pairing.ipc`).

Exposes 5 tools:

* `search_docs(query, max_results?)` — Searches embedded/local JetPack SDK, security, and `app.toml` docs.
* `get_daemon_status()` — Checks daemon health, version, `127.0.0.1` WebSocket reachability, paired origins, and running workers.
* `read_worker_logs(app_id, lines?)` — Tails live or offline `worker.log` / `daemon.log` lines and crash tracebacks with sensitive token/key redaction.
* `validate_manifest(path)` — Validates `app.toml` against the current `AppManifest` schema.
* `scaffold_app(target_dir, app_id?, template?)` — Scaffolds a `python` or `fullstack` project with AI rules and Agent Skills.

### `jetpack codegen [--worker <path>] [--out <path>] [--strict]`

Parses Python worker source files or directories in pure Rust (`rustpython-parser`, zero Python runtime required) to extract Pydantic `BaseModel`, `TypedDict`, `@dataclass`, `@opcode(...)` / `client.opcode(...)` handlers, and `if opcode == ...` / `match` branches into a strict TypeScript declaration file (`jetpack-env.d.ts` exporting `JetPackOpcodeMap`):

```bash
jetpack codegen --worker worker.py --out src/jetpack-env.d.ts --strict
```

* `--worker <path>`: Python file or directory to inspect (defaults to `worker.py`).
* `--out <path>`: Output TypeScript `.d.ts` file path (defaults to `jetpack-env.d.ts`).
* `--strict`: Exit non-zero if zero opcodes are discovered.

### `jetpack validate [path]`

Validates `app.toml` (defaults to `app.toml` in the current directory, or `<dir>/app.toml` when a directory path is passed) using `#[serde(deny_unknown_fields)]` and semantic security checks:

```bash
jetpack validate ./workers/my-ocr-worker/app.toml
```

### `jetpack install-local [path]`

Delegates to `jetpack-daemon --install-local-app <path>` to package and install a local developer worker directly into `<root>/blobs/sha256/` and `<root>/apps/<app-id>/<version>/` in unverified developer mode (`is_verified = false`, restricted to `127.0.0.1` / `localhost`). Defaults `path` to `.`.

### `jetpack package [path] --out <bundle_dir> [--oci-ref <ref>]`

Delegates to `jetpack-daemon --package-app` to package a validated project into an OCI Image Layout v1.0.0 bundle (automatically excluding `AGENTS.md`, `CLAUDE.md`, `.cursor/`, `.agents/`, `jetpack-env.d.ts`, `.git/`, `.venv/`, and secret files). Defaults `path` to `.`.

***

## Daemon Developer & Publisher Flags (`jetpack-daemon`)

### `--init-app <target_dir>`

Scaffolds a ready-to-run starter Python worker project in `<target_dir>` using the same shared template builder as `jetpack init` (including `AGENTS.md`, `CLAUDE.md`, `.cursor/rules/jetpack.mdc`, and `.agents/skills/`, and refusing to overwrite an existing `app.toml`):

```bash
jetpack-daemon --init-app ./workers/my-ocr-worker
```

Creates:

* `app.toml` — Validated starter manifest using a non-official `com.dev.<slug>` ID (`com.dev.my-ocr-worker`).
* `worker.py` — Executable (`0755` on Unix) Python worker using `jetpack_sdk.JetPackClient`.
* `pyproject.toml` — Project metadata (`requires-python = ">=3.12"`, `jetpack-sdk` dependency).
* `README.md` — Project-specific validation, local install, and packaging commands.
* `AGENTS.md`, `CLAUDE.md`, `.cursor/rules/jetpack.mdc`, and `.agents/skills/*` — AI agent instructions and workflows (automatically excluded from OCI packaging).

### `--validate-manifest <path>`

Parses an `app.toml` file with `#[serde(deny_unknown_fields)]` and runs all semantic and security checks:

```bash
jetpack-daemon --validate-manifest ./workers/my-ocr-worker/app.toml
```

### `--install-local-app <source_dir>`

Validates, packages, and installs a local developer worker directly into `<root>/blobs/sha256/` and `<root>/apps/<app-id>/<version>/` in unverified developer mode (`is_verified = false`):

```bash
jetpack-daemon --install-local-app ./workers/my-ocr-worker
```

* Blocks official `SignedAppCatalog` IDs (`APP_SPOOFING_DETECTED`).
* Displays the app with an `UNVERIFIED / LOCAL DEV` badge in Control Center and restricts WebSocket origins to `127.0.0.1` / `localhost`.

### `--package-app <source_dir> --out <bundle_dir> [--oci-ref <ref>]`

Packages a validated worker directory into a standards-compliant **OCI Image Layout v1.0.0** directory and outputs `manifest_sha256` plus a `CatalogAppEntry` JSON snippet:

```bash
jetpack-daemon \
  --package-app ./workers/my-ocr-worker \
  --out ./dist/my-ocr-worker-oci \
  --oci-ref ghcr.io/your-org/my-ocr-worker:0.1.0
```

Automatically strips `*.key`, `*.pem`, `*.secret`, `jetpack-root-signing*`, `.git/`, `.venv/`, and `__pycache__/` so secrets cannot leak into published layers.

## Catalog Operator Commands

### `--generate-catalog-keypair --out-priv <path>`

Generates an `Ed25519` root keypair using `OsRng`:

```bash
jetpack-daemon --generate-catalog-keypair --out-priv ~/.secrets/jetpack-root-signing.key
```

* Writes the 32-byte seed (64 lowercase hex characters) atomically (`create_new(true)`) with mode `0600` on Unix.
* Prints only the public key hex and the Rust `CATALOG_VERIFY_PUBKEY` literal to stdout.

### `--sign-catalog <catalog_json_path> [--key <priv_key_path>]`

Canonicalizes `SignedAppCatalog`, signs it with `Ed25519` (reading `--key` or `JETPACK_SIGNING_KEY`), writes `ed25519_signature` atomically, and immediately re-verifies the file from disk:

```bash
jetpack-daemon --sign-catalog crates/jetpack-daemon/assets/catalog.signed.json --key ~/.secrets/jetpack-root-signing.key
```

## Runtime, Inspection & Diagnostic Flags

| Flag | Purpose |
| :--- | :--- |
| `--version, -V` | Print the compiled `jetpack-daemon` semantic version and exit cleanly. |
| `--headless` | Run `jetpack-daemon` without a desktop system tray icon (CI / server). |
| `--seed-core-apps` | Package and pre-install curated core workers from `examples/` (e.g., slim release bundle `examples/ffmpeg-media-converter`, excluding `.venv`, `__pycache__`, `tests`, and media files) into CAS and `<root>/apps/`. |
| `--export-diagnostics [--out <path>]` | Collect a sanitized `.zip` diagnostic bundle (`diagnostics.json` + redacted `daemon.log` tail) and write it to `<path>` or `<root>/logs/jetpack-diagnostics-<timestamp>.zip`. |
| `--print-hardware-profile` | Probe CPU topology, RAM/unified memory, and GPUs (`NVML`, `Vulkan`, `DX12`, `Metal`). |
| `--list-paired-clients` | List approved origins and client UUIDs stored in `paired_clients.db`. |
| `--list-installed-apps` | Scan `<root>/apps/` and print discovered manifests and versions. |
| `--register-uri-scheme` | Register `jetpack://` and `corenode://` custom URL schemes in user space. |
| `--generate-test-pair-uri` | Print a deterministic test `jetpack://pair` URI and SHA-256 fingerprint. |
| `--test-ws-client` | Connect to a running daemon (`127.0.0.1:9470`) to verify PNA, handshake, and `system.echo` (supports `--send-json` and `--origin`). |
| `--ws-smoke-test` | Run offline end-to-end WebSocket security and `hardware.get_profile` smoke test. |
| `--oci-smoke-test` | Run offline OCI CAS, SHA-256 verification, and hardlink/APFS clone smoke test. |
| `--python-smoke-test` | Run offline hermetic `uv` + `python-build-standalone` provisioner smoke test. |
| `--sandbox-smoke-test` | Run offline OS native kernel sandbox (`AppContainer` / `Seatbelt` / `bwrap+Landlock`) smoke test. |
| `--worker-ipc-smoke-test` | Run offline sandboxed worker IPC, userspace proxy, and lifecycle cleanup smoke test. |
| `--full-e2e-smoke-test` | Run the full end-to-end orchestrator, sandbox, GPU worker, and WebSocket integration test. |
| `--background` | Hidden internal flag omitted from `--help`; passed by OS autostart entries (`LaunchAgents`, `HKCU\...\Run`, XDG `.desktop`) to suppress auto-opening the Control Center window at OS login. |
| `--cleanup-old-binary <path>` | Hidden internal post-update flag spawned by the self-updater; validates a `.old` suffix and removes the previous binary or `.app.old` bundle asynchronously (`10 × 200ms` retries). |
