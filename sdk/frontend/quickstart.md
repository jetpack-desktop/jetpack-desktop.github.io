---
url: https://jetpack-desktop.github.io/sdk/frontend/quickstart.md
description: >-
  Connect, auto-pair, provision workers, and exchange zero-copy binary frames
  from browser TypeScript with @jetpack/client.
---

# Frontend Quickstart (`@jetpack/client`)

The `@jetpack/client` TypeScript package (`web/packages/jetpack-client`) provides `JetPackAppClient`, a high-level wrapper over `jetpack-wasm` that automates cryptographic identity management, Out-of-Band `jetpack://pair` onboarding, catalog pulls, worker launch, correlated binary requests, and streaming event subscriptions.

## Installation

Install `@jetpack/client` from NPM in any Vite, TypeScript, or modern ESM web project:

```bash
npm install @jetpack/client
```

`@jetpack/client` **bundles the compiled `jetpack-wasm` WebAssembly binary and JS glue directly inside the package** (`@jetpack/client/wasm`). External applications work out-of-the-box with zero manual WASM copying or path overrides—both `vite dev` (dependency pre-bundling) and `vite build` resolve the bundled `.wasm` module automatically.

If you host the `.wasm` binary on a custom CDN path or supply pre-loaded bindings, you can still pass `wasmModule`, `wasmModulePath`, or `wasmBinaryOrUrl` to `connectAndEnsureApp()`. To access low-level bindings directly, import from the `@jetpack/client/wasm` subpath export.

## 1. Connect, Pair, Pull, and Launch in One Call

```typescript
import { JetPackAppClient, JetPackClientError } from '@jetpack/client';

const client = new JetPackAppClient();

try {
  await client.connectAndEnsureApp({
    appId: 'io.jetpack.media-converter',
    daemonPort: 9470,
    downloadUrl: 'https://github.com/jetpack-desktop/releases/releases/latest',
    controlBar: {
      appName: 'FFmpeg Media Converter',
      warnOnUnsavedWork: () => hasActiveUnsavedWork(),
      stopConfirmTitle: 'Stop Media Converter?',
      stopConfirmMessage: 'Transcoding is active. Stopping the worker will lose progress.',
      onBeforeStop: async () => {
        // Optional extra async cleanup or veto (return false to keep worker running)
        return true;
      },
      onAfterStop: () => {
        console.log('Worker stopped cleanly from top bar');
      },
    },
    onPairingUri: (uri) => {
      // Optional: display an interactive "Approve Pairing in JetPack" button or link
      console.log('Approve pairing URI:', uri);
    },
    onProgress: (progress) => {
      console.log(
        `[${progress.stage ?? 'pull'}] ${progress.percent ?? 0}% - ${progress.message ?? ''}`
      );
    },
  });

  console.log('Worker ready:', client.getAppId());
} catch (err) {
  if (err instanceof JetPackClientError) {
    if (err.code === 'DAEMON_NOT_RUNNING') {
      console.error(`Install or start JetPack Desktop: ${err.downloadUrl}`);
    } else {
      console.error(`JetPack error [${err.code}]: ${err.message}`);
    }
  } else {
    throw err;
  }
}
```

### What `connectAndEnsureApp()` Automates

