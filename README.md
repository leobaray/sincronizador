# Sincronizador

Folder synchronisation between a Windows desktop and a Linux server, tunnelled over Cloudflare so the server never exposes an inbound port.

## Why

The obvious way to push a folder to a remote machine is to open SSH or a file service and forward a port. On a small office network that means a router change, a static address, and a service listening on the public internet. Sincronizador avoids all of it: the server daemon binds to `127.0.0.1` and a Cloudflare Tunnel publishes it, so the only thing reachable from outside is the tunnel hostname — and the daemon authenticates every request itself, with a password it stores only as a salted PBKDF2 hash.

Transfers move as `tar` streams compressed with **zstd** and piped in chunks, so a folder of many small files costs a handful of batched requests instead of one round-trip per file.

## Features

- **Push and pull** between a local folder and a remote directory, from a desktop GUI or the command line.
- **Differential sync** — a manifest lists remote files with size and mtime; only new or changed files (size mismatch, or mtime beyond a 2 s tolerance) are transferred. This is an *additive* sync: files removed on one side are **not** deleted on the other, so it is not a mirror.
- **Two transfer strategies, chosen automatically.** Up to 3 changed files go as concurrent per-file HTTP transfers (one worker per file, so at most 3 at a time). More than that are streamed as a single `tar` + zstd archive, split into batches of 800 files each — the 50 MB batch cap additionally applies on upload — with per-batch retry and exponential backoff. A 16-connection HTTP pool (`PARALLEL_WORKERS`) is kept ready underneath, but it sizes the connection pool, not the number of concurrent transfers.
- **Streaming compression, no full-buffer in memory** — packing runs in a thread that writes to an `os.pipe`; the HTTP layer reads the other end and streams it out. Multi-threaded zstd (up to 8 threads).
- **Password-authenticated daemon** — PBKDF2-HMAC-SHA256, 600 000 iterations, 32-byte random salt, constant-time compare. Legacy plain-SHA-256 hashes are verified once and transparently upgraded on the next successful login.
- **Brute-force protection** — per-IP rate limit of 5 authentication attempts per 60 s window; sessions are bearer tokens with a 24 h TTL.
- **Path-traversal protection** — every requested path is resolved and confined to the server root (`/sistemas`), and every `tar` member name is rejected if it is absolute or contains a `..` component. The same check runs on both client and server.
- **Upload size cap** — 10 GB ceiling enforced by both a `Content-Length` pre-check (HTTP 413) and a streaming `_LimitedReader` that fails closed rather than filling the disk.
- **Desktop GUI** (CustomTkinter): connection panel, push/pull tabs, live progress with ETA and throughput, a pre-push file-filter dialog to exclude entries, persistent settings, and rotating file logs.
- **Single-file Windows build** via the bundled PyInstaller spec.

## Stack

| Layer | Dependencies |
|-------|--------------|
| Runtime | Python ≥ 3.10 |
| Server | Flask ≥ 2.3 · waitress ≥ 2.1 · zstandard ≥ 0.21 |
| Client | requests ≥ 2.31 · zstandard ≥ 0.21 · CustomTkinter ≥ 5.2 (Tkinter) |
| Packaging | PyInstaller (`Sincronizador.spec`) |
| Dev | pytest ≥ 8 · pytest-cov · ruff ≥ 0.5 |

## Architecture

```
 Windows / desktop client                 Cloudflare              Linux server
 ────────────────────────                 ──────────              ─────────────
 main.py ─► client/desktop.py
              │  (GUI or CLI push/pull)
              ▼
        client/sync_logic.py   manifest diff · batching · retry
              │
              ▼
        client/http.py  ──HTTPS──►  sync.example.com  ──tunnel──►  127.0.0.1:5000
          tar + zstd stream                                          server.py
          bearer token                                                 │  auth + rate limit
                                                                       │  _safe_path guard (/sistemas)
                                                                       │  _LimitedReader (10 GB cap)
                                                                       └─ tar + zstd stream  ⇄  disk
```

The daemon listens only on loopback (`run_daemon(port, bind="127.0.0.1")`); nothing but the tunnel reaches it. HTTP surface: `POST /auth`, `POST /logout`, `GET /manifest`, `GET|POST /file`, `POST|PUT /archive`.

## Getting started

### Server (Linux)

```bash
git clone https://github.com/leobaray/sincronizador.git
cd sincronizador

python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt          # flask, waitress, zstandard

python server.py setpassword             # stores a PBKDF2 hash in config.json
python server.py daemon --port 5000      # binds 127.0.0.1:5000
```

Then point a Cloudflare Tunnel at `127.0.0.1:5000`. The daemon serves paths under `/sistemas` only. To bind an external interface directly (no tunnel), pass `--bind 0.0.0.0`.

### Client (desktop)

```bash
pip install -r client/requirements.txt   # requests, zstandard, customtkinter
cp config.example.json config.json        # set host and folder pairs (optional; the GUI persists them)
python main.py                            # launch the GUI
```

`config.example.json` is the client template: the tunnel hostname plus one folder pair per direction.

```json
{
  "host": "https://sync.example.com",
  "push_local":  "C:/path/to/local/folder",
  "push_remote": "/sistemas/destination",
  "pull_local":  "C:/path/to/local/destination",
  "pull_remote": "/sistemas/source"
}
```

Command-line push/pull (no GUI):

```bash
python main.py push C:/local https://sync.example.com /sistemas/dest --password ****
python main.py pull https://sync.example.com /sistemas/src C:/local    --password ****
# or export SYNC_PASSWORD to omit --password
```

### Windows executable

```bash
pip install pyinstaller
pyinstaller Sincronizador.spec           # produces a single windowed dist/Sincronizador.exe
```

### Tests

```bash
pip install -e ".[client,dev]"
pytest                                   # 37 tests
```

## Project structure

```
server.py              Linux daemon: auth, rate limiting, safe paths, tar+zstd streaming, size caps
main.py                client entry point (delegates to client.desktop:main)
client/
  desktop.py           GUI/CLI dispatch (push · pull · gui)
  gui.py               CustomTkinter interface
  sync_logic.py        manifest diff, batching, parallel vs archive strategy, retries
  http.py              SyncClient: HTTP transport, tar+zstd pack/unpack, auth
  config.py            client config load/save (0600 on Unix)
  constants.py         tunables: workers, batch sizes, chunk size, zstd level
  logger.py            rotating file logs (APPDATA on Windows, ~/.sincronizador on Unix)
  utils.py             formatting + tar-member name safety
tests/                 safe paths, limited reader, password hashing, sync diff/batching, formatters
Sincronizador.spec     PyInstaller build for a single Windows executable
config.example.json    client config template
pyproject.toml         packaging, optional-dependency groups, pytest + ruff config
```

## Status and limitations

Personal project, in active use for moving working folders between a Windows workstation and a Linux server. It syncs configured folder pairs on demand:

- No file watcher — transfers are triggered manually.
- No intra-file delta — a changed file is re-sent whole.
- No conflict resolution and no deletion propagation — it copies new and modified files only, so it is a one-directional additive update per run, not a versioned backup or a true mirror.
- The server root is fixed to `/sistemas` in source; changing it means editing `server.py`.
- The GUI depends on Tkinter/CustomTkinter and is packaged for Windows; the server targets Linux (waitress).

## License

MIT.
