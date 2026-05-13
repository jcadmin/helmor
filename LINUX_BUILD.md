# Building Helmor on Linux

This guide covers building and running Helmor on Linux from source. For an installer-only path, see the `.deb` / `.AppImage` artifacts on the [fork's Releases page](https://github.com/jcadmin/helmor/releases).

## Supported Distributions

- **Ubuntu 22.04+** / **Debian 12+** — primary target, fully tested
- **Fedora 39+** — untested but should work with the equivalent dependencies (`webkit2gtk4.1-devel`, `gtk3-devel`, `libsoup3-devel`, `librsvg2-devel`, `libayatana-appindicator3-devel`, `patchelf`)

WebKit2GTK 4.1 is required. Ubuntu 20.04 and other distros that only ship 4.0 are not supported.

## System Dependencies

On Debian/Ubuntu:

```bash
sudo apt update
sudo apt install -y \
  libwebkit2gtk-4.1-dev \
  libgtk-3-dev \
  libsoup-3.0-dev \
  librsvg2-dev \
  libayatana-appindicator3-dev \
  patchelf \
  build-essential \
  curl \
  wget \
  file
```

## Toolchain

- **Rust stable 1.95+** — install via [rustup](https://rustup.rs/)
- **Bun 1.3+** — install via `curl -fsSL https://bun.sh/install | bash`
- **Node 20+** — required by a few sidecar scripts

If you prefer a reproducible environment, the repo ships a `flake.nix` with all of the above pinned. See [NIX_SETUP.md](./NIX_SETUP.md).

## Build Steps

```bash
git clone https://github.com/jcadmin/helmor.git
cd helmor

bun install
bun run tauri build --bundles deb,appimage --target x86_64-unknown-linux-gnu
```

Artifacts land in:

- `src-tauri/target/x86_64-unknown-linux-gnu/release/bundle/deb/Helmor_*.deb`
- `src-tauri/target/x86_64-unknown-linux-gnu/release/bundle/appimage/Helmor*.AppImage`

For development, `bun run dev` works the same as on macOS.

## Install

`.deb`:

```bash
sudo dpkg -i src-tauri/target/x86_64-unknown-linux-gnu/release/bundle/deb/Helmor_*.deb
sudo apt -f install   # if dpkg complains about missing runtime deps
```

`.AppImage`:

```bash
chmod +x src-tauri/target/x86_64-unknown-linux-gnu/release/bundle/appimage/Helmor*.AppImage
./src-tauri/target/x86_64-unknown-linux-gnu/release/bundle/appimage/Helmor*.AppImage
```

## Known Differences vs macOS

This fork ports Helmor to Linux but does not try to emulate every macOS-specific behavior. Notable differences:

- **No macOS Keychain integration.** Claude OAuth credentials are read directly from `~/.claude/.credentials.json` (the same file the `claude` CLI writes). There is no libsecret/D-Bus fallback.
- **No Finder "Reveal in Finder".** The equivalent action opens the containing directory in the system default file manager via `xdg-open`.
- **No native macOS menu bar.** Linux uses GTK's default window menu; menu items wired to the macOS app menu (e.g. global Quit, Edit submenu) are not duplicated.
- **Clipboard backend is platform-detected.** On Wayland the app shells out to `wl-copy` / `wl-paste`; on X11 it uses `xclip`. Install whichever one matches your session if clipboard actions stop working.

These differences are gated in source behind `#[cfg(target_os = "linux")]` blocks so upstream macOS code stays untouched.

## Troubleshooting

**`could not execute process sccache (No such file or directory)`**

The repo's `.cargo/config.toml` only sets `rustc-wrapper = "sccache"` when `sccache` is on `PATH`. If you see this anyway, install sccache (`cargo install sccache`) or unset the env var that's forcing it.

**`Package webkit2gtk-4.1 was not found`**

You're on a distro that only has 4.0. Upgrade to Ubuntu 22.04+ / Debian 12+ — Tauri v2 dropped 4.0 support upstream and we don't backport it.

**AppImage refuses to start / exits silently**

Run with `--appimage-extract-and-run` to bypass FUSE, or extract and inspect:

```bash
./Helmor*.AppImage --appimage-extract
./squashfs-root/AppRun
```

Most "silent exit" cases are missing `libfuse2` (install `sudo apt install libfuse2t64` on 24.04+).

**`deb` install reports missing dependencies**

Run `sudo apt -f install` after `dpkg -i` to pull in transitive runtime deps. The `.deb` declares the same `libwebkit2gtk-4.1-0`, `libgtk-3-0`, etc. that the build needs.

## Updating from Upstream

This repository is a fork of [dohooo/helmor](https://github.com/dohooo/helmor) and rebases onto upstream `main` periodically. Linux-specific code is intentionally kept inside `#[cfg(target_os = "linux")]` blocks (or platform-suffixed files like `linux_credentials.rs`) so upstream merges produce minimal conflicts.

If you're contributing a Linux fix, please keep the same convention — don't touch shared code paths unless the change benefits macOS as well.
