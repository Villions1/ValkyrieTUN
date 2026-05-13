# valkyrieTUN

> Modern, security-hardened SSH client for Linux. No telemetry, no cloud sync,
> all data stays on your machine.

valkyrieTUN is a desktop SSH/SFTP client built with Electron + React +
xterm.js. It connects to remote servers over SSH, lets you manage sessions,
transfer files over SFTP, generate and store keys, run scripts, and forward
ports. Credentials are encrypted at rest with a master password (PBKDF2-SHA512,
600 000 iterations) and the workspace can auto-lock on idle.

Repository: <https://github.com/Villions1/ValkyrieTUN>

---

## Features

### Terminal
- Full **xterm.js** emulator: 24-bit color, Unicode 11, ligature-friendly font
  rendering, configurable scrollback (default 5 000 lines)
- **Multiple tabs**, each with its own SSH connection and scrollback
- **Split view** — vertical or horizontal. Right-click any tab to pin it to
  the second pane; the previously active tab is used by default
- **Broadcast mode** — type once, send to every connected terminal
- **In-terminal search** (`Ctrl+F`)
- **Clipboard** — `Ctrl+Shift+C` / `Ctrl+Insert` to copy, `Ctrl+Shift+V` /
  `Shift+Insert` to paste; bare `Ctrl+C` still sends `SIGINT` to the remote
- **Command palette** (`Ctrl+K`) — fuzzy switch between sessions and views
- Local terminal panel (right side) for quick commands on your own machine

### Sessions
- Create, edit, delete, duplicate SSH hosts; organize them into groups
- Store host, port, user, auth method (password / private key / agent),
  jump host, color tag, notes, production flag
- **Recent sessions** + **quick-connect** bar on the home screen
- **Import/export** sessions as JSON; **import `~/.ssh/config`** as sessions
  (Host / HostName / Port / User / IdentityFile / ProxyJump)
- Per-session post-login script, keepalive interval / count, custom env

### SFTP file manager
- Dual-pane (local ↔ remote) with breadcrumbs, parent-up, refresh
- Upload / download with a progress queue
- Right-click: rename, delete, chmod, mkdir, edit-in-place, download
- Inline text editor for small remote files

### Keys
- Generate RSA / ED25519 / ECDSA keys in-app with optional passphrase
- Import existing keys; auto-detect from `~/.ssh/`
- Show public keys, fingerprints; copy to clipboard
- Assign keys per host

### Scripts
- Named script library with descriptions and tags
- `{{VARIABLE}}` placeholders are prompted on run
- One-click run on the active connection; auto post-login per session

### Tunnels
- **Local**, **Remote**, and **Dynamic (SOCKS5)** port forwarding
- Dynamic SOCKS5 supports optional user/pass authentication
- LAN binding (listen on `0.0.0.0`) is disabled by default and gated by an
  explicit setting; otherwise tunnels bind to loopback only
- Visual status, one-click start/stop, optional auto-start on connect

### Settings
- Dark / light theme with accent color picker
- Terminal font, size, cursor style, scrollback length
- Master password for credential encryption
- Idle auto-lock (configurable in minutes, off by default)
- Tunnel LAN binding toggle
- TOFU known-hosts panel: review, accept, or forget pinned host keys

---

## Security

The codebase went through a full pentest-style audit. Highlights:

- **TOFU host-key pinning** — first connection to a host records its SHA-256
  fingerprint. On any subsequent change the connection is refused and a
  prompt asks the user to inspect old vs new fingerprint before accepting
- **Master password gates every privileged IPC handler** (not just the UI).
  Failed unlocks trigger exponential backoff. Master key is wiped from
  process memory on lock
- **PBKDF2-SHA512, 600 000 iterations**, format `pbkdf2$<iter>$<salt>$<hash>`;
  legacy hashes are read transparently and upgraded on next unlock
- **AES-256-GCM** for stored credentials (passwords, passphrases, private
  keys); ciphertext only, never plaintext, leaves the master process
- **Settings allow-list** — only known keys are writable; you can't snipe
  `masterPasswordHash` via `settings:update` anymore
- **SFTP `localPath` allow-list** — uploads/downloads canonicalize the path
  and refuse anything outside the user's home or system temp
- **Strict CSP**, `setWindowOpenHandler` + `will-navigate` guards, no
  permission grants (mic, camera, geolocation)
- **SQLite DB / WAL / SHM** stored at `0600`; exports saved at `0600`
- **Audit log** — host-key trust events recorded to a local table
- Unit tests for the crypto round-trip and path validation are committed in
  `electron/services/__tests__/`

Full report: `SECURITY_AUDIT.md`.

---

## Tech stack

| Layer    | Tech                                                            |
| -------- | --------------------------------------------------------------- |
| Frontend | React 18 + TypeScript + TailwindCSS                             |
| Terminal | xterm.js + fit / search / web-links / unicode11 addons          |
| SSH      | `ssh2` (Node.js)                                                |
| SFTP     | `ssh2` SFTP subsystem                                           |
| DB       | `better-sqlite3` (WAL mode, `0600` perms)                       |
| State    | Zustand                                                         |
| Icons    | Lucide React                                                    |
| Build    | Vite + `vite-plugin-electron` + `electron-builder`              |
| Targets  | `.AppImage` (recommended), `.deb`, `.rpm`, `.tar.gz`, x64 + arm64 |

---

## Build

### Prerequisites

