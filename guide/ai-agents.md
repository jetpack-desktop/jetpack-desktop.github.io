---
url: https://jetpack-desktop.github.io/guide/ai-agents.md
description: >-
  Connect AI coding agents via llms-full.txt, the local jetpack mcp stdio
  server, AGENTS.md scaffolding, built-in Agent Skills, and jetpack codegen.
---

# AI Agents & MCP Integration

JetPack includes a complete AI-Agent Developer Experience (DX) suite designed so coding agents (**Claude Code**, **Cursor**, **Windsurf**, **OpenClaw**, and custom MCP clients) can scaffold, type-check, validate, and debug `@jetpack-desktop/client` + Python Worker applications with zero hallucinations.

## 1. LLM-Friendly Documentation (`llms.txt` & `llms-full.txt`)

Every JetPack documentation build publishes clean, HTML-free Markdown entry points at the root of the static site:

* **`https://jetpack-desktop.github.io/llms.txt`** — Structured index of all Guide, Security Audit, Frontend SDK, Worker SDK, and Publishing & CLI pages with 1-line summaries.
* **`https://jetpack-desktop.github.io/llms-full.txt`** — Single concatenated Markdown document containing the entire JetPack documentation set.

### Linking Docs in AI IDEs

* **Cursor**: Open **Settings → Features → Docs → Add new doc** and enter `https://jetpack-desktop.github.io/llms-full.txt`, then reference `@JetPack` in Composer/Chat.
* **Claude Code / OpenClaw**: Fetch or reference `https://jetpack-desktop.github.io/llms.txt` to discover relevant pages, or attach `llms-full.txt` directly when working offline.
* **Offline / Local via MCP**: Use the `search_docs` tool inside `jetpack mcp` (which embeds version-matched `docs/` Markdown directly inside the `jetpack` CLI binary and supports overriding with `JETPACK_DOCS_DIR`).

## 2. Built-In Local MCP Server (`jetpack mcp`)

The standalone console CLI `jetpack` (`crates/jetpack-cli`) includes a Model Context Protocol server powered by the official `rmcp` crate over `stdio` transport (`stdin`/`stdout` JSON-RPC, with all diagnostic tracing routed strictly to `stderr`).

### Client Configuration (`mcpServers`)

Every official JetPack release bundle includes the `jetpack` CLI alongside `jetpack-daemon`, so you can point your MCP client directly at the installed binary without building from source.

Add the appropriate configuration for your OS to **Claude Code** (`.mcp.json` or `~/.claude.json`), **Cursor** (`.cursor/mcp.json` or `~/.cursor/mcp.json`), **Windsurf** (`~/.codeium/windsurf/mcp_config.json`), or any MCP-compatible IDE:

#### macOS (`darwin/arm64`)

When `JetPack.app` is installed in `/Applications`:

```json
{
  "mcpServers": {
    "jetpack": {
      "command": "/Applications/JetPack.app/Contents/MacOS/jetpack",
      "args": ["mcp"]
    }
  }
}
```

*(If you placed `JetPack.app` in `~/Applications`, use `/Users/<you>/Applications/JetPack.app/Contents/MacOS/jetpack`.)*

#### Windows (`windows/amd64`)

When `JetPack-windows-x64.zip` is extracted into `%LOCALAPPDATA%\JetPack\`:

```json
{
  "mcpServers": {
    "jetpack": {
      "command": "C:\\Users\\<you>\\AppData\\Local\\JetPack\\jetpack.exe",
      "args": ["mcp"]
    }
  }
}
```

#### Linux (`linux/amd64`)

When `JetPack-linux-x64.tar.gz` is extracted into `~/.local/opt/jetpack/`:

```json
{
  "mcpServers": {
    "jetpack": {
      "command": "/home/<you>/.local/opt/jetpack/jetpack",
      "args": ["mcp"]
    }
  }
}
```

#### Using `PATH` or Building from Source

If `jetpack` is on your shell `PATH` (for example via `cargo install --path crates/jetpack-cli` or a symlink in `/usr/local/bin` / `~/.local/bin`), or if you are testing an unreleased local build (`target/release/jetpack`):

```json
{
  "mcpServers": {
    "jetpack": {
      "command": "jetpack",
      "args": ["mcp"]
    }
  }
}
```

> **Note:** Regardless of where `jetpack` is invoked from, `jetpack mcp` automatically discovers and communicates with your active `jetpack-daemon` via `<JETPACK_HOME>/config/pairing.ipc` (including when the daemon was launched from `/Applications/JetPack.app`, `%LOCALAPPDATA%\JetPack\jetpack-daemon.exe`, or `~/.local/opt/jetpack/jetpack-daemon`), and `jetpack install-local` / `jetpack package` automatically use the sibling `jetpack-daemon` binary in the same directory.

### Exposed MCP Tools

| Tool | Parameters | Description |
| :--- | :--- | :--- |
| `search_docs` | `query: string`, `max_results?: number` (`1..20`, default `5`) | Searches embedded version-matched JetPack documentation (or `JETPACK_DOCS_DIR`) by keyword/topic and returns ranked Markdown snippets. |
| `get_daemon_status` | *(none)* | Checks `jetpack-daemon` health, version, bound `127.0.0.1` WebSocket port, reachability, paired origins, and active sandboxed workers via token-authenticated `pairing.ipc`. |
| `read_worker_logs` | `app_id: string`, `lines?: number` (`1..2000`, default `100`) | Tails live or offline `worker.log` / `daemon.log` crash tracebacks and stdout/stderr for `app_id`, automatically redacting sensitive keys, tokens, and user paths. |
| `validate_manifest` | `path: string` | Validates `app.toml` (or a directory containing `app.toml`) against the strict `#[serde(deny_unknown_fields)]` schema, permissions, volumes, runtimes, and `python_env`. |
| `scaffold_app` | `target_dir: string`, `app_id?: string`, `template?: "python" \| "fullstack"` (default `"fullstack"`) | Generates a starter JetPack app with `app.toml`, `worker.py`, optional `@jetpack-desktop/client` TypeScript frontend, `AGENTS.md`, `CLAUDE.md`, `.cursor/rules/jetpack.mdc`, and `.agents/skills/`. |

