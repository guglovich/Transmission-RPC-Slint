# Transmission GUI

A native desktop GUI for **Transmission daemon** built with **Rust + Slint**.

## Features

| Feature | Details |
|---|---|
| Torrent list | Name, status, progress bar, ↓/↑ speed |
| Per-torrent control | ▶ Start / ⏸ Stop button on each row |
| Bulk actions | "Start All" / "Stop All" with confirmation |
| Delete | 🗑 Remove (keep files) · 💥 Remove + delete files — both with confirm dialog |
| Auto-refresh | Polls Transmission every **2 seconds** via async tokio |
| Virtual scroll | `ScrollView` over a `VecModel` — handles hundreds of torrents efficiently |
| Architecture | tokio async backend ↔ sync channels ↔ Slint UI thread |

## Prerequisites

### Rust toolchain
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

### System libraries (Linux / Debian/Ubuntu)
```bash
sudo apt install -y \
  build-essential cmake pkg-config \
  libfontconfig1-dev libfreetype-dev \
  libxcb-shape0-dev libxcb-xfixes0-dev libxcb-render0-dev \
  libxkbcommon-dev libwayland-dev
```

### macOS
Xcode Command Line Tools are sufficient — no extra libraries needed.

### Windows
MSVC toolchain (`rustup default stable-x86_64-pc-windows-msvc`). No extra setup.

## Build & Run

```bash
# Clone / unzip the project, then:
cd transmission-gui
cargo run --release
```

First build downloads and compiles ~80 crates (~2-3 min on a modern machine).

## Configuration

Edit the constant at the top of `src/rpc.rs` to change the Transmission address:

```rust
const RPC_URL: &str = "http://localhost:9091/transmission/rpc";
```

If your daemon requires **authentication**, add:

```rust
.basic_auth("username", Some("password"))
```

to the `reqwest` request builder in `TransmissionClient::call()`.

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  UI Thread (Slint event loop)                               │
│  ┌──────────────┐    Timer @20Hz drains channels            │
│  │  MainWindow  │◄── update_rx (Vec<RawTorrent>)            │
│  │  VecModel    │◄── status_rx (String)                     │
│  │  Callbacks   │──► cmd_tx  (Command enum)                 │
│  └──────────────┘                                           │
└──────────────────────────┬──────────────────────────────────┘
                           │  mpsc channels (thread-safe)
┌──────────────────────────▼──────────────────────────────────┐
│  Tokio async runtime (separate OS threads)                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │  backend_task                                          │ │
│  │    tokio::select!                                      │ │
│  │      cmd_rx.recv()  → immediate RPC action             │ │
│  │      interval.tick()→ torrent-get every 2s             │ │
│  └────────────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────┐                            │
│  │  TransmissionClient (rpc.rs)│                            │
│  │  reqwest HTTP + 409 retry   │                            │
│  └─────────────────────────────┘                            │
└─────────────────────────────────────────────────────────────┘
```

**Zero UI logic in .slint** — all business logic lives in Rust.  
Slint only renders; the `VecModel` diff ensures minimal repaints.

## File Structure

```
transmission-gui/
├── Cargo.toml
├── build.rs          ← compiles main.slint
├── ui/
│   └── main.slint    ← all UI layout & styling
└── src/
    ├── main.rs       ← UI wiring, channel pump, Slint model updates
    └── rpc.rs        ← async Transmission JSON-RPC client
```
