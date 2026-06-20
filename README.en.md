# EngiFTP

[简体中文](README.md) | **English**

**EngiFTP** is a full-featured FTP server written in Rust, with a built-in web admin console, user management, fine-grained permissions, and S3 object storage support.
The FTP protocol is implemented from scratch on the tokio async runtime; the web backend is built on axum.

## Features

### FTP Protocol (RFC 959 / 2389 / 2428 / 3659)

- **Authentication**: USER / PASS / REIN / ACCT, with optional anonymous login (toggleable, forced read-only); auto-disconnect after 3 consecutive wrong passwords
- **Transfer modes**: passive (PASV / EPSV) and active (PORT / EPRT), with IPv6 extension commands; ASCII and binary types (TYPE A / I)
- **File operations**: RETR / STOR / APPE / STOU (unique filename) / DELE / RNFR+RNTO / SIZE / MDTM
- **Resume**: REST + RETR / STOR
- **Directory operations**: CWD / CDUP / PWD / MKD / RMD, plus X-prefixed compatibility commands (XPWD, etc.)
- **Listing**: LIST (unix ls style) / NLST / MLSD / MLST (machine-readable, with perm facts) / STAT
- **Others**: FEAT / OPTS UTF8 / SYST / HELP / NOOP / ABOR / MODE / STRU / ALLO / SITE CHMOD
- **Multi-encoding**: filenames are always stored as UTF-8 on disk; filenames sent by legacy / old Windows clients in GBK and other encodings are transcoded automatically.
  Server-level encoding is configurable (auto-detect / UTF-8 / GBK / GB18030 / Big5 / Shift_JIS / EUC-KR / Windows-1252),
  and a specific encoding can be **forced** per user in User Management (highest priority, overriding auto-detect and OPTS UTF8 to avoid mis-detection).
  Default "auto": UTF-8 first, invalid UTF-8 bytes decoded as GB18030 (a GBK superset), with replies and listings echoed in the same encoding;
  once a modern client sends `OPTS UTF8 ON`, that session is fixed to UTF-8.
- **Security**: each user is confined to their own home directory (virtual root jail, `..` cannot escape); idle timeout; max connection limit; per-command line length capped at 8 KB
- **FTPS (explicit TLS, RFC 4217)**: AUTH TLS / PBSZ / PROT supported, **plain and encrypted modes available at the same time**, both control and data connections can be encrypted; on first enable a self-signed certificate is generated in the data directory (cert.pem/key.pem, private key 0600), replaceable with a real certificate; toggleable in Settings

### Storage Management

- **Storage pools**: two types — `local` (local disk directory) and `s3` (**direct-API** object storage,
  with a self-implemented SigV4 signer, zero external dependencies, compatible with AWS S3 / MinIO / Alibaba Cloud OSS, etc.)
- **No local filesystem mount**: FTP operations are translated directly into object-storage APIs — listing uses ListObjectsV2
  with **streaming pagination** (replies 150 to open the data connection first, then sends pages as they are fetched, so even huge directories never time out the client),
  download uses GetObject (Range supports resume), upload uses a single PUT for small files and 8 MB multipart for large files (bounded memory), directories are expressed via marker objects,
  rename is Copy+Delete (recursive for directories)
- Each user can be assigned a storage pool; home directory = pool path (or bucket prefix) / sub-directory in pool
  (blank defaults to the username; `/` uses the **pool root** directly, so multiple users can share the pool root);
  if no pool is selected, a custom path is used directly (backward compatible)
- **Storage statistics**: the overview shows total file count and storage used; the storage page shows per-pool file count, size, and stat time,
  with a manual "Refresh Stats"; S3 pools use streaming traversal (O(1) memory), so massive object counts use no extra memory; stats run asynchronously in the background without blocking the UI
- S3 pool config: bucket (optionally with `:/prefix`), Region, custom Endpoint, AK/SK
  (the admin API never returns the Secret), one-click connection test in the web UI.
  **For MinIO-compatible storage, just fill in the username/password** (i.e. AccessKey/SecretKey),
  set Endpoint to e.g. `http://<minio-host>:9000`; full end-to-end tests pass against real MinIO
  (`tests/test_minio.py`, with address and credentials given via environment variables)