## 3. Project Scaffolding Rules & Agent Skills (`jetpack init`)

Running `jetpack init` (or `jetpack-daemon --init-app`, or the `scaffold_app` MCP tool) generates a ready-to-run project preloaded with AI guardrails:

```bash
# Python worker starter + AI rules & skills
jetpack init ./workers/my-ocr-worker --template python

# Fullstack TypeScript (@jetpack-desktop/client) + Python worker starter
jetpack init ./apps/my-ai-app --app-id com.dev.my-ai-app --template fullstack
```

### Generated AI Artifacts

* **`AGENTS.md`** — Canonical instructions enforcing `JetPackAppClient.connectAndEnsureApp()`, strict MsgPack metadata vs zero-copy `Uint8Array` payload separation, frontend-to-worker opcode parity, and `app.toml` security rules.
* **`CLAUDE.md`** — Regular-file mirror of `AGENTS.md` (not a symlink, ensuring full compatibility on Windows and OCI layer packaging).
* **`.cursor/rules/jetpack.mdc`** — Cursor MDC rule file auto-attaching JetPack architecture constraints.
* **`.agents/skills/`** — Step-by-step reusable Agent Skills:
  * `.agents/skills/create-jetpack-worker/SKILL.md` — Scaffolding, manifest setup, `jetpack validate`, and local installation.
  * `.agents/skills/add-binary-opcode/SKILL.md` — Adding a Pydantic/typed Python worker handler, regenerating `jetpack-env.d.ts`, and calling it from `@jetpack-desktop/client`.
  * `.agents/skills/debug-pairing/SKILL.md` — Diagnosing `127.0.0.1:9470` WebSocket pairing, `jetpack://pair` OOB approval, and worker launch errors via MCP.

All AI instruction files (`AGENTS.md`, `CLAUDE.md`, `.cursor/`, `.agents/`) and generated `jetpack-env.d.ts` declarations are automatically excluded when packaging OCI bundles via `jetpack package` or `jetpack-daemon --package-app`.

## 4. End-to-End Agentic Workflow with `jetpack codegen`

To eliminate frontend/worker opcode drift and metadata typos, define your Python worker schemas with Pydantic `BaseModel`, `TypedDict`, or `@dataclass`, and optionally annotate handlers with `@opcode` (or `client.opcode(...)`) from `jetpack_sdk`:

```python
from pydantic import BaseModel
from jetpack_sdk import JetPackClient, opcode


class TranscribeRequest(BaseModel):
    language: str = "en"
    diarize: bool = False


class TranscribeResponse(BaseModel):
    text: str
    duration_ms: int


@opcode("audio.transcribe", request=TranscribeRequest, response=TranscribeResponse, binary=True)
def handle_transcribe(client: JetPackClient, meta: TranscribeRequest, payload: bytes):
    return {"text": "ok", "duration_ms": len(payload)}
```

Generate strict TypeScript declarations from the worker AST (without executing Python):

```bash
jetpack codegen --worker worker.py --out src/jetpack-env.d.ts --strict
```

Then pass `JetPackOpcodeMap` as the generic parameter to `JetPackAppClient`:

```typescript
import { JetPackAppClient } from '@jetpack-desktop/client';
import type { JetPackOpcodeMap } from './jetpack-env';

const client = new JetPackAppClient<JetPackOpcodeMap>();
await client.connectAndEnsureApp({ appId: 'com.dev.my-ai-app', daemonPort: 9470 });

// Compile-time checked by tsc: opcode name, meta shape, required Uint8Array, and response type
const reply = await client.request('audio.transcribe', { language: 'en' }, audioBytes);
```

If an agent invents a nonexistent opcode, forgets the required `Uint8Array` payload when `binary: true`, or misspells a metadata field, `tsc --noEmit` fails immediately so the agent can self-correct before runtime.