1. **Loopback Liveness & Cold-Visit Probe (`GET /health`)**: Probes `http://127.0.0.1:<daemonPort>/health` (bounded by `healthProbeTimeoutMs = 2000` by default). If `jetpack-daemon` is not installed or not running, `connectAndEnsureApp()` rejects immediately with `JetPackClientError('DAEMON_NOT_RUNNING', ..., { downloadUrl })` **without** triggering an unhandled `jetpack://pair` deep link, and updates `JetPackControlBar` to `'Error'` with a sanitized HTTPS **"Download JetPack"** link.
2. **WASM Initialization & Identity**: Loads `jetpack-wasm` and restores (or creates) `WasmClientIdentity` (`Ed25519` + `X25519` keys persisted in `localStorage`).
3. **Auto-Pairing (`autoPair: true`) & WebSocket Session**: If `/health` reports `origin_paired: false` (with W3C Private Network Access headers) or the initial `ClientHello` fails because the origin is unpaired, generates `jetpack://pair?client_id=...&pubkey=...&origin=...`, invokes `onPairingUri(uri)`, triggers the deep link (unless `disableAutoDeepLink: true`), and polls until approved or `pairingTimeoutMs` (`60000ms` default) elapses, then completes the encrypted `X25519` / `Ed25519` handshake on `ws://127.0.0.1:9470`.
4. **Install & Provision Check**: Queries `list_apps()`. If `appId` is missing, resolves `ociReference` from options or `get_catalog()` (verified `SignedAppCatalog`) and runs `pull_app()`, forwarding download, extraction, and environment stages (`oci_layer`, `runtime_download`, `runtime_extract`, `uv_bootstrap`, `python_standalone`, `uv_sync`) to `onProgress` (`percent` is `number | null`).
5. **Worker Boot**: Calls `launch_app(appId, resetVolumes, onProgress)` if the worker is not already `Running` (or attaches this WebSocket session to the running worker), forwarding any just-in-time `runtime_*` / `uv_*` / `sandbox_launch` `app.launch.progress` updates to `onProgress`.
6. **SDK Top Control Bar (`JetPackControlBar`)**: In browser environments (unless `controlBar: false`), automatically mounts an isolated Shadow-DOM top bar displaying the JetPack logo, app name, live status pill (`Connecting`, `Running`, `Stopped`, `Error`), `"Control Center"` link, optional `"Download JetPack"` link (on `DAEMON_NOT_RUNNING`), and `"Stop"` action.

## 2. Send Correlated Zero-Copy Binary Requests (`request`)

Send structured metadata (encoded as MsgPack by `jetpack-wasm`) alongside a zero-copy `Uint8Array` payload directly to your ensured worker:

```typescript
const fileBytes = new Uint8Array(await selectedFile.arrayBuffer());

const response = await client.request(
  'media.inspect',
  { filename: selectedFile.name, include_streams: true },
  fileBytes,
  { timeoutMs: 30_000 }
);

console.log('Reply opcode:', response.opcode);
console.log('Correlation ID:', response.correlationId);
const jsonSummary = JSON.parse(new TextDecoder().decode(response.payload));
console.log('Probe result:', jsonSummary);
```

Under the hood, `request()` assigns a monotonic `correlation_id`, encrypts the `HybridFrame` with `ChaCha20-Poly1305`, and resolves once the worker replies with the matching `correlation_id` (or rejects on `app.error` / `system.error` / `REQUEST_TIMEOUT`). Launched workers stay running until explicitly stopped by the user or daemon/session teardown; if the worker is stopped externally (for example from Control Center or the system tray) or exits (`"app.worker.stopped"`), `JetPackControlBar` switches to `'Stopped'` with a `"Relaunch"` button and `request()` transparently wakes the worker via `ensureWorkerRunning()` on your next action.

## 3. Subscribe to Streaming Events (`onStream`) & Clean Up (`close` / `destroy`)

Subscribe to real-time progress frames, telemetry, or unsolicited worker streams by exact opcode or wildcard `'*'`:

```typescript
const unsubscribeProgress = client.onStream('media.transcode.progress', (payload, ctx) => {
  const text = new TextDecoder().decode(payload);
  console.log(`Progress (corr=${ctx.correlationId}):`, text);
});

const unsubscribeAll = client.onStream('*', (payload, ctx) => {
  console.debug(`[stream:${ctx.opcode}] ${payload.byteLength} bytes`);
});

// Trigger a long-running transcode request
const result = await client.request(
  'media.transcode',
  { preset: 'hevc_videotoolbox', crf: 24 },
  new Uint8Array(0)
);

// Clean up when unmounting your UI component
unsubscribeProgress();
unsubscribeAll();
client.close();
// Or call client.destroy() to close the session AND unmount JetPackControlBar
```

`client.close()` shuts down the WebSocket, clears all `onStream` handlers, rejects any pending `request()` promises with `JetPackClientError('CLIENT_CLOSED', ...)`, and transitions `JetPackControlBar` to `'Stopped'`.

## 4. Safe Worker Stop, Standalone `/apps/<app-id>/`, and External `https://` Hosting

`JetPackControlBar` uses `attachShadow({ mode: 'open' })` so host CSS never leaks into the bar or confirmation modal, and Control Bar styles never affect your application:

