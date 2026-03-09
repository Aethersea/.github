<div align="center">

# Aethersea

**Open-source, high-performance remote desktop**

[![License](https://img.shields.io/github/license/aethersea/leviathan?style=flat-square)](https://github.com/aethersea/leviathan/blob/main/LICENSE)
[![Go](https://img.shields.io/badge/Go-1.24-00ADD8?style=flat-square&logo=go)](https://go.dev)
[![Electron](https://img.shields.io/badge/Electron-35-47848F?style=flat-square&logo=electron)](https://www.electronjs.org)
[![Rust](https://img.shields.io/badge/Rust-napi--rs-CE422B?style=flat-square&logo=rust)](https://napi.rs)

[theaethersea.com](https://theaethersea.com) · [Docs](https://theaethersea.com/docs/intro) · [Shen Docs](https://shen.theaethersea.com) · [Leviathan Docs](https://leviathan.theaethersea.com)

</div>

---

Aethersea is a remote desktop platform built for low latency and high fidelity. It uses WebRTC as its media transport with a custom FEC layer, hardware-accelerated capture and encoding on the server side, and hardware-accelerated decoding on the client side.

## Repositories

### [Leviathan](https://github.com/aethersea/leviathan) — Server

A Go server that runs on the host machine. It captures the desktop and audio, encodes them with the GPU, and streams them over WebRTC to connected Shen clients.

**Stack:** Go 1.24 · CGo (Objective-C / C) · pion/webrtc v4 · gRPC + Protocol Buffers

| Platform | Capture | Encoder |
|----------|---------|---------|
| Windows 10/11 | DXGI Desktop Duplication + WASAPI | Media Foundation Transform (NVENC / AMF / QuickSync) |
| macOS 12+ | ScreenCaptureKit | VideoToolbox (HEVC) · rav1e software (AV1) |


---

### [Shen](https://github.com/aethersea/shen) — Client

A cross-platform desktop client built with Electron + a Rust native module (`napi-rs`). The renderer uses WebCodecs / WebGL2 to decode and display the stream; a SharedArrayBuffer ring-buffer architecture eliminates copies between the Rust layer and the browser pipeline.

**Stack:** Electron 35 · TypeScript · React 19 · MUI v7 · Zustand · Vite · Rust (napi-rs) · SDL3

| Platform | Build target | Native module |
|----------|-------------|---------------|
| Windows x64 | NSIS installer | `win32-x64-gnu` |
| macOS Apple Silicon | Universal DMG | `darwin-arm64` |
| macOS Intel | Universal DMG | `darwin-x64` |
| Linux x64 / ARM64 | AppImage | `linux-x64-gnu`, `linux-arm64-gnu` |


## Documentation

Full documentation is available at [theaethersea.com](https://theaethersea.com):

- [Architecture overview](https://theaethersea.com/docs/architecture)
- [Leviathan — getting started](https://leviathan.theaethersea.com/docs/getting-started)
- [Leviathan — configuration](https://leviathan.theaethersea.com/docs/configuration)
- [Leviathan — encoding](https://leviathan.theaethersea.com/docs/encoding)
- [Shen — getting started](https://shen.theaethersea.com/docs/getting-started)
- [Shen — features](https://shen.theaethersea.com/docs/features)
- [Shen — configuration](https://shen.theaethersea.com/docs/configuration)
