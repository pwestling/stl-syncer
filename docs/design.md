# MyMiniFactory Sync — Design Document

*rev 0.2 – 2025‑06‑09*

---

## 1 · Purpose

Ship a lightweight cross‑platform desktop application that authenticates to **MyMiniFactory (MMF)** (and, via plugins, other 3‑D model sites), mirrors a user’s library to local disk, and lets them browse / re‑download assets while preserving state.

---

## 2 · Tech Stack

| Layer          | Choice                                       | Notes                                                            |
| -------------- | -------------------------------------------- | ---------------------------------------------------------------- |
| Shell          | **Tauri** (Rust backend + system WebView UI) | 5‑10 MB bundles, auto‑update, OS keychain                        |
| UI             | **React 18** + Vite + TypeScript             | Chakra UI; TanStack Table; easy theming                          |
| Async runtime  | tokio 1.x                                    | all network & FS ops awaitable                                   |
| HTTP           | reqwest                                      | HTTP 2, resumable downloads via Range                            |
| Storage        | SQLite (rusqlite)                            | `sync.db` file in app data dir                                   |
| Crypto         | ring                                         | SHA‑256 for file integrity                                       |
| Keychain       | tauri‑plugin‑store                           | Secure token storage                                             |
| **Plugin SDK** | Rust traits + dynamic dispatch               | Enables additional content providers (Cults3D, Printables, etc.) |

---

## 3 · High‑Level Architecture

```
          ┌──────────── UI (React) ────────────┐
          │  Login · Library · Downloads      │
          └────────▲──────────┬────────▲──────┘
                   │ IPC cmd  │ IPC events
                   ▼          ▼
            ┌────────────── Core (Rust) ──────────────┐
            │ AuthSvc │ LibrarySvc │ DLManager │ DB  │
            │ PluginRegistry  │                     │
            └────▲────┴────▲─────┴────▲──────────────┘
                 │        │           │
            keychain   HTTP/GQL     FS+DB
```

* **PluginRegistry** holds content‑provider plugins loaded at runtime.
* Each **Plugin** implements a common trait (`Provider`) with async methods: `authenticate`, `library_page`, `file_meta`, `download_url`.
* Built‑in plugin: **mmf**. Others compiled in via Cargo features or dropped in as `.dll/.so/.dylib`.

---

## 4 · Detailed Components

### 4.1 Authentication Service

* Delegates to the active plugin’s `authenticate` implementation.
* For MMF: OAuth 2 auth‑code flow via popup WebView → intercept redirect `tauri://auth?code=…` → exchange for tokens → store in keychain → auto‑refresh.

### 4.2 Library Service

* Calls `plugin.library_page(page_idx)` until exhausted.
* Upserts each asset into `assets` table (see 4.3).
* Emits `LibrarySynced` IPC with delta counts.

### 4.3 Local DB Schema (simplified)

```sql
assets(
  id TEXT PRIMARY KEY,
  provider TEXT,          -- e.g. "mmf" or "cults"
  title TEXT,
  creator TEXT,
  updated_at TEXT,
  thumb_url TEXT,
  wanted BOOLEAN DEFAULT 1
);
files(
  id TEXT PRIMARY KEY,
  asset_id TEXT REFERENCES assets(id),
  provider TEXT,
  path TEXT,
  sha256 TEXT,
  downloaded_at INTEGER,
  removed BOOLEAN DEFAULT 0
);
settings(key TEXT PRIMARY KEY, value TEXT);
```

### 4.4 Sync Algorithm

1. Plugin pulls remote library list → upsert.
2. For each remote file meta:

   * If no matching `files` row **or** SHA‑256 differs **and** `removed = 0` → enqueue download.
3. Mark paths manually deleted via UI as `removed = 1` (prevents automatic re‑download).
4. Respect per‑asset `wanted` flag for selective sync.

### 4.5 Download Manager

* Configurable concurrency (default = 3).
* Each job: `HEAD` → stream in 4 MiB chunks → verify SHA‑256 → move to final path → update DB.
* Retries with exponential back‑off (max 5 attempts).

### 4.6 File Layout

```
<root>/
  <Provider>/<Creator>/<Model>/
    renders/
    files/<filename>
```

Root defaults to `~/MyMiniFactorySync`; user can change in Settings.

### 4.7 UI Views

1. **Login** – provider picker (MMF, others) + OAuth button + status.
2. **Library** – filter by provider; table with title · creator · provider · status · last‑sync · wanted toggle.
3. **Downloads** – history with filters (Succeeded / Failed / Removed) and provider column.
4. **Explorer** – tree nav (`provider → creator → model`) + preview pane + re‑download.
5. **Settings** – path selector · concurrency slider · auto‑launch · plugin management (enable/disable).

### 4.8 Plugin Architecture

* **Trait Definition**

  ```rust
  #[async_trait]
  pub trait Provider: Send + Sync {
      fn id(&self) -> &'static str; // "mmf"

      async fn authenticate(&self, ctx: &AuthCtx) -> Result<()>;
      async fn library_page(&self, page: u32) -> Result<Vec<AssetMeta>>;
      async fn file_meta(&self, asset_id: &str) -> Result<Vec<FileMeta>>;
      async fn download_url(&self, file_id: &str) -> Result<Url>;
  }
  ```
* **Registration** – macro `register_provider!(MyPlugin)` adds to a global `HashMap`; dynamic plugins loaded via `libloading` when placed in `plugins/` dir.
* **Versioning** – semantic version in `provider.toml`; core checks `api_version` compatibility.
* **Security** – downloaded plugins signed; public key baked into core.

---

## 5 · Non‑functional Requirements

* **Platforms**: Windows 10+, macOS 12+ (intel & arm64), Ubuntu 22+.
* **Bundle size** ≤ 20 MB (core); each plugin ≤ 2 MB.
* **Throughput**: sustain 3 × 10 MB/s without UI freeze.
* **Integrity**: all files SHA‑256 verified.
* **Privacy**: tokens only in keychain; hashed names in logs.

---

## 6 · Build & Release

1. CI (GitHub Actions) – `cargo clippy --all-targets --all-features && cargo test`; `npm run lint test build`.
2. Build matrix (win‑x64, win‑arm64, mac‑x64, mac‑arm64, linux‑x64).
3. `tauri build` with auto‑update channel; plugins built as separate artifacts.

---

## 7 · Future Enhancements

* STL/OBJ preview via WebGL WASM viewer.
* Bandwidth throttling per time window.
* Headless CLI mode (`mmfsync sync --headless`).
* Cloud‑drive awareness (skip OneDrive “online‑only” files).

---

## 8 · Open Questions (as of rev 0.2)

| Topic                     | Decision Needed                                                |
| ------------------------- | -------------------------------------------------------------- |
| MMF API rate‑limits       | Hard‑code 4 req/s until tested                                 |
| Detect remote file change | Rely on SHA‑256 + `updated_at`, or leverage `ETag` if present? |
| Plugin sandboxing         | Allow full FS access or restrict to sandbox dir?               |

