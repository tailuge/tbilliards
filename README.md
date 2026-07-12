# Billiards — Tauri Desktop Wrapper

A minimal Tauri v2 desktop app that wraps the hosted billiards web game at
`https://billiards.tailuge.workers.dev/lobby` in a desktop window.

No game assets are bundled — the app loads the remote URL directly.

## Prerequisites (Ubuntu)

```bash
sudo apt-get update && sudo apt-get install -y \
    libwebkit2gtk-4.1-dev \
    libgtk-3-dev \
    libayatana-appindicator3-dev \
    librsvg2-dev \
    patchelf \
    xdg-utils \
    build-essential \
    pkg-config \
    curl \
    file
```

Install Rust (if not already installed):

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"
```

Install the Tauri CLI:

```bash
cargo install tauri-cli --version "^2"
```

## Run in development

```bash
cargo tauri dev
```

This opens a 1024×768 window titled "Billiards" that loads the remote game.

## Build locally (Linux)

```bash
cargo tauri build
```

The compiled binary will be in `src-tauri/target/release/`.

## Configuration

The remote URL is set in one place:

**`src-tauri/tauri.conf.json`** → `build.frontendDist`

To later bundle local assets instead, change this to a relative path (e.g. `"../dist"`)
and remove the `devUrl` entry.

## Project structure

```
src-tauri/
├── tauri.conf.json        # Main config (URL, window, app metadata)
├── Cargo.toml             # Rust dependencies
├── build.rs               # Tauri build script
├── src/
│   ├── main.rs            # Entry point
│   └── lib.rs             # Minimal Tauri builder
├── capabilities/
│   └── default.json       # Permissions for remote URL access
└── icons/                 # App icons (replace with your own)
```
