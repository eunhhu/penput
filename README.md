# Penput

Turn a phone or tablet into a wireless touchpad for your PC. A small Rust server exposes a web touchpad over HTTP/WebSocket and a binary UDP endpoint for the native iOS client, and moves the host mouse with `enigo`.

[![CI](https://github.com/eunhhu/penput/actions/workflows/ci.yml/badge.svg)](https://github.com/eunhhu/penput/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

**Status:** Maintained — v0.3.1; pointer movement only (no clicks, scrolling, or gestures). Developed primarily against a Windows host.

## Features

- Web client: open a URL on any phone on the same LAN, tap **Connect**, go fullscreen, and move the mouse
- Native iOS/iPadOS client (SwiftUI, `Network.framework` over UDP) for lower latency than WebKit; optional **stylus-only** mode
- Absolute mode (touch position = screen position, pad resized to match the host aspect ratio) and relative, trackpad-style mode on both clients
- Single active client per transport, with a `y/n` approval prompt in the server terminal (`--auto-approve` to skip)
- Latest-value-only mouse worker (Condvar-based) so a backlog never causes lag; on-screen metrics overlay with RTT, send rate, ping/pong intervals
- Linux and Windows server binaries built by GitHub Actions; an unsigned iOS `.ipa` is built on macOS runners

## How it works

- `src/http.rs` serves `static/` (`index.html`, `app.js`, `touchpad.js`, `style.css`) via `tower-http` `ServeDir` with `Cache-Control: no-store`.
- `src/websocket.rs` accepts a JSON `init` message with the client viewport, replies with `remote_screen`, then reads 4-byte big-endian `x:u16 | y:u16` frames and app-level ping/pong.
- `src/udp.rs` implements the native protocol: client → server `HELLO 0x01 [w:u16][h:u16]`, `MOVE 0x02 [x:u16][y:u16]`, `PING 0x03 [t:u64]`; server → client `ACCEPT 0x10 [w:u16][h:u16]`, `REJECT 0x11`, `BUSY 0x12`, `PONG 0x13 [t:u64]`. The session is released after ~5 s without traffic.
- `src/connection.rs` holds the single-client slot and the approval broker; `src/mouse.rs` maps coordinates to the host display (`display-info`) and drives `enigo` from a dedicated thread.
- `src/main.rs` parses flags, prints the LAN URLs, and starts the three servers with tokio.

## Tech stack

- Language: Rust 2024 (MSRV 1.88)
- Server: tokio, axum 0.8 (`ws`), tokio-tungstenite, tower-http, enigo 0.6, display-info, mdns-sd, tracing
- Web client: vanilla HTML/CSS/JS
- iOS client: SwiftUI, Network.framework, CADisplayLink; project generated with XcodeGen (`ios/PenputIOSClient/project.yml`)
- CI: `cargo fmt` / `clippy -D warnings` / build on push (`ci.yml`); release builds when the head commit message contains `release` (`release-build.yml`)

## Getting started

### Requirements

- Rust 1.88+ (Linux builds need X11/xdo development packages for `enigo`)
- Phone and PC on the same network; allow TCP 8080/9001 and UDP 9002 through the firewall
- iOS client: Xcode + XcodeGen on macOS, or sideload the CI-built `.ipa` (unsigned; free Apple IDs expire after 7 days)

### Install

```bash
cargo install penput
# or from source
cargo build --release
```

Pre-built Linux and Windows binaries are on the [Releases page](https://github.com/eunhhu/penput/releases).

### Run

```bash
penput                                   # HTTP 8080, WebSocket 9001, UDP 9002
penput --port 8080 --ws-port 9001 --udp-port 9002 --auto-approve
```

1. Start the server; it prints `http://<PC_IP>:8080`.
2. Web: open `http://<PC_IP>:8080/?ws=9001` on the phone, tap **Connect**, approve in the terminal.
3. iOS app: enter the PC IP and UDP port, pick absolute or relative mode, connect.
4. **Exit (✕)** leaves fullscreen and disconnects.

Build the iOS project locally:

```bash
cd ios/PenputIOSClient
xcodegen generate
open PenputIOSClient.xcodeproj
```

Add `NSLocalNetworkUsageDescription` to `Info.plist` (already present) so iOS prompts for local-network access.

## Project structure

```text
src/
  main.rs         CLI flags, startup banner, server spawn
  http.rs         static file server
  websocket.rs    web client protocol
  udp.rs          native client protocol
  connection.rs   single-client slot + approval prompt
  mouse.rs        coordinate mapping and enigo worker thread
static/           web touchpad (app.js, touchpad.js, index.html, style.css)
ios/PenputIOSClient/   SwiftUI client (UdpTouchClient.swift, TouchPadView.swift, ContentView.swift)
.github/workflows/     ci.yml, release-build.yml
```

## Configuration

Command-line flags only: `--port` (HTTP, 8080), `--ws-port` (9001), `--udp-port` (9002), `--auto-approve`. No environment variables or config files.

## Troubleshooting

- Phone cannot reach the page: same LAN? firewall open on 8080/9001/9002?
- "Connection failed": the `?ws=` query must match `--ws-port`.
- No approval prompt: focus the server terminal; or run with `--auto-approve`.
- Choppy movement in the web client: try the native iOS client over UDP.

## License

[MIT](LICENSE).
