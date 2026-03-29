# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Frixos is ESP-IDF firmware for an ESP32-based daylight projection clock. It drives an ST7735S 128×128 LCD via LVGL, serves a captive-portal settings web UI from SPIFFS, and integrates with OpenWeatherMap, Home Assistant, Dexcom/Freestyle Libre/Nightscout (CGM), and Finnhub stocks.

## Build System

This is an ESP-IDF CMake project targeting ESP32. **Use the Podman container for all builds** — it uses the same environment as CI and avoids local toolchain issues.

### Podman build (preferred)

```bash
podman run --rm -v "$PWD:/project:Z" -w /project espressif/idf:v5.4 idf.py build
```

### Local build (alternative)

```bash
source ~/.espressif/tools/activate_idf_v6.0.sh
idf.py build
```

### One-time setup

```bash
cp env.example .env   # then fill in WEATHER_API_KEY and any optional defaults
```

### Other useful commands

```bash
idf.py -p /dev/tty.usbserial-* flash monitor
idf.py update-dependencies   # after editing main/idf_component.yml
idf.py reconfigure            # after sdkconfig changes
idf.py fullclean              # remove all build artifacts
```

`config.h` is auto-generated from `.env` by the top-level `CMakeLists.txt` before compilation — never edit `main/include/config.h` directly.

### NVS and flashing

NVS (Non-Volatile Storage) is **erased on every `idf.py flash`**. To avoid re-entering settings after each flash, add developer-specific defaults to `.env`:

| .env key | Setting |
|----------|---------|
| `LATITUDE` / `LONGITUDE` / `TIMEZONE` | Location |
| `HA_URL` / `HA_TOKEN` | Home Assistant |

These pre-populate NVS on first boot. Web UI changes persist and take precedence. OTA updates preserve NVS.

## Hardware

- **MCU**: ESP32 (240 MHz, 8 MB flash)
- **Display**: ST7735S 128×128 via SPI (`LCD_SPI_NUM`)
- **Light sensor**: LTR303 on I2C (GPIO 21/22)
- **Backlight**: PWM via `esp_driver_ledc`
- **Partition layout**: OTA dual-slot (app0/app1 @ 3.2 MB each) + SPIFFS (1.5 MB @ `/spiffs`) + coredump

## Key sdkconfig Settings

- Flash: 8 MB, custom partition table (`partitions.csv`)
- BT: NimBLE host only (`BT_CONTROLLER_DISABLED`, `BT_NIMBLE_MEM_ALLOC_MODE_DEFAULT`)
- LVGL FS: POSIX driver enabled, drive letter `S` (83), mount point `/spiffs`
- WiFi SW coexistence: disabled

## Architecture

Two RTOS tasks run after boot:

- **`display_task`** (core 1, 10 KB stack) — Owns the LVGL tick, renders digits/weather/scroll messages, drives backlight via LTR303 readings.
- **`wifi_task`** (core 0, 8 KB stack) — WiFi connect/reconnect, NTP sync, OpenWeatherMap fetch, external API polling (integrations), mDNS.

Settings are persisted in NVS under the `"frixos"` namespace. The HTTP server (in `f-settings.c`) runs on both cores and serves the single-page web UI from SPIFFS along with a REST API (`/api/settings`, `/api/status`, `/api/log`, `/api/update`).

### Module responsibilities

| File | Responsibility |
|------|---------------|
| `main.c` | Boot, NVS init, SPIFFS mount, task creation, circular log buffer |
| `f-display.c` | LVGL setup, digit rendering, font caching, scroll messages, weather icons, moon phase |
| `f-wifi.c` | WiFi connect, NTP, weather fetch, HTTP event handler, mDNS |
| `f-settings.c` | HTTP server, settings GET/POST endpoints, OTA upload handler, web UI serving |
| `f-integrations.c` | Token substitution in scroll messages; orchestrates Dexcom/Freestyle/Nightscout/Finnhub/HA polls |
| `f-provisioning.c` | SoftAP mode, captive portal DNS, QR code display |
| `f-ota.c` | OTA firmware update (dual-slot with rollback) |
| `ltr303.c` | I2C ambient light sensor for auto-dimming |
| `f-membuffer.c` | Shared HTTP response buffer (call `get_shared_buffer` / `release_shared_buffer`) |
| `frixos_*.c`, `font-pixelate_10.c` | Bitmap font glyph data (sizes 8–14 px) |

### Display font caching

`f-display.c` maintains a font cache keyed by pixel size. Custom fonts are defined in `frixos_N.c` files and registered with LVGL. Use the existing cache API rather than loading fonts directly.

### Scroll message tokens

`f-integrations.c` replaces `{token}` placeholders in the scroll message string with live data from Home Assistant entities, CGM readings, or stock quotes. Token resolution runs on the wifi_task poll cycle.

## IDF v6.0 Migration Notes

Several ESP-IDF v5→v6 API renames already applied in this codebase:

- `wifi_provisioning` → `network_provisioning` (component name)
- `json` → `cjson` (component REQUIRES)
- `esp_lcd_panel_dev_config_t.color_space` → `.rgb_ele_order` (`LCD_RGB_ELEMENT_ORDER_BGR`)
- `ESP_LCD_COLOR_SPACE_BGR` → `LCD_RGB_ELEMENT_ORDER_BGR`
- `LCD_RGB_ENDIAN_RGB/BGR` → `LCD_RGB_DATA_ENDIAN_BIG/LITTLE`
- `rgb_endian` field → `data_endian` field
- `WIFI_BW_HT20` → `WIFI_BW20`
- `HTTP_EVENT_ON_HEADERS_COMPLETE`, `HTTP_EVENT_ON_STATUS_CODE` are new enum values that must be handled in HTTP event switches