- **Credential pass-through**: an s3 pool can enable "use FTP login credentials as S3 AK/SK" — each user logs in with their own
  object-storage keys (username = AccessKey, password = SecretKey), validated by a minimal S3 request at login; permissions are controlled by the object-storage side, and the server stores no keys
- **Directory cache & background scan**: S3 directory listings and metadata have a TTL cache (default 60 s, configurable in Settings, 0 = off).
  A background task pre-warms pool roots, periodically refreshes hot directories (recently accessed), and progressively crawls sub-directories within a budget;
  when caching a parent listing it also pre-fills child metadata, so CWD/SIZE return with zero requests — in production, root listing and CWD dropped from 11 s to milliseconds. Data freshness has three guarantees: this server's writes **immediately invalidate** the relevant cache (including all ancestor directories), cache hits trigger an async background refresh (stale-while-revalidate), and data written to S3 directly by other systems becomes visible within at most one TTL
- Known S3 limitations: APPE append and REST resume upload are not supported (object storage cannot write in place) and return a clear 504; SITE CHMOD only applies to local pools; a forcibly interrupted session may leave incomplete multipart uploads — configuring an AbortIncompleteMultipartUpload lifecycle rule on the bucket is recommended

### Web Admin Console (default http://127.0.0.1:8080)

- **Overview**: online sessions, user count, uptime, service config summary (auto-refresh every 5 s)
- **User Management**: add / edit / delete / enable-disable, per-user home directory, 6 permission toggles (list, download, upload, delete, rename, make dir), with assignable storage pool and session encoding
- **Storage**: storage pool list (type, path, status, number of users), create/edit/delete, S3 connection test
- **Sessions**: live view of client address, logged-in user, current directory, last command, upload/download traffic; one-click kick
- **Settings**: ports, passive port range, PASV public address (NAT), max connections, idle timeout, welcome message, anonymous toggle — saved online
- **License Management**: online serial activation / offline activation for air-gapped devices (import `license.dat`), shows machine code and license status, Free ↔ Enterprise switching (see "License" below)
- **Logs**: view the last 500 log entries online
- Admin and user passwords are stored as salted SHA-256 hashes; login sessions use HttpOnly cookies with a 12-hour validity

## Quick Start

```bash
cargo build --release
./target/release/engiftp            # default data dir ./data
./target/release/engiftp /etc/ftpd  # or a specific data dir
```

### Linux Deployment (one-command systemd install)

```bash
./build-linux.sh                    # cross-compile on macOS / any dev machine
                                    # output: dist/engiftp-linux-amd64 / engiftp-linux-arm64
scp dist/engiftp-linux-amd64 server:/tmp/engiftp
ssh server 'sudo /tmp/engiftp install'   # copies to /usr/local/bin, generates and enables the systemd service
```

The `install` subcommand will: create the data directory (default `/var/lib/engiftp`, override with `install /custom/path`),
install the binary to `/usr/local/bin/engiftp`, write `/etc/systemd/system/engiftp.service`
(with `Restart=on-failure`, `LimitNOFILE=65536`), and `systemctl enable --now` for auto-start on boot.
**Online update**: once a new build is ready, update in one command (stop service → back up old version → replace → restart;
if the new version fails to start it **automatically rolls back** to the old one and restores the service):

```bash
scp dist/engiftp-linux-amd64 server:/tmp/engiftp-new
ssh server 'sudo /tmp/engiftp-new update'      # the new binary installs itself
# or: sudo engiftp update /tmp/engiftp-new      # run by the already-installed program
```

Uninstall: `sudo engiftp uninstall` (keeps the data directory).

Cross-compiled to a musl static binary with no runtime dependencies, compatible with all Linux distributions;
required tools: `brew install zig cargo-zigbuild` (or prepare them per build-linux.sh).

On first launch, a default config is generated and the initial admin account is printed:

```
Web console: http://127.0.0.1:8080
Admin: admin  Initial password: admin123   ← log in and change it immediately
```

Then create FTP users in the web UI and connect with any FTP client:

```bash
ftp 127.0.0.1 2121
# or
curl -T file.txt ftp://127.0.0.1:2121/ --user alice:password
```

