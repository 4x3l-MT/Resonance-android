# Arquitectura de Software: Resonance Android (`resonance-android`)

Especificación técnica y diseño modular de la aplicación nativa en **Kotlin**, encargada de operar simultáneamente como **Nodo Receptor de Audio Sincronizado** (cliente compatible con el protocolo binario de Snapcast) y **Consola de Control Multi-Room**.

---

## 1. Visión General y Dualidad Operativa

`resonance-android` desacopla la reproducción de audio en tiempo real del control de estado de la interfaz mediante dos planos de comunicación concurrentes hacia el servidor central (`resonance-core`):

* **Plano de Datos (*Data Plane* — Puerto TCP `1704`):** Conexión de socket binario persistente que recibe flujo de audio PCM sin comprimir (`44.1 kHz / 16-bit / Estéreo`), sincroniza el reloj monotónico local mediante un algoritmo tipo NTP y alimenta el hardware de salida (`AudioTrack`) con corrección de deriva en microsegundos.
* **Plano de Control (*Control Plane* — Puerto WebSocket `8000`):** Canal JSON bidireccional que recibe metadatos de la sesión activa (carátula, título, artista, progreso) y transmite comandos de gestión de habitaciones (volumen, muteo y compensación de latencia por nodo).

```text
                       [ Servidor Central: resonance-core ]
                         │                              │
            (TCP Binario :1704)                 (WebSockets :8000)
                         │                              │
                         ▼                              ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                           RESONANCE ANDROID APP                            │
│                                                                            │
│  ┌───────────────────────────────┐    ┌─────────────────────────────────┐  │
│  │     SnapcastTcpClient       │    │     BackendWebSocketClient      │  │
│  │  (Socket TCP + Little-Endian) │    │  (OkHttp WS + JSON Serializer)  │  │
│  └───────────────┬───────────────┘    └────────────────┬────────────────┘  │
│                  │                                     │                   │
│         ┌────────┴────────┐                            │                   │
│         ▼                 ▼                            ▼                   │
│  ┌─────────────┐   ┌─────────────┐            ┌─────────────────┐          │
│  │ NtpTimeSync │   │ RingBuffer  │            │ MultiroomRepo   │          │
│  │   Engine    │──►│  & Audio    │            │ (StateFlow Hub) │          │
│  └─────────────┘   │   Track     │            └────────┬────────┘          │
│                    └──────┬──────┘                     │                   │
│                           │                            ▼                   │
│                    ┌──────▼──────┐            ┌─────────────────┐          │
│                    │ OutputRoute │            │  Jetpack Compose│          │
│                    │   Monitor   │            │   UI (MVVM)     │          │
│                    └──────┬──────┘            └─────────────────┘          │
└───────────────────────────┼────────────────────────────────────────────────┘
                            │
              ┌─────────────┴─────────────┐
              ▼                           ▼
    [ Jack 3.5 mm (0 ms) ]     [ Bluetooth A2DP (-160 ms) ]
```

---

## 2. Stack Tecnológico y Requisitos del Sistema

* **Lenguaje:** Kotlin 2.0+ (Coroutines, `StateFlow`, `Channel`, `java.nio.ByteBuffer`).
* **Interfaz de Usuario:** Jetpack Compose + Material Design 3.
* **Patrón Arquitectónico:** MVVM (*Model-View-ViewModel*) con separación estricta entre Capa de Dominio, Capa de Datos/Red, Motor de Audio en Tiempo Real y Presentación.
* **SDK Mínimo (`minSdk`):** API 26 (Android 8.0 Oreo) — necesario para instanciar `AudioTrack` en modo de baja latencia (`PERFORMANCE_MODE_LOW_LATENCY`) y registrar canales de notificación para servicios en primer plano, manteniendo compatibilidad con dispositivos Android antiguos reutilizados.
* **SDK Objetivo (`targetSdk`):** API 35.

### Permisos Críticos (`AndroidManifest.xml`)
```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
<uses-permission android:name="android.permission.WAKE_LOCK" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK" />
<uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
<uses-permission android:name="android.permission.BLUETOOTH_CONNECT" />
```

