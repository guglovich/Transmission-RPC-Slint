# Transmission Remote — Slint

Лёгкий нативный графический клиент для **Transmission daemon**, написанный на **Rust + Slint**.  
Без GTK, без Qt — рендеринг через Skia/OpenGL или Vulkan.

> **Разработан с помощью Claude (Anthropic AI).**

---

## Сравнение

| Функция | **transmission-remote-slint** | transmission-remote-gtk | transmission-qt | Transmission GTK 4.x |
|---|---|---|---|---|
| Тип | Только удалённый | Только удалённый | Standalone + Remote | Standalone |
| Тулкит | Slint (Rust) | GTK 3 | Qt 5/6 | GTK 4 |
| Системный трей | ✅ Работает (SNI/D-Bus) | ✅ Работает | ✅ Работает | ⚠️ Сломан в GTK 4¹ |
| Уведомления | ✅ | ✅ | ✅ | ✅ |
| ОЗУ (простой) | ~50 МБ | ~80 МБ | ~90 МБ | ~150 МБ |
| Лицензия | GPL-2.0-or-later | GPL-2.0-or-later | GPL-2.0-or-later | GPL-2.0-or-later |

> ¹ GTK 4 убрал поддержку трея. Исправление разрабатывается, но не влито по состоянию на начало 2026 года.

---

## Возможности

- **Список раздач** — имя, статус, прогресс, ↓/↑ скорость, текст ошибки прямо в строке
- **Действия над раздачей** — Старт / Пауза / Перепроверить / Открыть папку / Удалить / Удалить с файлами
- **Массовые операции** — Запустить все / Остановить все с диалогом подтверждения
- **Фильтр по дискам** — группировка и пауза/возобновление раздач по физическому диску (через `lsblk`)
- **Мгновенный поиск** — фильтрация по имени без ожидания RPC
- **Системный трей** — StatusNotifierItem через D-Bus (нативный zbus 4, без ksni/GTK)
- **Уведомления рабочего стола** — завершение загрузки, конец проверки, ошибки раздач
- **Одиночный экземпляр** — повторный запуск поднимает окно или добавляет `.torrent` файл
- **Авто-определение Transmission** — читает `settings.json`, запускает демон если не запущен
- **Открытие `.torrent` файлов** — из файлового менеджера или как аргумент командной строки
- **Мультиязычность** — русский и английский, настраивается в `config.toml`
- **Иконка приложения** — встроена в бинарник, устанавливается в hicolor тему через PKGBUILD
- **Автозапуск** — опциональный `.desktop` файл в `~/.config/autostart/`
- **Бэкенд рендеринга** — автовыбор Vulkan → OpenGL → Программный

---

## Установка

### AUR (Arch Linux) — сборка из исходников

```bash
paru -S transmission-remote-slint
# или вручную:
git clone https://aur.archlinux.org/transmission-remote-slint.git
cd transmission-remote-slint
makepkg -si
```

### AUR — готовый бинарник

```bash
paru -S transmission-remote-slint-bin
```

### Сборка из исходников

```bash
# Arch
sudo pacman -S rust base-devel libxcb libxkbcommon fontconfig freetype2

# Debian/Ubuntu
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

## Опциональные зависимости

| Пакет | Назначение |
|---|---|
| `zenity` или `kdialog` | Диалоги выбора файлов |
| `libnotify` | Уведомления рабочего стола |
| `snixembed` | Поддержка трея в XFCE / Openbox |
| `xfce4-statusnotifier-plugin` | Поддержка трея в XFCE (альтернатива) |
| `xdotool` | Иконка в докбаре через `_NET_WM_ICON` |

---

## Конфигурация

Файл конфигурации: `~/.config/transmission-gui/config.toml`  
Создаётся автоматически при первом запуске:

```toml
language = "ru"                 # "ru" или "en"
suspend_on_hide = false         # заморозить процесс при скрытии в трей
start_minimized = false         # запускать свёрнутым в трей
refresh_interval_secs = 2       # интервал опроса демона
delete_torrent_after_add = true # удалять .torrent файл после добавления
autostart = false
```

Подключение к Transmission определяется автоматически из `settings.json`.

---

## Аргументы командной строки

```
transmission-remote-slint [ФАЙЛ.torrent] [--gl|--vk|--sw|--wl]

--gl    Принудительно OpenGL
--vk    Принудительно Vulkan
--sw    Программный рендеринг (CPU)
--wl    Принудительно Wayland
```

---

## English documentation

See [README.md](README.md)

---

## Лицензия

GPL-2.0-or-later. См. [LICENSE](LICENSE).  
Использует [Slint](https://slint.dev) под лицензией GPLv3.
