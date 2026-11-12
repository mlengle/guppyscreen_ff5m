# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A fork of [ballaswag/guppyscreen](https://github.com/ballaswag/guppyscreen) (LVGL touch UI for Klipper, talks to Moonraker over its websocket JSON-RPC API) adapted for **zMod** on Flashforge printers (Adventurer 5M / AD5X). The upstream README, DEVELOPMENT.md, installers (`installer*.sh`, `update.sh`, `reinstall-creality.sh`), `k1/` and `.github.old/` (upstream CI, disabled) describe the Creality K1 world and are largely irrelevant to the FF5M build.

## Build

Submodules (`lvgl`, `lv_drivers`, `libhv`, `spdlog`) must be checked out (`git submodule update --init --recursive`) and patched before building:

```sh
(cd lv_drivers && git apply ../patches/0001-lv_driver_fb_ioctls_flashforge_ad5m.patch)  # or ..._ad5x.patch (backlight ioctls disabled)
(cd spdlog     && git apply ../patches/0002-spdlog_fmt_initializer_list.patch)
(cd lvgl       && git apply ../patches/0003-lvgl-dpi-text-scale.patch)
(cd libhv      && git apply ../patches/0004-fix-cppflags-override.patch)
(cd libhv      && git apply ../patches/0005-libhv-websocket-release-big-message-buffer.patch)
(cd lvgl       && git apply ../patches/0006-lvgl-draw-indexed-8bit-decoded-images.patch)
(cd lvgl       && git apply ../patches/0007-lvgl-draw-indexed-8bit-images-whole.patch)
```

Key make variables (see `Makefile`):
- `FF5M=1` — Flashforge build: defines `GUPPY_FF5M`, adds `--sysroot=$(SYSROOT)`, links `-lts` (tslib), passes the cross compilers to spdlog's cmake.
- `CROSS_COMPILE=<prefix>-` — target toolchain. **If unset, the build is a desktop SDL2 simulator** (`-D SIMULATOR`, links `-lSDL2`).
- `SYSROOT` — required with `FF5M`.
- `GUPPYSCREEN_VERSION`, `GUPPY_THEME=zbolt`, `GUPPY_SMALL_SCREEN`, `GUPPY_ROTATE`, `EVDEV_CALIBRATE` — optional upstream knobs.

```sh
make build            # full: cleans and rebuilds libhv.a, libspdlog.a, then guppyscreen
make -j$(nproc)       # incremental rebuild of src/ after an initial `make build`
make kd_graphic_mode  # helper binary
```

Output: `build/bin/guppyscreen`. There is no linter.

## Tests

`make test` builds the app for the PC and runs the integration tests in `tests/`, each test case in its own process, several at a time. `make test TEST='*name*'` runs a subset; `SANITIZE=1` adds ASan and UBSan. The tests run the real app (all of `src/` except `main.cpp`). Only external systems are faked: Moonraker (a WebSocket server answering from `tests/fixtures/*.json`), the display and touch, and the printer's file paths. See `docs/testing.md` for the harness API (`app::`, `ui::`, `FakeMoonraker`) and how to add tests. Coverage is deliberately minimal: when refactoring a code path, first add a smoke test for it. Then check the test fails when that path is broken. Final verification still means running on the printer.

Note: `src/main.cpp` includes `<tslib.h>` unconditionally, so tslib headers are needed even for simulator builds.

Cross-building for the printer: see `docs/build-toolchain.md`. It covers the Ubuntu GCC 13 toolchain, `env.sh`, and the compiler wrappers. The wrappers force 32-bit `time_t` to match zMod's `libts`. Always build through them. Quick form: `. <xtools>/env.sh && make -j$(nproc) CROSS_COMPILE=$CROSS_COMPILE FF5M=1 SYSROOT=$SYSROOT GUPPY_SMALL_SCREEN=1 GUPPYSCREEN_VERSION=0.0.30.1-zmod+<hash>`. zMod builds with `GUPPY_SMALL_SCREEN` (DPI 90); without it the UI is scaled differently.

## Architecture

Full overview: `docs/guppy-arch.md` (threads, startup sequence, message dispatch, `State`, panel tree, memory-relevant facts). Summary:

- **Entry** (`src/main.cpp`): sets up gettext, loads `guppyconfig.json` from the executable's directory, initialises display (fbdev) and touch (tslib, non-blocking) via `hal_init`, then runs `GuppyScreen::loop()`.
- **`GuppyScreen`** (`src/guppyscreen.cpp`): singleton owning the global `KWebSocketClient ws`, the LVGL mutex `lv_lock`, logging, theme (from `themes/<name>.json`), screen-saver/backlight sleep, and the top-level panels (`SpoolmanPanel`, `MainPanel`, `InitPanel`). `loop()` runs `lv_timer_handler()` under `lv_lock` every 5 ms.
- **Threading rule**: Moonraker callbacks run on libhv's event-loop thread, not the LVGL thread. Any code touching LVGL objects from a websocket callback or `consume()` must hold `lv_lock` (`std::lock_guard<std::mutex> lock(lv_lock);`).
- **Moonraker I/O** (`src/websocket_client.*`): `send_jsonrpc(method, params, cb)` for request/response; `gcode_script(gcode)` to run G-code; `register_notify_update(NotifyConsumer*)` to receive every `notify_status_update`; `register_method_callback(method, handler_name, cb)` for other notification methods.
- **State** (`src/state.*`): singleton `NotifyConsumer` that mirrors the subscribed Klipper object tree under `/printer_state/...`; read with `State::get_instance()->get_data("/printer_state/<obj>/<field>"_json_pointer)`. Panels typically read the initial value from `State` and then incremental updates from the notification JSON (`/params/0/<obj>/<field>`) in their own `consume()`.
- **Startup** (`src/init_panel.cpp`, `InitPanel::connected`): resets `State`, calls `printer.objects.list`, fetches roots/info/Moonraker DB items into `State`, calls `main_panel.subscribe()`, creates sensor/fan/LED widgets, then `printer.objects.subscribe`s to every object except `gcode_macro *`; the subscribe result seeds `/printer_state` and calls `main_panel.init(data)`.
- **Panels**: one class per screen (`*_panel.cpp/h`), composed from reusable widgets (`button_container`, `slider_container`, `switch_container`, `sensor_container`, `image_label`, `selector`, `numpad`, ...). `MainPanel` owns most sub-panels (tune, print, extruder, fans, ...).
- **Config** (`src/config.cpp`): JSON at `<exe dir>/guppyconfig.json`, created with FF5M defaults if missing; per-printer settings live under `Config::df()` (`/printers/<default_printer>/`).

## FF5M / zMod specifics

- Everything Flashforge-specific is behind `#ifdef GUPPY_FF5M` / `#ifndef GUPPY_FF5M` (wifi/wpa_supplicant, printer selection, several sysinfo/setting/print/tune features are compiled out). Keep upstream code paths intact behind these guards rather than deleting them.
- Multi-tool (AD5X "T0–T3") support reads zMod's `zmod_color` Klipper object (`total_tools`, `active_tool_id`); extruders map to `extruder`, `extruder1`, ... (see `extruder_panel.cpp`, `pid_panel.cpp`, `print_status_panel.cpp`).
- Touch device is `/dev/input/event4` under `GUPPY_FF5M` (`lv_drv_conf.h`); thumbnails go to `/usr/data/printer_data/thumbnails`.
- UI images in `assets/material/*.c` are LVGL C image arrays, stored as `LV_IMG_CF_INDEXED_8BIT` (5 KB instead of 16 KB; drawn whole and zoomable thanks to patches 0006/0007). Run `assets/icons_to_indexed.py <file.c>` on any icon added from LVGL's converter (true colour + alpha). `assets/icons_to_png.py <out dir> <files.c>` exports icons as PNGs to look at. `assets/material_46` and `assets/zbolt` are upstream-only sets, not converted. Unused ones are parked in `assets/*/rem/` (not compiled, since the Makefile only globs the top level). Fonts `assets/notosemi_{16,20}.c` are generated by `assets/fonts/build.sh` (Docker + `lv_font_conv`); add new glyph codepoints (e.g. Cyrillic, FontAwesome symbols) to its `--range` lists.

## Localization

User-visible strings are wrapped in `_("...")` (gettext, macro in `websocket_client.h`). Text domain `guppyscreen`, loaded at runtime from `/usr/data/zmod/zmod/.shell/locale`. After adding/changing strings, regenerate catalogs:

```sh
cd src/locale && ./Make_pot   # xgettext -> guppyscreen.pot, msgmerge into each <lang>/LC_MESSAGES/guppyscreen.po, msgfmt -> .mo
```

Both `.po` and compiled `.mo` files are committed. Languages: ru, en, de, it, fr, es, zh, ja, ko, pt, cs, tr.