---

## 3. Estructura de Paquetes (`com.axel.resonance`)

```text
app/src/main/java/com/axel/resonance/
├── ResonanceApp.kt
├── MainActivity.kt
│
├── domain/
│   ├── model/
│   │   ├── SnapPacket.kt
│   │   ├── TrackMetadata.kt
│   │   ├── RoomZone.kt
│   │   └── SyncStats.kt
│   └── repository/
│       └── MultiroomRepository.kt
│
├── data/
│   ├── local/
│   │   └── NodePreferences.kt
│   └── network/
│       ├── binary/
│       │   ├── SnapcastProtocol.kt
│       │   └── SnapcastTcpClient.kt
│       └── control/
│           └── BackendWebSocketClient.kt
│
├── audio/
│   ├── sync/
│   │   └── NtpTimeSyncEngine.kt
│   ├── output/
│   │   ├── LowLatencyAudioPlayer.kt
│   │   └── OutputRouteMonitor.kt
│   └── service/
│       └── AudioPlayerService.kt
│
└── ui/
    ├── theme/
    │   ├── Color.kt
    │   ├── Theme.kt
    │   └── Type.kt
    ├── viewmodel/
    │   └── MultiroomViewModel.kt
    ├── components/
    │   ├── NowPlayingBanner.kt
    │   ├── ZoneControlCard.kt
    │   ├── NodeStatusSwitch.kt
    │   └── TelemetryBadge.kt
    └── screens/
        ├── DashboardScreen.kt
        └── SettingsScreen.kt
```

---

## 4. Especificación Detallada por Capa

### 4.1 Nivel Raíz
* **`ResonanceApp.kt`:** Inicializa el grafo de dependencias de la aplicación y crea el `NotificationChannel` requerido para el servicio en primer plano en Android 8.0+.
* **`MainActivity.kt`:** Solicita en tiempo de ejecución los permisos `POST_NOTIFICATIONS` y `BLUETOOTH_CONNECT`, vincula (*bind*) la UI con `AudioPlayerService` y aloja el contenedor de navegación Compose.

### 4.2 Capa de Dominio (`domain/`)
* **`SnapPacket.kt`:** Jerarquía `sealed class` que modela los paquetes del protocolo binario:
  * `Hello`: Handshake inicial JSON con UUID persistente del dispositivo.
  * `CodecHeader`: Parámetros del flujo PCM (`44100 Hz`, `16-bit`, `2` canales).
  * `WireChunk`: Carga útil de muestras PCM junto con su timestamp de captura en servidor.
  * `ServerSettings`: Volumen remoto, estado de muteo, `bufferMs` y `latencyOffsetMs`.
  * `Time`: Paquete de intercambio de marcas de tiempo (`sent` / `received`).
* **`TrackMetadata.kt`:** Estado inmutable de la pista actual (título, artista, álbum, URL de carátula, `durationMs`, `positionMs` e `isPlaying`).
* **`RoomZone.kt`:** Representación de cada nodo activo en la red (`id`, `name`, `volume`, `isMuted`, `latencyOffsetMs`, `isOnline`, `outputType`).
* **`SyncStats.kt`:** Telemetría local en vivo (`clockOffsetMs`, `jitterMs`, salud del buffer circular y ruta de salida física).
* **`MultiroomRepository.kt`:** Única fuente de verdad (*Single Source of Truth*) que combina los flujos locales de telemetría con los eventos remotos del WebSocket.

