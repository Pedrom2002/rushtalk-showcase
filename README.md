<h1 align="center">RushTalk</h1>

<p align="center">
  <strong>A real-time voice and text communication platform for gamers — built from scratch with a custom Rust audio engine.</strong>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Rust-Audio%20Engine-orange?style=for-the-badge&logo=rust&logoColor=white" />
  <img src="https://img.shields.io/badge/Go-Backend%20API-00ADD8?style=for-the-badge&logo=go&logoColor=white" />
  <img src="https://img.shields.io/badge/Svelte%205-Frontend-FF3E00?style=for-the-badge&logo=svelte&logoColor=white" />
  <img src="https://img.shields.io/badge/Tauri%202.0-Desktop-24C8D8?style=for-the-badge&logo=tauri&logoColor=white" />
  <img src="https://img.shields.io/badge/LiveKit-WebRTC%20SFU-7B61FF?style=for-the-badge&logo=webrtc&logoColor=white" />
  <img src="https://img.shields.io/badge/PostgreSQL-16-4169E1?style=for-the-badge&logo=postgresql&logoColor=white" />
  <img src="https://img.shields.io/badge/Docker-Kubernetes-2496ED?style=for-the-badge&logo=docker&logoColor=white" />
</p>

---

## Overview

**RushTalk** is a full-stack, production-grade voice communication application — think Discord, rebuilt from scratch with a focus on low-latency audio and modern architecture. The project spans four languages (Rust, Go, TypeScript, SQL) across 12,000+ lines of code, delivering a complete desktop client with real-time voice chat, text messaging, server management, and a gaming overlay.