* **Confirmation Guard**: Clicking **Stop** opens an in-Shadow-DOM confirmation dialog whenever `warnOnUnsavedWork` evaluates to `true` (boolean or predicate callback) or `onBeforeStop` is provided (and `warnOnUnsavedWork` is not `false`).
* **Programmatic Stop**: Call `await client.stopApp()` to send `app.terminate` (`terminate_app(appId)`) without tearing down the bar, keeping it visible with status `'Stopped'`.
* **Standalone MPA/SPA Routes, Dynamic `ui/` Bundles & Window Downloads**: App frontends served by `jetpack-daemon` live under `/apps/<app-id>/` (`http://127.0.0.1:9470/apps/<app-id>/`). When an installed package ships its own `ui/` folder (`<root>/apps/<app-id>/<active_version>/ui/`), `jetpack-daemon` serves that decoupled bundle directly (with canonical boundary checks, MPA `<subpath>/index.html`, and extensionless SPA fallback) before falling back to embedded Control Center assets. By default (`open_in_app_window = true`), Control Center uses `client.open_url(...)` (`"ui.open_url"`) so `jetpack-daemon` opens each app (`/apps/<app-id>/`) and Control Center in its own OS-native `wry` WebView desktop window (`WKWebView` on macOS, `WebView2` on Windows, `WebKitGTK` on Linux) sharing persistent storage in `<root>/webview-data` and supporting both standard HTTP file downloads (into the user's `Downloads` directory) and `blob:` / `data:` `<a download>` exports via a native `rfd` save-file dialog (with transparent default-browser fallback when running headless, or browser tabs when `"Open Dashboard & Apps in Standalone Windows"` is turned off in **My Hardware & Security**), while `jetpack-daemon` multiplexes HTTP static serving, cross-session worker attachment, and reference-counted cleanup on `127.0.0.1:9470`.
* **External `https://` Domains & `CatalogAppEntry.launch_url`**: Third-party web frontends hosted on developer domains (e.g., Vercel, Cloudflare Pages, or a custom `https://` site) connect seamlessly to `http://127.0.0.1:9470/health` and `ws://127.0.0.1:9470` thanks to W3C Private Network Access (`Access-Control-Allow-Private-Network: true`) and canonical `Origin` normalization in `paired_clients.db`. When an `Ed25519`-signed `SignedAppCatalog` entry specifies `launch_url` (an `https://` URL—or loopback `http://` for local development—whose normalized origin is listed in `allowed_origins`), Control Center, the system tray **Launch / Open** menu, and `ui.open_url` open that external URL directly in a dedicated OS-native `wry` window (`WKWebView` with `NSAllowsLocalNetworking` on macOS, `WebView2` on Windows, `WebKitGTK` on Linux) or default browser tab instead of `/apps/<app-id>/`. Both navigation and `window.open` handlers inside `wry` intercept `jetpack://pair` deep links cleanly without error pages.

## 5. Developer Mode Live Log Terminal (`JetPackLogTerminal`)

When Developer Mode is enabled on the local daemon (`developer_mode === true`), `connectAndEnsureApp()` automatically checks `get_settings()`, subscribes via `logs.subscribe`, and mounts an isolated Shadow-DOM bottom drawer (`JetPackLogTerminal`) alongside `JetPackControlBar`:

* **Hidden by default in Standard Mode**: If `developer_mode === false`, the terminal remains unmounted/hidden and never subscribes to `logs.subscribe` (daemon also enforces `DEV_MODE_REQUIRED`).
* **Collapsed Bar + Unread Badge**: Starts collapsed into a slim bottom bar displaying an unread `WARN`/`ERROR` counter badge.
* **Expanded Controls**: Clicking the bar expands live logs with auto-scroll, severity coloring (`INFO`, `WARN`, `ERROR`, `DEBUG`, `TRACE`), source filters (`All`, `Daemon`, `Worker`), `"Clear"`, and `"Copy Logs"` (`"Copied!"` feedback).
* **Opt-out or Custom Config**: Pass `logTerminal: false` to disable auto-mounting or `logTerminal: { title: 'My Worker Logs', maxEntries: 500 }` to customize options, or stream manually via `client.subscribeLogs(entry => ...)` / `client.unsubscribeLogs()`.

## Next Steps

* Inspect all TypeScript interfaces, `JetPackControlBar`, `JetPackLogTerminal`, and low-level `jetpack-wasm` bindings in the [Frontend API Reference](/sdk/frontend/api-reference).
* Review the exact wire layout in the [Binary Framing Protocol](/sdk/frontend/binary-framing).
