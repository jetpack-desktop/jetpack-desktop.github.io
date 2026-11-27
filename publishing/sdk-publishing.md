---
url: https://jetpack-desktop.github.io/publishing/sdk-publishing.md
description: >-
  Build, verify in isolated consumers, synchronize versions, and publish
  @jetpack/client (NPM) and jetpack-sdk (PyPI).
---

# SDK Publishing (NPM & PyPI)

Maintainers can build, verify in clean isolated environments, synchronize versions, and publish both official JetPack SDKs using repeatable user-space `Makefile` commands:

* **Frontend SDK (`@jetpack/client`)** — `web/packages/jetpack-client`, bundling compiled `jetpack-wasm` (`crates/jetpack-wasm`) into `wasm/` and re-exporting it via `@jetpack/client/wasm` so external consumers require zero manual WASM path setup.
* **Python Worker SDK (`jetpack-sdk`)** — `sdks/python`, built with `hatchling` (`sdist` + `py3-none-any` wheel) with zero mandatory dependencies (`dependencies = []`) and optional `msgpack` acceleration (`jetpack-sdk[msgpack]`).

See [Frontend Quickstart](/sdk/frontend/quickstart) and [Python Worker SDK](/sdk/worker/python-sdk) for consumer usage.

## 1. Build Both SDKs (`make sdk-build`)

```bash
make sdk-build
```

`make sdk-build` executes in user space:

1. Compiles `crates/jetpack-wasm` via `wasm-pack --target web` (`make wasm`).
2. Copies compiled WASM glue and binaries (`jetpack_wasm.js`, `jetpack_wasm.d.ts`, `jetpack_wasm_bg.wasm`, `jetpack_wasm_bg.wasm.d.ts`) into `web/packages/jetpack-client/wasm/` and compiles TypeScript to `dist/`.
3. Cleans `sdks/python/dist/` and builds fresh `.tar.gz` (`sdist`) and `.whl` (`wheel`) packages via `uv build --directory sdks/python` (or `python3 -m build`).

## 2. Verify Packaging & Isolated Consumers (`make sdk-check`)

```bash
make sdk-check
```

`make sdk-check` runs `make sdk-build` and executes `scripts/sdk_check.sh`, verifying:

1. **Version Parity**: `@jetpack/client` (`package.json`), `jetpack-sdk` (`pyproject.toml` PEP 440), and `EMBEDDED_JETPACK_SDK_VERSION` (`crates/jetpack-daemon/src/python.rs`) match.
2. **TypeScript SDK Unit Tests & Tarball Inspection**: Runs `npm --prefix web/packages/jetpack-client test`, packs the `.tgz`, and verifies `dist/`, `wasm/`, `README.md`, and `LICENSE` are present without leaking `src/`, `tests/`, or `node_modules/`.
3. **Isolated Vite + TypeScript Consumer Smoke Check**: Installs the packed `.tgz` into a fresh temporary Vite + TypeScript project, verifies `tsc --noEmit` and `vite build` emit `jetpack_wasm_bg-*.wasm` into `dist/assets/`, and confirms `vite dev` serves `/node_modules/@jetpack/client/wasm/jetpack_wasm_bg.wasm` with `200 OK` and `application/wasm`.
4. **Python Metadata & Isolated Venv Check**: Runs `twine check sdks/python/dist/*`, inspects `.whl` contents (including `jetpack_sdk/py.typed`), installs the wheel into a clean temporary `uv venv` without host site-packages, and passes `sdks/python/tests`.

## 3. Version Synchronization (`scripts/sync_version.sh`)

Before tagging a release, synchronize all workspace and SDK manifests from a single semantic version or tag:

```bash
bash scripts/sync_version.sh v0.2.0
```

This updates `Cargo.toml`, `crates/jetpack-daemon/assets/Info.plist`, `web/package.json`, `web/packages/jetpack-client/package.json`, `sdks/python/pyproject.toml` (converting SemVer pre-release tags such as `-alpha.1` to PEP 440 `a1`), and `EMBEDDED_JETPACK_SDK_VERSION` in `crates/jetpack-daemon/src/python.rs`.

## 4. Dry-Run & Live Publishing (`make sdk-publish`)

Always run a non-mutating dry run before publishing:

```bash
make sdk-publish DRY_RUN=1
```

`DRY_RUN=1` validates the full `make sdk-check` suite, runs `npm publish ./web/packages/jetpack-client --access public --dry-run`, checks `sdks/python/dist/*` with `twine check`, and skips live uploads (even on a dirty git tree).

When publishing live from a clean git working tree:

| Command | Target Registry | Notes |
| :--- | :--- | :--- |
| `make sdk-publish` | NPM + PyPI | Fails if git tree is dirty unless `ALLOW_DIRTY=1` is set |
| `make sdk-publish-npm` | NPM only (`@jetpack/client`) | Runs `sdk-check` first |
| `make sdk-publish-pypi` | PyPI only (`jetpack-sdk`) | Runs `sdk-check` first |

### Authentication Options

* **NPM (`@jetpack/client`)**: Set `NPM_TOKEN` (or `NODE_AUTH_TOKEN`)—written to an ephemeral `0600` temporary config deleted on exit without modifying `~/.npmrc`—or log in beforehand via `npm login`.
* **PyPI (`jetpack-sdk`)**: Set `UV_PUBLISH_TOKEN` (or `PYPI_API_TOKEN` / `TWINE_PASSWORD`), or rely on `~/.pypirc` / an interactive prompt.

```bash
# Non-interactive CI / token-based publish
NPM_TOKEN=... UV_PUBLISH_TOKEN=... make sdk-publish

# Allow publishing while testing uncommitted local edits (not recommended for releases)
ALLOW_DIRTY=1 make sdk-publish-npm
```

## 5. GitHub Actions Release Integration

`.github/workflows/release.yml` runs `sdk-check` on every tag (`v*`) or manual `workflow_dispatch` run, gating public desktop releases on SDK packaging verification. When `NPM_TOKEN` and/or `PYPI_API_TOKEN` repository secrets are configured, the `publish-sdks` job publishes both packages automatically after release artifacts succeed; if either secret is omitted, the step logs an informational notice and skips cleanly without failing binary or GitHub Pages releases.

See also [CLI Tools Reference](/publishing/cli-tools) and [Store Submission & Catalog Signing](/publishing/store-submission).
