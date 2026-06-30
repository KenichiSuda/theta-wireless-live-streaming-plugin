# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is an Android plugin for RICOH THETA cameras (V, Z1, X) that enables live 360° video streaming via RTMP to YouTube or any RTMP endpoint. It runs as a THETA plugin — a full Android app that runs directly on the camera hardware.

## Build Commands

```bash
# Build debug APK
./gradlew assembleDebug

# Build release APK
./gradlew assembleRelease

# Clean build outputs
./gradlew clean

# Run unit tests
./gradlew test

# Run instrumented tests (requires connected device)
./gradlew connectedAndroidTest

# Install on connected THETA device
adb install -r app/build/outputs/apk/debug/wirelessLiveStreaming_v1.2.3_debug.apk
```

The output APK is named `wirelessLiveStreaming_v{versionName}_{variant}.apk`.

After installing, grant the plugin permissions via Settings > Apps > Wireless Live Streaming > Permissions on the device (viewable through Vysor or similar).

## Module Structure

```
app/                        — Main plugin application (com.theta360.cloudstreaming)
pedroLibrary/
  encoder/                  — Hardware video/audio encoder (H264/AAC)
  rtmp/                     — RTMP protocol implementation
  rtsp/                     — RTSP protocol implementation
  rtplibrary/               — High-level streaming API (RtmpCamera1)
pluginsdk/
  pluginlibrary/            — RICOH THETA plugin SDK (LED control, key callbacks, OLED)
libs/
  XCamera.jar               — THETA X camera control (binary, not source)
```

The `settings.gradle` wires these together: `app` depends on `encoder`, `rtmp`, `rtplibrary`, and `pluginlibrary`.

## Architecture

**Main flow:** `MainActivity` extends `PluginActivity` (THETA SDK) and implements `ConnectCheckerRtmp` (streaming callbacks). On start it:
1. Launches `AndroidWebServer` (NanoHTTPD on port 8888) serving the web configuration UI from `app/src/main/assets/`
2. Starts background threads: `SettingPolling` (polls SQLite for config changes every 1s), `ShutDownTimer` (auto-exits after no-operation timeout), `ScheduleStreaming` (serializes start/stop calls)
3. Streams via `RtmpCamera1` from `pedroLibrary/rtplibrary`, rendering through an `OpenGlView`

**Settings persistence:** All streaming settings (server URL, stream key, resolution, bitrate, audio rate, timeout) are stored in SQLite via `Theta360SQLiteOpenHelper`. The stream key is AES-encrypted before storage — `AndroidWebServer.encodeStreamName()` / `decodeStreamName()` handle this.

**Web UI:** The browser UI at `http://<camera-ip>:8888` is served from `assets/`. Multiple HTML variants exist for different contexts:
- `index.html` / `index_sp.html` — THETA V (desktop/mobile)
- `index_z1.html` — THETA Z1
- `index_x.html` / `index_x_v012000.html` — THETA X (with/without 15fps options added in firmware 1.20.0)

**Device branching:** `ThetaModel.isVCameraModel()` distinguishes THETA V/Z1 from THETA X. `ThetaInfo.getThetaFirmwareVersion()` is used to enable features gated on firmware version (e.g., 15fps modes for X firmware ≥ 1.20.0). LED and OLED display calls differ between models — check these branches when modifying any status indicator logic.

**Physical controls:** The shutter button (KEYCODE_CAMERA) toggles streaming. Long-press of the record button exits the plugin on THETA V, or stops streaming and closes on THETA X. Power button (`KEYCODE_POWER`) also stops streaming.

**Streaming lifecycle:** Start/stop goes through `ScheduleStreaming.setSchedule()` → `streaming()` → `startStreaming()` / `stopStreaming()`. The `SettingPolling` thread is paused during active streaming and resumed on stop. A `DelayJudgment` thread monitors streaming delay post-start and adjusts LED indicators.

**Logging:** Timber is used throughout. Daily log files are written to `context.getFilesDir()` and auto-deleted after 30 days.

## Key Files

| File | Purpose |
|------|---------|
| `app/src/main/java/com/theta360/cloudstreaming/MainActivity.java` | Entry point, streaming orchestration, button handling |
| `app/src/main/java/com/theta360/cloudstreaming/httpserver/AndroidWebServer.java` | NanoHTTPD web server, REST API for Web UI, stream key encryption |
| `app/src/main/java/com/theta360/cloudstreaming/httpserver/Theta360SQLiteOpenHelper.java` | SQLite schema for settings |
| `app/src/main/java/com/theta360/cloudstreaming/settingdata/Bitrate.java` | Resolution/bitrate/fps constants |
| `app/src/main/java/com/theta360/cloudstreaming/settingdata/SettingData.java` | Settings POJO |
| `app/src/main/assets/` | Web UI HTML/CSS/JS files |

## SDK Versions

- compileSdkVersion: 29
- minSdkVersion: 25 (required by THETA plugin SDK)
- targetSdkVersion: 25 (pluginlibrary) / 29 (app)
- Gradle Plugin: 4.2.2 / Gradle wrapper: 6.7.1
- Android Support: 27.1.1 (`ANDROID_SUPPORT_VERSION` in `gradle.properties`)