## Cross-platform Build

```bash
./build-all.sh     # build five targets at once into dist/ (with SHA256SUMS.txt)
```

Targets: Linux amd64 / arm64 (musl static), macOS amd64 / arm64, Windows amd64.
Required tools: `brew install zig cargo-zigbuild`.

## Default Configuration

| Item | Default | Notes |
|---|---|---|
| FTP port | 2121 | listens on 0.0.0.0 (port 21 requires root) |
| Passive port range | 50000–50999 | must be opened in firewall/NAT; caps concurrent transfers |
| Web admin port | 8080 | listens on 0.0.0.0, directly accessible on the LAN |
| Max connections | 2000 | 1000+ live sessions measured at ~26 MB RAM (25 KB/session) |
| User home directory | ./ftp_root/<username> | customizable when creating a user |
| Idle timeout | 600 s | control connection disconnects if no command |

On startup the process file-descriptor limit is raised automatically (stepping up to 65536) to support thousands of concurrent connections;
passive ports are allocated round-robin to avoid linear-scan overhead under high concurrency.

All configuration is stored in `<data-dir>/config.json`, user data in `<data-dir>/users.json`;
these files can also be edited directly and reloaded by restarting. Listen-address / port changes require a process restart; all other settings take effect immediately.
User and storage-pool changes also take effect in real time for **already-connected sessions**: a hot reload happens before the next command
(permissions/encoding tighten immediately, storage-pool switch swaps the backend immediately, disabling or deleting an account disconnects it immediately).

## Security

Audited and hardened systematically. Highlights:

- **Password hashing**: user and admin passwords use **Argon2id** (resistant to GPU brute force); legacy formats are verified for compatibility and upgraded on password change
- **Credential files**: `config.json` / `users.json` / `pools.json` are set to `0600`, the data directory to `0700` (existing files are tightened on startup too)
- **FTP protocol**: PORT/EPRT force the target IP == control-connection peer (anti FTP bounce); passive data connections verify the source IP (anti port hijack); unauthenticated connections have a 30 s auth timeout (anti slowloris)
- **Path safety**: the local backend does a canonicalize boundary check on every access plus `O_NOFOLLOW` on writes, blocking symlink escapes
- **Password-change revocation**: changing the admin password revokes all login tokens and forces re-login
- **SSRF**: S3 endpoints always reject cloud metadata endpoints (169.254.169.254, etc.); set `ENGIFTP_BLOCK_PRIVATE_S3=1` to further block all intranet/loopback addresses (allowed by default for intranet MinIO compatibility)
- **Forced password change**: while still using the factory default `admin/admin123`, the admin is prompted to change it immediately after login, then forced to re-login
- **FTPS**: FTP supports explicit TLS encryption (see above) for sensitive data; the web console has no built-in HTTPS

**Deployment note**: the admin console listens on `0.0.0.0:8080` by default (for remote management of headless servers). If exposed to the public internet, put it behind a reverse proxy with HTTPS or restrict source IPs with a firewall; change the admin password immediately after first launch as prompted.

## Deployment Tips

- **Public internet**: the web console listens on 0.0.0.0 by default — before exposing it, change to 127.0.0.1 (`web_listen` in Settings or config.json) and access via SSH tunnel or a reverse proxy (with HTTPS)
- **Behind NAT**: set "PASV public address" to the public IP in Settings, and open the passive port range
- For sensitive plaintext FTP scenarios, enable **FTPS** (toggle in Settings, explicit TLS) or use a VPN / intranet

## License

**Free for personal use** (Free Edition); commercial, for-profit, or organizational production use requires an **Enterprise license** —
contact **service@xnet.ltd** for a serial number.

### Activating the Enterprise Edition

Use **Settings → License Management** in the web console. The page shows this machine's **machine code**
(randomly generated, persisted in the data directory as `device_id`, the license is bound to it).

- **Online activation**: enter the serial number and click "Activate". The client first validates the serial format locally, then requests the
  license server, which verifies it and issues a license token bound to the machine code, saved locally to `license.json` in the data directory.
  On each subsequent start the token is re-verified offline to restore the Enterprise license — no network needed again.
