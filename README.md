# WaLinux

WaLinux is an experimental, unofficial Linux desktop client built with Qt 6/QML and Go. The window is rendered by Qt Quick; it does not embed a browser or load WhatsApp's web interface. The Go service connects through the multi-device protocol implementation in [whatsmeow](https://github.com/tulir/whatsmeow).

> **Account and terms risk:** WhatsApp does not provide an officially supported consumer API for independent desktop clients. WaLinux uses an unofficial companion-device protocol implementation. WhatsApp's terms restrict unauthorized third-party apps and access, and WhatsApp may log out or restrict an account. This project is not affiliated with or endorsed by WhatsApp or Meta. Read [the research and limitations](docs/RESEARCH.md) before linking an account.

## Current capability

- QR linking as a companion device and encrypted persistence of the linked-device record.
- Receive and send one-to-one and group text messages; import available text history.
- Send image, video, audio and document files through WhatsApp's encrypted media upload flow; save received attachments through encrypted media download to a user-selected location.
- Conversation list and local search, unread counts, timestamps, delivery/read indicators, and native desktop notifications.
- Light/dark appearance, optional XDG autostart entry, desktop tray icon, keyboard shortcuts, native file dialogs, and a private local Unix socket between the Qt process and the Go service.
- SQLCipher encrypted message history and device store. The random database key is stored through the desktop Secret Service; WaLinux does not fall back to plaintext credentials if Secret Service is unavailable.

This is an early, source-built release. Media is shown as an attachment row and can be saved; inline image previews, video playback, voice recording, calls, reply/forward/edit/delete, chat pin/mute/archive controls, statuses, stickers, profile/contact/group detail pages, emoji selection, clipboard image support, drag and drop, and a first-run keyring setup wizard are not implemented. File uploads are capped at 100 MiB and downloads at 2 GiB. The client does not advertise absent functions as available.

Calls are not supported by whatsmeow and WaLinux has no call controls. See [the feature matrix](docs/RESEARCH.md#feature-matrix) for details. A real WhatsApp account is required to verify account linking and live message exchange; those checks have not been performed as part of this build.

## Install on CachyOS or Arch Linux

The one-command installer downloads the latest GitHub release, checks its SHA256 checksum, validates archive paths and shared libraries, then installs under `~/.local`. If dependencies are missing, it asks before using `sudo pacman`; it never runs the downloaded application during installation.

```sh
curl -fsSL https://raw.githubusercontent.com/axmezonar/walinux/main/install.sh | bash
```

For a noninteractive installation without a terminal for the dependency prompt, install the required packages first:

```sh
sudo pacman -S --needed binutils qt6-base qt6-declarative qt6-wayland openssl
```

The desktop session must provide an unlocked Secret Service implementation, such as GNOME Keyring. WaLinux intentionally refuses to create or open the device/history databases without access to the keyring. On Hyprland, configure a Secret Service provider in the session if one is not already started.

Uninstall the app and preserve encrypted account data:

```sh
~/.local/bin/walinux-uninstall
```

Remove the app and its local data/settings explicitly:

```sh
~/.local/bin/walinux-uninstall --purge-data
```

Data is stored under `${XDG_DATA_HOME:-~/.local/share}/walinux`, settings under `${XDG_CONFIG_HOME:-~/.config}/walinux`, and temporary runtime IPC under `$XDG_RUNTIME_DIR/walinux`.

## Build from source

Build dependencies on Arch Linux: `base-devel`, `cmake`, `ninja`, `go`, `qt6-base`, `qt6-declarative`, `qt6-wayland`, `openssl`, and `pkgconf`. The Go driver embeds SQLCipher; OpenSSL supplies its cryptographic provider.

```sh
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build
ctest --test-dir build --output-on-failure
```

The backend embeds the SQLCipher implementation through its pinned Go driver and links the system OpenSSL library. An Arch package recipe is in [packaging/arch/PKGBUILD](packaging/arch/PKGBUILD). No AppImage is published yet because Qt remains a system runtime dependency; a self-contained bundle needs a separately maintained and audited runtime set.

## Architecture and security

- **Frontend:** C++/Qt 6 integration, Qt Quick Controls and QML. Native Qt application window, `QSystemTrayIcon`, freedesktop notifications over the session bus, persistent `QSettings`, and native file dialogs. No web renderer is linked.
- **Backend:** Go with whatsmeow, launched as a child process. The frontend creates a user-only Unix domain socket, accepts only the spawned backend PID and current UID, and exchanges bounded newline-delimited JSON events.
- **Storage:** SQLCipher databases live in the XDG data directory with owner-only permissions. A 256-bit key comes from the desktop Secret Service. Credentials and message bodies are not logged by the backend.
- **Network:** WhatsApp protocol traffic is handled by whatsmeow. The current library opens a WebSocket to `wss://web.whatsapp.com/ws/chat`; that is a protocol connection, not an HTML page or WebView. Media endpoints are discovered by the protocol library. Details and third-party dependencies are in [docs/SECURITY.md](docs/SECURITY.md).
- **Telemetry:** none is implemented.

Encryption at rest protects database files while the key remains in the keyring. It does not encrypt data displayed on screen, process memory, notifications, or a compromised desktop session. Device backups are the user's responsibility. See [docs/SECURITY.md](docs/SECURITY.md).

## Development status

The repository has an Arch Linux build/test workflow and tagged release workflow. CI can verify compilation and local storage/IPC components, but it cannot verify current WhatsApp account linking. Do not assume compatibility with WhatsApp is guaranteed; service-side protocol changes may interrupt the client.

## License

WaLinux is distributed under the Mozilla Public License 2.0. See [LICENSE](LICENSE). Third-party components keep their own licenses; whatsmeow is MPL-2.0 and Qt is provided by the system packages.