The core differentiator is a **custom zero-allocation Rust audio engine** that achieves sub-60ms mouth-to-ear latency — competitive with professional tools like Mumble and TeamSpeak. Unlike Electron-based alternatives, RushTalk uses Tauri 2.0 for a native desktop experience at a fraction of the memory footprint (~40-60 MB vs Discord's ~300 MB).

The backend follows **Clean Architecture** principles in Go, with a PostgreSQL-backed domain model, Redis caching layer, WebSocket real-time events, and a full Kubernetes deployment pipeline. Every layer is designed, tested, and documented to production standards.

---

## :sparkles: Key Features

### Voice Communication
- Multi-user voice channels via LiveKit SFU with **< 60ms latency** (same region)
- Custom audio processing pipeline: high-pass filter, noise suppression, AGC, VAD
- Opus encoding at 32 kbps with adaptive Forward Error Correction
- Adaptive jitter buffer (EMA-based, 10-80ms range)
- Packet Loss Concealment — repeats last decoded frame for up to 50ms
- Per-user volume control, mute/deafen, and speaking indicators
- Push-to-talk and voice activity detection modes

### Text Chat & Messaging
- Per-channel message history with keyset cursor pagination
- Real-time delivery via WebSocket with typing indicators
- File attachments via S3-compatible storage (MinIO)
- Message grouping for consecutive messages within 5-minute windows

### Server & Channel Management
- Create and join servers via invite codes
- Channel types: text, voice, and category
- Discord-style role-based permission system (64-bit bitmask with channel overwrites)
- Member management with multi-role assignment

### Gaming Overlay
- Always-on-top transparent window with click-through support
- Global hotkey toggle (`Shift + Backtick`)
- Shows active voice channel participants with speaking indicators
- Configurable opacity and position

### Authentication
- Email/password registration and login
- OAuth: Steam (OpenID 2.0) and Google (OAuth 2.0 + PKCE)
- JWT RS256 with 15-minute access tokens and 7-day sliding refresh
- Automatic token refresh on 401 responses

### Cross-Platform
- Windows (x86_64)
- macOS (ARM64 + Intel universal)
- Linux (x86_64 AppImage)

---

## :building_construction: Architecture

```
                                 ┌──────────────────────┐
                                 │    LiveKit SFU        │
                                 │  (WebRTC Routing)     │
                                 └──────┬───────┬───────┘
                                        │       │
                              publish   │       │  subscribe
                              (PCM)     │       │  (PCM)
                                        │       │
┌────────────────────────────────────────┼───────┼────────────────────────────────┐
│  Desktop Client (Tauri 2.0)           │       │                                │
│                                       │       │                                │
│  ┌──────────┐   ┌──────────────────┐  │       │  ┌──────────────────────────┐  │
│  │  Svelte  │◄──│  Tauri IPC       │◄─┘       └─►│  Rust Audio Engine       │  │
│  │  Frontend│   │  Commands (40+)  │              │                          │  │
│  │          │──►│                  │──────────────►│  Capture → Pipeline →    │  │
│  │  18 comps│   │  audio / voice / │              │  Opus → LiveKit          │  │
│  │  9 stores│   │  overlay / net   │              │                          │  │
│  │  6 svc   │   └──────────────────┘              │  LiveKit → Mixer →       │  │
│  └──────────┘                                     │  Playback (per-user)     │  │
│                                                   └──────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
                              │
                              │  REST + WebSocket
                              ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Go Backend (Clean Architecture)                                               │
│                                                                                │
│  ┌───────────┐  ┌──────────────┐  ┌───────────────┐  ┌──────────────────────┐  │
│  │  Echo     │  │  Application │  │  Domain        │  │  Infrastructure      │  │
│  │  Handlers │─►│  Services    │─►│  Entities +    │  │  PostgreSQL (pgx v5) │  │
│  │  (41 eps) │  │  (5 svc)     │  │  Permissions   │  │  Redis (sessions)    │  │
│  │           │  │              │  │  (pure logic)  │  │  MinIO (S3 storage)  │  │
│  │  WS Hub   │  └──────────────┘  └───────────────┘  │  LiveKit (tokens)    │  │
│  │  (events) │                                        └──────────────────────┘  │
│  └───────────┘                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### By Layer

**Desktop / Frontend** — Svelte 5 with TypeScript, 18 components, 9 reactive stores, 6 service modules. SvelteKit routing for main, login, register, and overlay windows.

**Backend API** — Go with Echo framework. Clean Architecture: domain entities with pure business logic, application-layer use cases, infrastructure adapters (PostgreSQL, Redis, MinIO, LiveKit). 41 REST endpoints + WebSocket hub for real-time events.

**Audio Engine** — Custom Rust crate (`rushtalk-audio`, 1,400+ LOC). Handles capture, processing pipeline, Opus codec, multi-user mixing, and playback. Zero heap allocations on the hot path.

**Voice Transport** — Rust crate (`rushtalk-voice`, 650+ LOC) integrating LiveKit's Rust SDK for WebRTC publish/subscribe. LiveKit SFU handles NAT traversal (ICE/STUN/TURN) and multi-user routing.

**Infrastructure** — Docker Compose for local development (7 services), Kubernetes manifests with HPA and TLS via cert-manager, Prometheus + Grafana monitoring, GitHub Actions CI/CD with multi-platform builds.

---

## :headphones: Audio Pipeline

The audio engine is the core differentiator — a custom, low-latency pipeline built entirely in Rust with real-time thread safety guarantees.

### TX (Transmit) Path

```
Microphone (cpal, 48kHz mono, 5ms buffers)
    │
    ▼
Raw PCM Ring Buffer (lock-free SPSC, 200ms capacity)
    │
    ▼
CaptureProcessor (dedicated thread, elevated priority)
    ├── High-pass filter (100Hz cutoff)
    ├── Noise suppression (configurable: Off / Low / Standard / Aggressive)
    ├── Automatic Gain Control (target -18 dBFS)
    ├── Voice Activity Detection (energy + threshold)
    ├── Input volume scaling
    │
    └──► LiveKit NativeAudioSource (PCM i16 → WebRTC handles Opus internally)
```

### RX (Receive) Path

```
LiveKit NativeAudioStream (per remote participant)
    │
    ▼
RX Bridge Task (tokio, 10ms polling)
    │
    ▼
Decoded Audio Ring Buffer (lock-free SPSC, 64 packets)
    │
    ▼
PlaybackProcessor (dedicated thread, elevated priority)
    ├── Per-user frame accumulation
    ├── Packet Loss Concealment (repeat last frame, max 50ms)
    ├── Multi-user AudioMixer (time-based, 10ms ticks)
    ├── Stale user garbage collection (2s timeout)
    │
    ▼
Mixed PCM Ring Buffer → Speaker (cpal, 48kHz, volume scaling + deafen)
```

### Design Principles

- **Audio callback threads are sacred** — no allocation, no mutex, no I/O, no logging
- **Lock-free ring buffers** (`rtrb`) for all inter-thread communication
- **TX and RX are fully decoupled** — a spike in RX processing (many participants) never affects TX (your mic)
- **Spin-yield capture loop** with 64-iteration spin budget before sleep fallback
- **Per-participant state**: each remote user gets a dedicated decoder and jitter buffer (Opus state is per-stream)

---

## :zap: Technical Highlights

### Performance Metrics

| Metric | Value |
|--------|-------|
| End-to-end latency | **< 60ms** (same region) |
| Frame size | 480 samples (10ms @ 48kHz) |
| Hardware buffer | 240 samples (5ms) when supported |
| Opus bitrate | 32 kbps (adaptive FEC) |
| Jitter buffer range | 10-80ms (EMA-adaptive) |
| Hot-path allocations | **Zero** (pre-allocated buffers) |
| Memory footprint | **~40-60 MB** (vs Discord ~300 MB) |
| API Docker image | **~20 MB** (distroless multi-stage build) |

### Engineering Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Ring buffers | `rtrb` (lock-free SPSC) | Zero-allocation, real-time safe — audio threads must never block |
| Desktop framework | Tauri 2.0 over Electron | 5-10x less memory, native Rust backend for audio |
| Audio I/O | `cpal 0.15` | Cross-platform (WASAPI / CoreAudio / PipeWire / ALSA) |
| Codec | Opus via `audiopus 0.2` | Industry standard for voice, adaptive FEC, 10ms frames |
| Backend architecture | Clean Architecture (Go) | Domain logic testable without DB; permission resolution is pure functions |
| Permissions | 64-bit bitmask | In-memory resolution with channel overwrites — no DB round-trip |
| Auth tokens | JWT RS256 | Stateless verification, 15-min access + 7-day sliding refresh |
| Database driver | `pgx v5` | Statement caching, connection pooling, type-safe queries |
| Release builds | LTO + strip + codegen-units=1 | Maximum binary optimization for production |

### Codebase Scale

| Component | Metric |
|-----------|--------|
| Total LOC | **12,000+** across Rust, Go, TypeScript, SQL |
| Rust audio engine | 1,400+ LOC |
| Rust voice integration | 650+ LOC |
| Rust shared protocol | 350 LOC |
| Tauri IPC commands | 40+ commands |
| Svelte components | 18 components |
| Frontend stores | 9 reactive stores |
| REST API endpoints | 41 endpoints |
| Database tables | 16 tables (310 lines of SQL, 3 migrations) |
| Docker services | 7 (local dev stack) |

### Test Coverage

| Suite | Framework | Tests | Status |
|-------|-----------|-------|--------|
| Rust Audio + Protocol | `cargo test` | 22 | All passing |
| Go Backend | `go test -race` (table-driven) | All | All passing |
| Frontend | Vitest | 12 | All passing |
| Type Check | `svelte-check` | 504 files | Zero errors |

---

## :framed_picture: Screenshots

*Screenshots of the desktop application, voice channels, chat interface, settings panel, and gaming overlay.*

> Screenshots available upon request.

---

## :wrench: Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| **Desktop Runtime** | Tauri 2.0 | Native wrapper, IPC bridge, auto-updater |
| **Frontend** | Svelte 5 + TypeScript | Reactive UI, state management, SvelteKit routing |
| **Audio Engine** | Rust (`cpal` + `audiopus` + `rtrb`) | Capture, processing pipeline, Opus codec, playback |
| **Voice Transport** | LiveKit (WebRTC SFU) | Multi-user voice routing, NAT traversal (TURN/STUN) |
| **Backend API** | Go (Echo framework) | REST endpoints, Clean Architecture, business logic |
| **Real-time Events** | WebSocket + Redis Pub/Sub | Chat, presence, typing indicators, voice state |
| **Database** | PostgreSQL 16 | Users, servers, channels, messages (16 tables) |
| **Cache / Sessions** | Redis 7 | JWT blocklist, rate limiting, presence, permissions cache |
| **Object Storage** | MinIO (S3-compatible) | File uploads, avatars, attachments |
| **Monitoring** | Prometheus + Grafana | Metrics collection, dashboards, alerting |
| **CI/CD** | GitHub Actions | 3 parallel test suites, multi-platform builds, Docker push |
| **Deployment** | Kubernetes + Docker | HPA (1-10 replicas), TLS (cert-manager), health checks |

---

## :books: Project Structure

```
RushTalk/
├── apps/
│   ├── api/                    # Go backend (Clean Architecture)
│   │   ├── cmd/server/         # Entry point, dependency injection
│   │   └── internal/
│   │       ├── domain/         # Business entities, repository interfaces
│   │       ├── application/    # Use-case services (auth, server, channel, voice)
│   │       ├── infrastructure/ # PostgreSQL, Redis, MinIO, LiveKit adapters
│   │       └── interface/      # HTTP handlers, WebSocket hub
│   │
│   └── desktop/                # Tauri 2.0 desktop application
│       ├── src/                # Svelte 5 frontend (18 components, 9 stores)
│       └── src-tauri/          # Rust backend (40+ IPC commands)
│
├── crates/                     # Rust workspace
│   ├── rushtalk-audio/         # Audio engine (1,400+ LOC)
│   ├── rushtalk-voice/         # LiveKit integration (650+ LOC)
│   └── rushtalk-protocol/      # Shared types and events (350 LOC)
│
├── deploy/
│   ├── k8s/                    # Kubernetes manifests
│   ├── prometheus/             # Monitoring configuration
│   └── livekit/                # LiveKit server config
│
├── .github/workflows/          # CI (test) + Release (multi-platform build)
├── docker-compose.yml          # Local dev stack (7 services)
├── Makefile                    # Development commands
└── Cargo.toml                  # Rust workspace configuration
```

---

## :shield: Status

> **Private repository** — this showcase documents the architecture and features of RushTalk. The source code is maintained in a private repository. For a walkthrough of the codebase, technical discussion, or a live demo, feel free to reach out.

---

<p align="center">
  Built with Rust, Go, Svelte, and a lot of audio engineering.
</p>