- **Offline activation (air-gapped devices)**: open the **offline activation page** from the "Offline Activation" panel, enter the serial number
  and machine code, download the signed `license.dat`, and paste its contents back into "Import License File". The client verifies the signature with
  the built-in public key plus device binding, then upgrades to Enterprise.
- After activation, all edition labels (brand badge, overview, login page) show "Enterprise Edition" and the agreement switches to the
  *EngiFTP Enterprise License Agreement*. You can "Deactivate" in License Management to revert to the Free Edition.

> The license integration parameters (product name / product ID / HMAC secret / public key / server URL / offline activation page)
> are compiled into the binary, and can also be overridden with environment variables: `ENGIFTP_LICENSE_SERVER`,
> `ENGIFTP_LICENSE_PRODUCT`, `ENGIFTP_LICENSE_PRODUCT_ID`, `ENGIFTP_LICENSE_SECRET`,
> `ENGIFTP_LICENSE_PUBKEY`, `ENGIFTP_LICENSE_OFFLINE_URL`.

### Telemetry

The program reports a heartbeat **silently** in the background (product ID, version, machine code, OS/architecture), once at startup
and then every 6 hours. It never blocks the main flow; any network/reporting failure is ignored and not logged.
Enabled only when a server URL is configured.

### Multi-language UI

The web console supports Simplified Chinese and English. Use the **EN / 中文** toggle on the login page or in the sidebar footer;
the choice is persisted in the browser (localStorage) and applied automatically on the next visit.

## Testing

`tests/run_tests.py` is the full integration suite (205 cases), which launches the server in various configurations automatically
(with a built-in Mock S3 service for object-storage end-to-end verification), covering:
protocol conformance (all commands, resume, ASCII/binary, Chinese filenames, path-traversal protection),
the permission matrix, network-environment simulation (active/passive, EPSV/EPRT, NAT, IPv6, LAN,
firewall-blocked data connections, slow clients, mid-transfer disconnects, idle timeout, brute force, over-long command lines),
16-way concurrent read/write, passive-port TIME_WAIT reuse, connection limit, multi-encoding transcoding
(automatic GBK client transcoding, OPTS UTF8, per-user forced-encoding priority), storage-pool management
(local pool allocation to disk, in-pool sub-directory boundary protection, disable guard, secret redaction, S3 direct end-to-end:
upload/download/Range resume/20 MB multipart upload/directory semantics/recursive rename/2500-file paginated listing/
credential pass-through login and rejection), all web admin APIs,
plus massive-concurrency scenarios (1050 sessions online at once + 100 concurrent transfers on top, verifying latency,
data integrity, memory usage, and resource reclamation after disconnect).

```bash
cargo build --release && python3 tests/run_tests.py
```

## Project Structure

```
src/
├── main.rs          # entry point: load config, start FTP and web services in parallel
├── config.rs        # server configuration (config.json)
├── users.rs         # user store, password hashing (users.json)
├── license.rs       # serial-number authentication & licensing (license.json, device_id)
├── install.rs       # Linux systemd install/uninstall/update subcommands
├── tls.rs           # FTPS explicit TLS: certificate load / self-sign generation
├── stats.rs         # storage stats: per-pool file count/size (local walk / S3 streaming)
├── storage.rs       # storage-pool definitions, backend construction, cache background scan (pools.json)
├── cache.rs         # S3 directory cache: listing/metadata TTL cache, hot-spot refresh
├── audit.rs         # FTP operation audit log (redb persistence)
├── index.rs         # global file index (redb) for fast listing of large S3 buckets
├── backend.rs       # storage backend abstraction: unified file ops for local disk / S3
├── s3.rs            # S3 REST client: SigV4 signing, paginated listing, multipart upload
├── state.rs         # global shared state: session registry, log ring buffer, web tokens, license
├── ftp/
│   ├── mod.rs       # FTP listener and connection acceptance (connection limit, session registry)
│   └── session.rs   # FTP protocol core: command parsing and all command implementations
└── web/
    ├── mod.rs       # web admin REST API (axum)
    └── index.html   # admin UI (single-page app with i18n, embedded into the binary at compile time)
```