### 4.3 Capa de Datos y Red (`data/`)
* **`NodePreferences.kt`:** Persistencia reactiva mediante `Jetpack DataStore`. Almacena la IP del servidor, UUID único del nodo, nombre de la habitación y dos perfiles independientes de compensación de latencia: `wiredOffsetMs` (`0 ms` por defecto) y `bluetoothOffsetMs` (`-160 ms` por defecto para compensar el retardo A2DP de bocinas Bluetooth).
* **`SnapcastProtocol.kt`:** Parser y serializador binario en orden `ByteOrder.LITTLE_ENDIAN`. Lee exactamente los **26 bytes** de cabecera fija (`type` [2B], `id` [2B], `refersTo` [2B], `sent_sec` [4B], `sent_usec` [4B], `received_sec` [4B], `received_usec` [4B], `size` [4B]) antes de extraer el payload dinámico.
* **`SnapcastTcpClient.kt`:** Administra el `Socket` TCP hacia el puerto `1704` en `Dispatchers.IO` con `tcpNoDelay = true` (desactiva el algoritmo de Nagle), envía el handshake `Hello`, enruta paquetes `Time` y `WireChunk`, y aplica reconexión automática con *backoff* exponencial (`1s → 2s → 4s → 8s → máx. 15s`).
* **`BackendWebSocketClient.kt`:** Cliente `OkHttp` WebSocket conectado a `ws://<SERVER_IP>:8000/ws` para recibir actualizaciones push de metadatos/zonas y emitir comandos JSON (`SET_VOLUME`, `TOGGLE_MUTE`, `SET_LATENCY_OFFSET`, `RENAME_CLIENT`).

### 4.4 Motor de Audio y Sincronización (`audio/`)
* **`NtpTimeSyncEngine.kt`:** Envía paquetes `Time` cada `1000 ms` registrando $t_1$ con `System.nanoTime()`. Con los timestamps del servidor ($t_2, t_3$) y de recepción local ($t_4$), calcula el desfase de reloj:
  $$\text{offset} = \frac{(t_2 - t_1) + (t_3 - t_4)}{2}$$
  Aplica un filtro de mediana sobre una ventana deslizante de 30 muestras para eliminar picos de *jitter* Wi-Fi y mapear cada `WireChunk` al reloj monotónico local.
* **`LowLatencyAudioPlayer.kt`:** Gestiona `android.media.AudioTrack` (`MODE_STREAM`, `USAGE_MEDIA`, `PERFORMANCE_MODE_LOW_LATENCY`). Consume un buffer circular (*RingBuffer*) desde un hilo dedicado (`Process.THREAD_PRIORITY_URGENT_AUDIO`) ejecutando **corrección de deriva (*Drift Correction*)**: descarta muestras si el atraso supera `5 ms` o inserta silencio mínimo si el flujo va adelantado.
* **`OutputRouteMonitor.kt`:** Escucha eventos del sistema mediante `AudioDeviceCallback` para detectar transiciones entre salida cableada (`TYPE_WIRED_HEADPHONES` / `TYPE_LINE_ANALOG`) y Bluetooth A2DP (`TYPE_BLUETOOTH_A2DP`), aplicando automáticamente el perfil de latencia almacenado.
* **`AudioPlayerService.kt`:** `ForegroundService` (`FOREGROUND_SERVICE_MEDIA_PLAYBACK`) que mantiene activo el pipeline de audio con la pantalla apagada reteniendo `PowerManager.PARTIAL_WAKE_LOCK` y `WifiManager.WifiLock` (`WIFI_MODE_FULL_LOW_LATENCY`).

### 4.5 Capa de Presentación (`ui/`)
* **`MultiroomViewModel.kt`:** Expone `StateFlow<MultiroomUiState>`, interpola localmente el avance de la barra de progreso cada segundo mientras `isPlaying == true` y despacha las acciones de usuario hacia el repositorio.
* **`NowPlayingBanner.kt`:** Muestra carátula (Coil), metadatos de la pista activa y progreso en tiempo real.
* **`ZoneControlCard.kt`:** Control por habitación con estado de conexión, slider de volumen, botón de silencio y calibrador fino de latencia (`-500 ms` a `+500 ms`).
* **`NodeStatusSwitch.kt`:** Interruptor para activar o detener `AudioPlayerService` localmente (permite usar un teléfono como control remoto puro sin reproducir audio).
* **`TelemetryBadge.kt`:** Indicador técnico en tiempo real con ruta física activa (`Jack 3.5mm` / `Bluetooth`), desfase NTP (`ms`) y jitter de red.
* **`DashboardScreen.kt` & `SettingsScreen.kt`:** Vistas principales construidas en Jetpack Compose para operación diaria y configuración de red/latencias.