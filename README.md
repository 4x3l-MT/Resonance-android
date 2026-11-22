# Resonance Android

[![Kotlin](https://img.shields.io/badge/Kotlin-2.0%2B-7F52FF?logo=kotlin&logoColor=white)](https://kotlinlang.org)
[![Jetpack Compose](https://img.shields.io/badge/Jetpack%20Compose-Material%203-4285F4?logo=jetpackcompose&logoColor=white)](https://developer.android.com/jetpack/compose)
[![Min SDK](https://img.shields.io/badge/Min%20SDK-26%20(Android%208.0)-3DDC84?logo=android&logoColor=white)](https://developer.android.com)

Native Android client for the **Resonance** distributed multi-room audio system. Built with **Kotlin** and **Jetpack Compose**, this application serves a dual purpose:

1. **Low-Latency Audio Receiver Node:** Turns any Android device (including legacy phones connected to wired speakers or Bluetooth receivers) into a sub-millisecond synchronized audio endpoint via a custom binary TCP protocol implementation.
2. **Multi-Room Control Console:** Provides real-time playback metadata, zone volume management, and per-device latency calibration over WebSockets.

> **Companion Repository:** Requires the [`resonance-core`](https://github.com/4x3l-MT/resonance-core) server running on your local network to handle Spotify Connect ingestion, audio piping, and RPC orchestration.

---

## Key Technical Features

* **Custom Binary Protocol Parser:** Pure `java.net.Socket` TCP implementation (`tcpNoDelay = true`) parsing 26-byte Little-Endian headers (`Hello`, `CodecHeader`, `WireChunk`, `ServerSettings`, `Time`) with exponential backoff auto-reconnection.
* **NTP-Style Clock Synchronization:** Periodic time-packet exchange calculating server-to-client clock offset using a 30-sample sliding median filter to eliminate Wi-Fi jitter spikes.
* **Drift-Corrected `AudioTrack` Pipeline:** Dedicated urgent-priority audio thread feeding `AudioTrack` (`PERFORMANCE_MODE_LOW_LATENCY`) from a ring buffer, dynamically dropping samples or inserting silence when drift exceeds $\pm 5\text{ ms}$.
* **Dynamic Hardware Output Routing:** Uses `AudioDeviceCallback` to detect physical switches between **3.5 mm Wired Jack** and **Bluetooth A2DP**, automatically applying saved latency compensation profiles (e.g., `-160 ms` offset for Bluetooth speakers).
* **Background Resilience:** Runs inside a `MediaPlayback` `ForegroundService` holding `PARTIAL_WAKE_LOCK` and `WIFI_MODE_FULL_LOW_LATENCY` locks to prevent OS throttling when the screen is off.

---

## Architecture Overview

The app follows **MVVM** principles and splits network I/O into two isolated planes:

```text
resonance-core (Linux Server)
 ├── TCP Binary Stream (:1704) ──► SnapcastTcpClient ──► NtpTimeSyncEngine ──► AudioTrack
 └── Control WebSocket (:8000) ──► BackendWebSocketClient ──► MultiroomRepository ──► Compose UI
```

For a deep dive into package structure, packet layouts, and audio buffer mechanics, see [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md).

---

## Tech Stack

* **Language & Concurrency:** Kotlin 2.0, Coroutines, `StateFlow`, `ByteBuffer`
* **UI Framework:** Jetpack Compose, Material Design 3, Coil 3
* **Networking:** Raw TCP Sockets (`java.net`), OkHttp WebSockets, `kotlinx.serialization`
* **Local Storage:** Jetpack DataStore Preferences

---

## Getting Started

### Prerequisites
* Android Studio Ladybug (or newer) with JDK 17+
* An Android device running **Android 8.0 (API 26)** or higher
* A running instance of `resonance-core` on the same local subnet (LAN)

### Build & Installation
1. Clone the repository:
   ```bash
   git clone https://github.com/4x3lmt/resonance-android.git
   cd resonance-android
   ```
2. Open the project in **Android Studio** and let Gradle sync dependencies.
3. Connect your target Android device via ADB and run:
   ```bash
   ./gradlew installDebug
   ```

### Initial Configuration
1. Launch **Resonance** on your device and grant notification and Bluetooth permissions when prompted.
2. Open **Settings** (gear icon) and enter:
   * **Server IP:** The static LAN IP of your `resonance-core` host.
   * **Room Name:** Identifier for this node (e.g., `Living Room`, `Bedroom`).
   * **Latency Offsets:** Keep `0 ms` for wired 3.5 mm speakers; adjust to ~`-150 ms` to `-180 ms` if bridging audio to a Bluetooth speaker.
3. Toggle **Local Receiver Node** on the dashboard to start synchronized playback, or leave it disabled to use the device strictly as a remote controller.

---

## Roadmap

- [x] Binary TCP protocol deserialization & NTP clock sync engine
- [x] Low-latency `AudioTrack` ring buffer with drift correction
- [x] Automatic output route detection (`AudioDeviceCallback`)
- [ ] Optional Opus/FLAC hardware decoding via `MediaCodec` for low-bandwidth Wi-Fi links
- [ ] mDNS auto-discovery of `resonance-core` to remove manual IP setup
