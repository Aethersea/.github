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

**Protocols**

- **Signaling** — bidirectional gRPC stream (`SignalingService.Session`) over self-signed TLS. Exchanges `SessionConfig`, SDP offers/answers, and ICE candidates for two separate WebRTC `PeerConnection`s: one for media (video + audio RTP tracks) and one for control (DataChannels).
- **Media transport** — RTP over WebRTC SRTP/DTLS, with a custom Reed-Solomon FEC layer (40% parity overhead, extension ID 9, `urn:leviathan:fec`). Supports HEVC and AV1 video tracks plus Opus audio.
- **Control channel** — Protobuf-encoded `ControlMessage` over a WebRTC DataChannel. Carries keyboard, mouse, gamepad, touch, and telemetry messages.
- **Overlay channel** — Cursor images (lossless WebP via `nativewebp`) and cursor signals forwarded in real time.
- **Clipboard channel** — Bidirectional clipboard sync (text, PNG images, files). File payloads use chunked DataChannel transfer with lazy/delayed rendering.
- **Pairing** — `PairingService.Pair` gRPC call; credentials stored as argon2id hashes. Trusted clients identified by DTLS fingerprint (SHA-256), persisted in `trusted_clients.json`.

**Key internal packages**

| Package | Responsibility |
|---------|----------------|
| `internal/capture` | Platform capture backends; push-based `VideoFrameHandler` / `AudioDataHandler` callbacks |
| `internal/encoder` | Platform encoder backends; `VideoEncoder` interface with `Encode(pixelBuffer)` / `EncodeTexture(texture)` |
| `internal/audio` | Opus encoder — 48 kHz stereo, 10 ms frame size for minimum latency |
| `internal/input` | Virtual input injection — CGEvent (macOS), Win32 `SendInput` + ViGEmBus gamepad (Windows) |
| `internal/streaming` | WebRTC session management, full pipeline (`Pipeline` interface), RTP packetization, RTCP monitoring, bandwidth probe |
| `internal/signaling` | gRPC signaling server with display-refresh helpers per platform |
| `internal/clipboard` | macOS: Unix socket IPC to `clipboard-helper` companion; Windows: Win32 clipboard with delayed rendering `HWND` |
| `internal/cursor` | Platform cursor capture; emits `CursorEvent{CursorID, Image(RGBA), Hidden}` |
| `internal/pairing` | `TrustStore` — argon2id credential verification, trusted client list |
| `internal/config` | TOML config — network mode (LAN/STUN/manual), video (codec, bitrate, FPS up to 4K/120), audio, input, clipboard |
| `internal/crypto` | ECDSA P-256 self-signed TLS cert for the gRPC server |

**Build**

```bash
# macOS: requires Xcode, ScreenCaptureKit, VideoToolbox frameworks
# Windows: requires MSYS2 MinGW-w64, libopus.a in cgo/windows/
make build     # → build/leviathan(.exe)
make proto     # regenerate protobuf Go files
make test
```

CLI:
```
leviathan [-grpc-port <port>] [-set-credentials -username <u> -password <p>]
```

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

**SharedArrayBuffer ring buffers**

Four SABs bridge the Rust native module to the browser pipeline with zero copies:

| SAB | Writer | Reader | Content |
|-----|--------|--------|---------|
| `videoSAB` | Rust (RTP receiver) | JS WebCodecs `VideoDecoder` | H.265 / AV1 NAL units |
| `audioSAB` | Rust (Opus receiver) | Web AudioWorklet | Decoded PCM frames |
| `inputSAB` | JS renderer | Rust (input poller) | Protobuf `ControlMessage`s |
| `immersiveSAB` | Rust OS hooks | JS renderer | Raw keyboard/mouse hook events |

**Rust native module — key responsibilities**

- **`server_manager`** — discovers, stores, and refreshes Leviathan server entries. Calls `ManagementService` gRPC to retrieve `ServerInfo` (name, platform, GPU names, supported codecs, display resolution). Generates and persists the client's DTLS fingerprint.
- **`session`** — drives the full WebRTC session lifecycle: connects to `SignalingService.Session`, emits `media-sdp-offer` to JS for browser-side `RTCPeerConnection`, handles the control `PeerConnection` entirely in Rust.
- **`media`** — creates the WebRTC media `PeerConnection` with H.265 and AV1 RTP capabilities. RTP receiver writes NAL units to the video SAB; FEC recovery (`fec.rs`, Reed-Solomon) runs inline.
- **`control`** — creates the control `PeerConnection`, manages DataChannels (control, telemetry, overlay, clipboard). Polls `inputSAB` and forwards protobuf `ControlMessage`s to the server.
- **`gamepad`** — SDL3-based gamepad polling (1 ms interval, up to 16 controllers). Limelight-compatible capability flags (analog triggers, rumble, LED, accelerometer, gyroscope, touchpad). Mouse emulation via long-press Start; quit combo (Start+Select+LB+RB).
- **`immersive`** — OS-level input hooks for seamless keyboard/mouse capture: `WH_KEYBOARD_LL` + `WH_MOUSE_LL` (Windows), `CGEventTap` (macOS), `XGrabKeyboard` (Linux X11), `zwp_keyboard_shortcuts_inhibit` (Linux Wayland).
- **`av1_decoder`** — software AV1 decoding via `rav1d` (pure-Rust dav1d port). Writes planar YUV frames to a separate decoded-video SAB; renderer uses a WebGL2 YUV→RGB shader.
- **`clipboard`** — macOS: `objc2`-based `NSPasteboard` access + Unix socket IPC to `clipboard-helper`. Windows: Win32 clipboard with delayed rendering.

