# Transmission Remote — Slint

A lightweight native desktop GUI for **Transmission daemon** built with **Rust + Slint**.  
No GTK, no Qt — pure Rust rendering via Skia/OpenGL or Vulkan.

> **Developed with assistance from Claude (Anthropic AI).**

 [Читать на русском](README.ru.md)

---

## UI Performance Comparison

GTK and Qt frontends share a well-known problem with large torrent libraries. Both render the torrent list on the **main UI thread** and rebuild the entire model on every poll cycle. The GTK 4 frontend is especially aggressive: it fires `gtk_list_store_clear()` + re-inserts all rows every few seconds, which causes the GTK main loop to stall completely.

Real-world reports confirm this:

- **GTK 4.1 with ~4,700 torrents** — a single click takes up to a minute; window artifacts appear on top of other applications. ([#8359](https://github.com/transmission/transmission/issues/8359))
- **Qt and GTK with 3,200+ torrents** — searching, opening, or altering a torrent can take all night to complete. ([#4193](https://github.com/transmission/transmission/issues/4193))

The Qt client behaves somewhat better in practice because Qt's `QAbstractItemModel` with `dataChanged` signals is more surgical — it can update individual cells without a full reset. However the underlying issue remains: all polling and model updates still happen on the main thread, and with thousands of active torrents firing rapid updates, the UI event loop gets saturated. Issue #4193 affecting both GTK and Qt was closed as a core regression, not fixed in the frontend.

**This project takes a different approach:**

```
┌──────────────────────────────────────────────────────────┐
│  Slint UI thread (event loop)                            │
│  MainWindow ◄── update_rx (torrents + stats)  50ms pump │
│             ◄── status_rx (status bar text)              │
│             ──► cmd_tx   (Command enum)                  │
└─────────────────────────┬────────────────────────────────┘
                          │  std::sync::mpsc
┌─────────────────────────▼────────────────────────────────┐
│  Tokio async runtime                                     │
│  backend_task: tokio::select!                            │
│    cmd_rx  → immediate RPC call                          │
│    interval tick → recently-active delta every 2s        │
│  TransmissionClient (reqwest, 409 session retry)         │
└──────────────────────────────────────────────────────────┘
```

- **Tokio async runtime** handles all network I/O in a separate thread — the UI never blocks on RPC calls
- **`recently-active` delta updates** — only torrents that changed in the last interval are fetched and pushed to the UI; the full list is never re-rendered unless explicitly requested
- **Slint virtual scrolling** — only visible rows are rendered, regardless of total library size
- The UI thread only receives a small diff via `mpsc` channel and applies it; it never touches the network

The result: the UI stays responsive at 1,000+ or 4,000+ torrents because the main thread simply never does the work that kills GTK and Qt at scale.

| Metric | **transmission-remote-slint** | transmission-remote-gtk | transmission-qt | Transmission GTK 4.1 |
|---|---|---|---|---|
| Type | Remote only | Remote only | Standalone + Remote | Standalone |
| Toolkit | Slint (Rust) | GTK 3 | Qt 5/6 | GTK 4 |
| UI thread blocked on poll? | ✅ Never | ❌ Always | ⚠️ Partially | ❌ Always |
| Update strategy | `recently-active` delta | Full list rebuild | Partial via signals | Full list rebuild |
| Virtual scrolling | ✅ | ❌ | ❌ | ❌ |
| System tray | ✅ Works (SNI/D-Bus) | ✅ Works | ✅ Works | ⚠️ Broken in GTK 4¹ |
| Desktop notifications | ✅ | ✅ | ✅ | ✅ |
| License | GPL-2.0-or-later | GPL-2.0-or-later | GPL-2.0-or-later | GPL-2.0-or-later |

> ¹ GTK 4 dropped tray support. The fix (`feature/gh7364-gtk-sni`) is in development but not yet merged as of early 2026.  

---

## Features

* **Torrent list** — name, status, progress, ↓/↑ speed, error messages inline
* **Per-torrent actions** — Start / Pause / Recheck / Open folder / Remove / Delete with files
* **Bulk actions** — Start All / Stop All with confirmation dialog
* **Disk filter bar** — group and pause/resume torrents by physical disk (detected via `lsblk`)
* **Search** — instant filter by torrent name
* **System tray** — StatusNotifierItem via D-Bus (no GTK, works in KDE/GNOME/XFCE)
* **Desktop notifications** — download complete, recheck done, torrent errors
* **Single instance** — second launch focuses the window or adds a `.torrent` file
* **Auto-detect Transmission** — reads `settings.json`, starts daemon if not running
* **`.torrent` file handler** — pass a file as argument or open from file manager
* **i18n** — Russian and English, configurable in `~/.config/transmission-gui/config.toml`
* **Autostart** — optional `.desktop` entry in `~/.config/autostart/`
* **Render backend** — auto-selects Vulkan → OpenGL → Software; override with `--vk / --gl / --sw`

---

## Installation

### AUR (Arch Linux) — build from source

* AUR: <https://aur.archlinux.org/packages/transmission-remote-slint>

```
paru -S transmission-remote-slint

# Manually
git clone https://aur.archlinux.org/transmission-remote-slint.git
cd transmission-remote-slint
makepkg -si
```

### AUR — prebuilt binary

* AUR: <https://aur.archlinux.org/packages/transmission-remote-slint-bin>

```
paru -S transmission-remote-slint-bin
```

### Build from source

```
# Dependencies (Arch)
sudo pacman -S rust base-devel libxcb libxkbcommon fontconfig freetype2

# Dependencies (Debian/Ubuntu)
sudo apt install -y build-essential cargo pkg-config \
  libfontconfig1-dev libfreetype-dev \
  libxcb-shape0-dev libxcb-xfixes0-dev libxcb-render0-dev \
  libxkbcommon-dev

git clone https://github.com/guglovich/Transmission-Remote-Slint.git
cd Transmission-Remote-Slint
cargo build --release
./target/release/transmission-remote-slint
```

---

## Optional runtime dependencies

| Package | Purpose |
|---|---|
| `zenity` or `kdialog` | File picker dialogs (add/create torrent) |
| `libnotify` | Desktop notifications |
| `snixembed` | Tray support in XFCE / Openbox |
| `xfce4-statusnotifier-plugin` | Tray support in XFCE (alternative) |

---

## Configuration

Config file: `~/.config/transmission-gui/config.toml`  
Created automatically on first launch with defaults:

```
language = "ru"              # "ru" or "en"
suspend_on_hide = false      # freeze process when minimized to tray
start_minimized = false      # start hidden in tray
refresh_interval_secs = 2   # poll interval
delete_torrent_after_add = true
autostart = false
```

Transmission connection is auto-detected from:

* `~/.config/transmission-daemon/settings.json`
* `~/.config/transmission/settings.json`
* `/var/lib/transmission/.config/transmission-daemon/settings.json`
* Fallback: `http://127.0.0.1:9091/transmission/rpc`

---

## Command-line options

```
transmission-remote-slint [FILE.torrent] [--gl|--vk|--sw|--wl]

--gl    Force OpenGL renderer
--vk    Force Vulkan renderer
--sw    Force software renderer (CPU)
--wl    Force Wayland backend
```

---

## Architecture

```
┌──────────────────────────────────────────────────────────┐
│  Slint UI thread (event loop)                            │
│  MainWindow ◄── update_rx (torrents + stats)  50ms pump │
│             ◄── status_rx (status bar text)              │
│             ──► cmd_tx   (Command enum)                  │
└─────────────────────────┬────────────────────────────────┘
                          │  std::sync::mpsc
┌─────────────────────────▼────────────────────────────────┐
│  Tokio async runtime                                     │
│  backend_task: tokio::select!                            │
│    cmd_rx  → immediate RPC call                          │
│    interval tick → recently-active delta every 2s        │
│  TransmissionClient (reqwest, 409 session retry)         │
└──────────────────────────────────────────────────────────┘
```

---

## File structure

```
├── Cargo.toml
├── Cargo.lock
├── build.rs              ← compiles main.slint
├── ui/
│   └── main.slint        ← all UI layout & styling
└── src/
    ├── main.rs           ← UI wiring, timers, model updates
    ├── rpc.rs            ← async Transmission JSON-RPC client
    ├── config.rs         ← reads Transmission settings.json
    ├── app_config.rs     ← application config (~/.config/…)
    ├── daemon.rs         ← auto-start/stop transmission-daemon
    ├── disks.rs          ← physical disk detection via lsblk
    ├── tray.rs           ← StatusNotifierItem tray (ksni)
    ├── notify.rs         ← desktop notifications (notify-rust)
    ├── filepicker.rs     ← zenity/kdialog file dialogs
    ├── single_instance.rs← Unix socket single-instance lock
    ├── suspend.rs        ← SIGSTOP/SIGCONT process suspend
    └── i18n.rs           ← ru/en static strings
```

---

## Roadmap to v1.0

The current v0.x releases are functional and stable for everyday use, but the 1.0 version will bring a significantly redesigned interface. The prototype is already in progress.

Planned for 1.0:

- Full redesign of the main window — torrent cards instead of a plain list, richer per-torrent detail at a glance
- Torrent detail panel (tracker list, file tree, peer list) without leaving the main window
- Magnet link support from command line and browser
- Torrent creation dialog
- Per-torrent bandwidth scheduling

<img width="1625" height="962" alt="изображение" src="https://github.com/user-attachments/assets/b988e481-9150-4053-9433-df95d7c3d4cf" />


> If you'd like to follow progress or leave feedback on the direction, open a Discussion or an Issue.

---

## Releases

### v0.3.0 — first stable release

The first stable release with a minimal but complete feature set designed for a smooth transition from any other Transmission client — especially for users with **1000+ torrents**. The UI stays responsive at any library size thanks to virtual scrolling and delta-only RPC updates (`recently-active`). Core workflow — add, monitor, pause, remove, open folder — works out of the box with zero manual configuration on a standard Arch/Debian setup.

---

## License

GPL-2.0-or-later. See [LICENSE](LICENSE).  
Uses [Slint](https://slint.dev) under GPLv3.

---

## Support My Work

If you find this project useful, consider supporting my open source work:

[![Support](https://img.shields.io/badge/Support-My%20Work-2ea043?logo=github)](https://guglovich.github.io/donate/)

Your donations help fund AI agent subscriptions and development time.