- **Node.js** ≥ 18 (LTS)
- **npm** ≥ 9
- **Python 3** with `setuptools` (required by `node-gyp` to build
  `better-sqlite3` and `node-pty`; Python ≥ 3.12 ships without `distutils`)
- **gcc / g++**, **make**

#### Arch / CachyOS

```bash
sudo pacman -S nodejs npm python python-setuptools gcc make
```

#### Debian / Ubuntu

```bash
sudo apt update
sudo apt install -y nodejs npm python3 python3-setuptools build-essential
# If the distro Node is older than 18:
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

#### Fedora

```bash
sudo dnf install -y nodejs npm python3 python3-setuptools gcc-c++ make
```

### Build it

```bash
git clone https://github.com/Villions1/ValkyrieTUN.git
cd ValkyrieTUN
npm install
npm run build:linux
```

Output goes to `release/`.

### Build only an AppImage

Either pass the target on the command line:

```bash
npx electron-builder --linux AppImage
```

or edit `electron-builder.yml` and leave only `AppImage` under `linux.target`.

### Run the AppImage

```bash
chmod +x release/valkyrieTUN-*.AppImage
./release/valkyrieTUN-*.AppImage
```

---

## Development

```bash
npm install          # one-time
npm run dev          # Vite + Electron with hot reload
npm run typecheck    # tsc --noEmit
npm run lint         # eslint . --max-warnings 0
```

`npm install` auto-rebuilds `better-sqlite3` and `node-pty` against the
Electron ABI via the `postinstall` hook. If you hit `NODE_MODULE_VERSION`
mismatch, run `npm run rebuild` manually.

### Tests

```bash
node --test electron/services/__tests__/crypto.test.mjs
```

(Six tests — PBKDF2 new+legacy format round-trip, `safeLocalPath` accept/deny
including `..` traversal and look-alike prefix attacks.)

---

## Project structure

```
ValkyrieTUN/
├── electron/                       # Electron main process
│   ├── main.ts                     # App entry, IPC handlers, window setup
│   ├── preload.ts                  # Context bridge
│   └── services/
│       ├── ssh.ts                  # SSH connection management (ssh2)
│       ├── sftp.ts                 # SFTP operations
│       ├── database.ts             # SQLite (better-sqlite3, WAL, 0600)
│       ├── keyManager.ts           # SSH key generation & import
│       ├── tunnelManager.ts        # Local / remote / SOCKS5 forwarding
│       ├── scriptRunner.ts         # Script execution
│       ├── crypto.ts               # AES-GCM + PBKDF2-SHA512 (600k)
│       ├── security.ts             # Allow-lists, path validation, redaction
│       ├── knownHosts.ts           # TOFU host-key store and verifier
│       └── __tests__/              # node:test unit tests
├── src/                            # React renderer
│   ├── App.tsx
│   ├── main.tsx
│   ├── index.css                   # TailwindCSS + custom styles
│   ├── components/
│   │   ├── auth/HostKeyPrompt.tsx  # TOFU mismatch modal
│   │   ├── CommandPalette.tsx      # Ctrl+K fuzzy palette
│   │   ├── layout/                 # Sidebar, TitleBar
│   │   ├── sessions/               # HomeView, SessionList, SessionEditor
│   │   ├── terminal/               # TerminalView, TerminalPane, Local panel
│   │   ├── sftp/                   # FileManagerView (dual-pane)
│   │   ├── keys/                   # KeyManagerView
│   │   ├── scripts/                # ScriptLibraryView
│   │   ├── tunnels/                # TunnelManagerView
│   │   └── settings/               # SettingsView (incl. known-hosts panel)
│   ├── store/                      # Zustand stores
│   ├── types/                      # TypeScript types
│   └── lib/                        # Renderer-side API bridge
├── electron-builder.yml            # Build targets
├── package.json
├── SECURITY_AUDIT.md               # Full pentest report + fix log
└── README.md
```

---

## Keyboard shortcuts

| Shortcut                          | Action                                |
| --------------------------------- | ------------------------------------- |
| `Ctrl+K`                          | Open command palette                  |
| `Ctrl+F`                          | Search inside terminal                |
| `Ctrl+Shift+C` / `Ctrl+Insert`    | Copy terminal selection               |
| `Ctrl+Shift+V` / `Shift+Insert`   | Paste into terminal                   |
| `Ctrl+C` (no Shift)               | Send `SIGINT` to remote shell         |
| Right-click on a tab              | Pin that tab to the second split pane |
| Right-click in the terminal area  | Copy selection, or paste if no selection |

---

## Data storage

Everything is local, single SQLite file in the Electron `userData` directory:

```
~/.config/valkyrieTUN/valkyrie-tun.db
~/.config/valkyrieTUN/valkyrie-tun.db-wal
~/.config/valkyrieTUN/valkyrie-tun.db-shm
```

All three are mode `0600`. Tables: `sessions`, `groups`, `keys`, `scripts`,
`tunnels`, `settings`, `known_hosts`, `audit_log`. Credentials are encrypted
with AES-256-GCM under a key derived from the master password.

Hot backup (while the app is running):

```bash
sqlite3 ~/.config/valkyrieTUN/valkyrie-tun.db \
  ".backup '/tmp/valkyrie-backup.db'"
```

Cold backup (app closed): just `cp` all three files somewhere safe.

No data is ever sent to any server.

---

## License

MIT