**Renderer pipeline**

1. `StreamingPage` creates a browser `RTCPeerConnection` for the media leg. Receives H.265 `EncodedVideoChunk`s via `RTCRtpReceiver`, feeds them to `VideoDecoder` (WebCodecs). For software AV1, NAL units are read from the video SAB and decoded in Rust.
2. Decoded frames rendered on a WebGL2 canvas.
3. `AudioDecoder` (WebCodecs Opus) feeds a `AudioWorklet` that reads PCM from the audio SAB.
4. Input events (`keydown`, `mousemove`, gamepad, touch) are encoded as protobuf `ControlMessage`s and written to the input SAB; the Rust input poller reads and forwards them.

**Key Electron main process flags**

`PlatformHEVCDecoderSupport`, `WebRtcAllowH265Receive`, `in-process-gpu` (eliminates Mojo IPC on `VideoFrame` release — critical for 60 fps+ decoding), `ignore-gpu-blocklist`, `enable-zero-copy`.

**Build**

```bash
npm install
npm run dev          # development (auto-rebuilds native module)
npm run build        # production installer / DMG / AppImage
npm run build:native # rebuild Rust native module only
```

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Leviathan (Server)                      │
│                                                                 │
│  ┌──────────────┐     ┌────────────────────┐                   │
│  │   Capture    │────▶│  Hardware Encoder  │                   │
│  │  DXGI / SCK  │     │ MFT / VideoToolbox │                   │
│  └──────────────┘     └─────────┬──────────┘                   │
│                                 │ H.265 / AV1 NAL               │
│  ┌──────────────┐     ┌─────────▼──────────┐   ┌────────────┐ │
│  │    Audio     │────▶│  RTP + FEC layer   │◀──│ RTCP Mon.  │ │
│  │   WASAPI /   │     │ Reed-Solomon 40%   │   └────────────┘ │
│  │  CoreAudio   │     └─────────┬──────────┘                   │
│  └──────────────┘               │                              │
│                        WebRTC SRTP/DTLS                        │
│  ┌──────────────────────────────▼──────────────────────┐       │
│  │   gRPC Signaling  ·  Control DC  ·  Clipboard DC    │       │
│  │   Overlay DC  ·  Telemetry DC  ·  Bandwidth Probe   │       │
│  └──────────────────────────────┬──────────────────────┘       │
└────────────────────────────────/───────────────────────────────┘
                                 │ WebRTC (SRTP + DTLS)
┌────────────────────────────────▼───────────────────────────────┐
│                           Shen (Client)                         │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐ │
│  │                    Rust Native Module                     │ │
│  │  server_manager · session · media · control · gamepad    │ │
│  │  immersive hooks · FEC recovery · clipboard · av1_dec    │ │
│  └───────┬──────────────┬────────────────────┬──────────────┘ │
│     videoSAB        audioSAB             inputSAB              │
│  ┌────────▼──┐   ┌──────▼──────┐    ┌────────▼───────────┐   │
│  │ WebCodecs │   │ AudioWorklet│    │  JS Input Encoder  │   │
│  │  H.265 /  │   │  PCM output │    │  (keyboard/mouse/  │   │
│  │  sw AV1   │   └─────────────┘    │   gamepad/touch)   │   │
│  └────▼──────┘                      └────────────────────┘   │
│  ┌────▼──────────────────────────────────────────────────┐   │
│  │  WebGL2 canvas · Cursor overlay · Performance HUD     │   │
│  └───────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────┘
```

## Protocol Buffers

Both repositories share a common set of `.proto` definitions:

| File | Purpose |
|------|---------|
| `signaling.proto` | Bidirectional gRPC session stream — SDP, ICE, `SessionConfig`, `SessionControl` |
| `pairing.proto` | PIN-based pairing — exchange of DTLS fingerprints |
| `management.proto` | Server info, config read/write, unpair |
| `control.proto` | Input events (keyboard, mouse, gamepad, touch), IDR request, rumble, bitrate estimate |
| `clipboard.proto` | Clipboard sync and chunked DataChannel file transfer |
| `overlay.proto` | Cursor images (WebP), cursor signals, settings |
| `clipboard_helper.proto` | IPC with the macOS `clipboard-helper` companion process |

## Documentation

Full documentation is available at [theaethersea.com](https://theaethersea.com):

- [Architecture overview](https://theaethersea.com/docs/architecture)
- [Leviathan — getting started](https://leviathan.theaethersea.com/docs/getting-started)
- [Leviathan — configuration](https://leviathan.theaethersea.com/docs/configuration)
- [Leviathan — encoding](https://leviathan.theaethersea.com/docs/encoding)
- [Shen — getting started](https://shen.theaethersea.com/docs/getting-started)
- [Shen — features](https://shen.theaethersea.com/docs/features)
- [Shen — configuration](https://shen.theaethersea.com/docs/configuration)
