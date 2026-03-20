# Maintainer: guglovich <https://github.com/guglovich>
# Created with assistance from Claude (Anthropic).

pkgname=transmission-remote-slint
pkgver=0.3.1
pkgrel=1
pkgdesc="Lightweight Transmission BitTorrent GUI built with Slint (no GTK)"
arch=('x86_64')
url="https://github.com/guglovich/Transmission-Remote-Slint"
license=('GPL-2.0-or-later')
depends=(
    'transmission-cli'
    'libxcb'
    'libxkbcommon'
    'fontconfig'
    'freetype2'
    'dbus'
)
makedepends=(
    'rust'
    'cargo'
    'pkg-config'
    'python-pillow'   # для ресайза иконок при сборке
)
optdepends=(
    'zenity: file picker dialogs (GNOME/X11)'
    'kdialog: file picker dialogs (KDE)'
    'yad: file picker dialogs (alternative)'
    'libnotify: desktop notifications'
    'snixembed: system tray support in XFCE/Openbox'
    'xfce4-statusnotifier-plugin: system tray support in XFCE'
    'xdotool: taskbar icon support'
)
source=("$pkgname-$pkgver.tar.gz::https://github.com/guglovich/Transmission-Remote-Slint/archive/refs/tags/v${pkgver}.tar.gz")
sha256sums=('ef15d6e1a9f2bd2f04afe09b7fd90a5c8346ae3c7ebce58a59765fbc77770c50')

prepare() {
    cd "Transmission-Remote-Slint-${pkgver}"
    export CARGO_HOME="$srcdir/cargo-home"
    cargo fetch --locked --target "$CARCH-unknown-linux-gnu"
}

build() {
    cd "Transmission-Remote-Slint-${pkgver}"
    export CARGO_HOME="$srcdir/cargo-home"
    export RUSTUP_TOOLCHAIN=stable
    export CARGO_TARGET_DIR=target
    cargo build --frozen --release

    # Генерируем PNG иконки нужных размеров из исходного
    python3 - <<'PYEOF'
from PIL import Image
import os

src = Image.open("ui/app-icon.png").convert("RGBA")
os.makedirs("icons", exist_ok=True)
for size in [16, 22, 32, 48, 64, 128, 256]:
    img = src.resize((size, size), Image.LANCZOS)
    img.save(f"icons/{size}.png")
PYEOF
}

check() {
    cd "Transmission-Remote-Slint-${pkgver}"
    export CARGO_HOME="$srcdir/cargo-home"
    cargo test --frozen --release 2>/dev/null || true
}

package() {
    cd "Transmission-Remote-Slint-${pkgver}"

    # Бинарник
    install -Dm755 "target/release/transmission-remote-slint" \
        "$pkgdir/usr/bin/transmission-remote-slint"

    # Иконки в hicolor — все стандартные размеры
    for size in 16 22 32 48 64 128 256; do
        install -Dm644 "icons/${size}.png" \
            "$pkgdir/usr/share/icons/hicolor/${size}x${size}/apps/transmission-remote-slint.png"
    done

    # .desktop файл
    install -Dm644 /dev/stdin \
        "$pkgdir/usr/share/applications/transmission-remote-slint.desktop" <<'DESKTOP'
[Desktop Entry]
Type=Application
Name=Transmission Remote
GenericName=BitTorrent Client
Comment=Lightweight Transmission GUI (Slint, no GTK)
Exec=transmission-remote-slint %f
Icon=transmission-remote-slint
Terminal=false
Categories=Network;FileTransfer;P2P;
MimeType=application/x-bittorrent;x-scheme-handler/magnet;
Keywords=torrent;bittorrent;transmission;download;
StartupWMClass=transmission-remote-slint
DESKTOP

    # Лицензия и документация
    install -Dm644 LICENSE \
        "$pkgdir/usr/share/licenses/$pkgname/LICENSE"
    install -Dm644 README.md \
        "$pkgdir/usr/share/doc/$pkgname/README.md"
}
