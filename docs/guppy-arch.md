# guppyscreen (FF5M fork): architecture overview

guppyscreen is a touch UI for Klipper written with LVGL 8. It talks to Moonraker over Moonraker's WebSocket JSON-RPC API. This fork targets zMod on Flashforge printers (Adventurer 5M / AD5X). It runs in the zMod chroot as `/root/guppyscreen/guppyscreen`, started by `S80guppyscreen`, and draws directly to `/dev/fb0`.

```
 touchscreen ──tslib──▶ ┌──────────────────────────── guppyscreen ───────────────────────────┐
                        │ main thread (LVGL)                │ libhv event-loop thread        │
 /dev/fb0 ◀──fbdev───── │  GuppyScreen::loop()              │  KWebSocketClient (ws)         │
                        │   lv_timer_handler() every 5 ms   │   json::parse(msg)             │
                        │   under lv_lock                   │   ├─ reply callbacks (by id)   │
                        │  display sleep / screen saver     │   ├─ notify consumers          │
                        │                                   │   └─ method callbacks          │
                        │  panels and widgets (lv_obj tree) │◀── take lv_lock before touching│
                        │                                   │    LVGL objects                │
                        │          State (nlohmann::json mirror of Klipper objects)          │
                        └────────────────────────────────────────────────────────────────────┘
                                                   ▲ ws://<moonraker_host>:<port>/websocket
                                                   ▼
                                     Moonraker ──▶ Klipper (+ zMod objects)
```

## Source layout

| Path | Contents |
|---|---|
| `src/*.cpp`, `*.h` | the application: one class per screen (`*_panel`) plus reusable widgets |
| `lvgl/` (submodule, patched) | LVGL 8 |
| `lv_drivers/` (submodule, patched) | fbdev display driver; the patches change the framebuffer and backlight ioctls for AD5M/AD5X |
| `libhv/` (submodule, patched) | networking library providing the event loop and `WebSocketClient` |
| `spdlog/` (submodule, patched) | logging |
| `lv_touch_calibration/` | touch calibration screen (the calibration code is commented out under FF5M) |
| `assets/` | fonts (`notosemi_16/20.c`, `materialdesign_font_40.c`, `dejavusans_mono_14.c`) and LVGL C-array images (`assets/material/*.c`); unused images are parked in `rem/` subfolders |
| `themes/` | colour themes (`<name>.json`) |
| `src/locale/` | gettext catalogs (`.po` / `.mo`) |
| `lv_conf.h`, `lv_drv_conf.h` | LVGL and driver configuration |
| `patches/` | patches applied to the submodules before building |

See `build-toolchain.md` for building and `ram-optimization.md` for the memory work.

## Threads and locking

There are two threads that matter:

1. **Main (LVGL) thread.** `GuppyScreen::loop()` runs `lv_timer_handler()` under `lv_lock`, then sleeps 5 ms. It also handles display sleep: after `display_sleep_sec` of no touch input it blanks the framebuffer (`fbdev_blank()`) and raises a black `screen_saver` object; a touch wakes it. All drawing, input and LVGL timers run here.
2. **libhv event-loop thread.** `GuppyScreen::ws` is created with a `NULL` loop, so libhv starts its own event-loop thread. All WebSocket I/O runs there, including JSON parsing and **every callback and `consume()` call**.

**Rule:** code running on the event-loop thread must hold `lv_lock` (`std::lock_guard<std::mutex> lock(lv_lock);`) before touching any LVGL object. `State` has its own mutex (`State::lock`) for the JSON tree. `lv_lock_update` guards the sleep and wake flags.

Note that `State::get_data()` returns a *reference* into the tree. The caller uses it after `State`'s lock has been released, so concurrent updates are a known race (see the TODO in `macros_panel.cpp`).

## Startup sequence

1. **`main()`** (`src/main.cpp`)
   - Sets up gettext with domain `guppyscreen` from `/usr/data/zmod/zmod/.shell/locale`.
   - Loads `guppyconfig.json` from the executable's directory through `Config` (created with FF5M defaults if missing). The thumbnail directory is `/usr/data/printer_data/thumbnails`.
   - Calls `GuppyScreen::init(hal_init)`, then `loop()`.
2. **`GuppyScreen::init`** (`src/guppyscreen.cpp`)
   - Logging: stdout plus a rotating file in `/opt/config/mod_data/log/`.
   - Loads the theme and calls `lv_init()`, then `fbdev_init()`.
   - `hal_init` sets up the draw buffers, the display driver (optional software rotation via `display_rotate`) and the default theme. The font is `notosemi_16` when the screen is at most 480 px tall, otherwise `notosemi_20`. It also sets up the touch input: tslib, non-blocking.
   - Initialises the PNG decoder and the custom styles and theme hook.
   - Creates the screen saver.
   - Registers `State` as a notify consumer.
   - Connects the WebSocket to `moonraker_host:moonraker_port`.
   - The constructors of the top-level members `SpoolmanPanel`, `MainPanel` and `InitPanel` build **the whole LVGL object tree for every panel up front**.
3. **`InitPanel::connected`** (`src/init_panel.cpp`). Runs on connect and on every `notify_klippy_ready`:
   1. Resets `State` and calls `printer.objects.list`.
   2. Fetches `printer.info`, `server.info` and Moonraker database items (e.g. `guppysettings`, console history) into `State`.
   3. Calls `main_panel.subscribe()`.
   4. Creates the sensor, fan and LED widgets from the objects that exist and the `monitored_sensors` / fan / LED settings in the config.
   5. Calls `printer.objects.subscribe` for **every object except `gcode_macro *`**, with all fields.
   6. The subscribe reply seeds `/printer_state`, calls `main_panel.init(data)` (which also builds the macros list) and `bedmesh_panel.refresh_views_with_lock(...)`, then hides the init overlay.
4. **`InitPanel::disconnected`** shows the overlay again ("Waiting for the Klipper launch..."). libhv reconnects automatically with a delay of 200 ms–2 s and sends a ping every 10 s.

## Moonraker I/O (`src/websocket_client.*`)

`KWebSocketClient` extends `hv::WebSocketClient`. Each incoming message is parsed once into `json`, then dispatched:

- **Replies** (have an `id`): the one-shot `consumers[id]` (a `NotifyConsumer*`) or `callbacks[id]` (a `std::function`) is called, then erased.
- **`notify_status_update` and `notify_filelist_changed`**: passed to every registered notify consumer, each consumer's `consume(json&)`.
- **`notify_klippy_ready` / `notify_klippy_disconnected`**: trigger the connected / disconnected handlers.
- **Any method**: also passed to the callbacks registered with `register_method_callback(method, handler_name, cb)`, e.g. `notify_gcode_response` for the console and prompts.

Sending:
- `send_jsonrpc(method[, params][, cb | consumer])` sends a request and registers its reply handler.
- `gcode_script(gcode)` wraps `printer.gcode.script`.

User strings use `_("...")` (gettext macro defined in `websocket_client.h`).

## State (`src/state.*`)

`State` is a singleton `NotifyConsumer` holding a single `nlohmann::json` tree:

| Key | Contents |
|---|---|
| `/printer_state/<object>/<field>` | mirror of every subscribed Klipper object, merged with `merge_patch` on each `notify_status_update` |
| `/printer_objs/objects` | result of `printer.objects.list`; the `get_extruders()`, `get_heaters()`, `get_sensors()`, `get_fans()`, `get_leds()` and `get_output_pins()` helpers filter it by name prefix |
| `/guppysettings/...`, `/console/...` | Moonraker database items |

Typical panel pattern:
1. Read the initial value from `State` (`State::get_instance()->get_data("/printer_state/x/y"_json_pointer)`).
2. Register as a notify consumer (`ws.register_notify_update(this)`).
3. In `consume(json &j)`, read the incremental updates (`j["/params/0/x/y"_json_pointer]`) and update widgets under `lv_lock`.

## UI structure

Top-level objects are members of `GuppyScreen`, built in its constructor. Indentation shows ownership by value; `&` marks a reference to a panel owned elsewhere.

```
GuppyScreen
├─ SpoolmanPanel
├─ InitPanel                  connection overlay; drives the startup sequence
└─ MainPanel                  tabview, temperature chart, main buttons
   ├─ Main tab: SensorContainer per heater/sensor, ButtonContainers
   │  ├─ HomingPanel, ExtruderPanel, FanPanel, LedPanel
   │  ├─ PrintPanel ── FilePanel (file browser + thumbnails)
   │  └─ PrintStatusPanel ── FineTunePanel, RetractPanel, ExcludePanel,
   │                        ProPanel, PidPanel   (+ MiniPrintStatus)
   ├─ Tune tab: PrinterTunePanel ── BedMeshPanel, &FineTunePanel, LimitsPanel,
   │            RetractPanel*, ProPanel*, PidPanel*, InputShaperPanel,
   │            BeltsCalibrationPanel  [Tmc*/Power: not under FF5M]
   ├─ Macros tab: MacrosPanel ── MacroItem per macro
   ├─ Console tab: ConsolePanel  (≤200 output lines, 20 history entries)
   ├─ Settings tab: SettingPanel ── SysInfoPanel  [Wifi/PrinterSelect: not under FF5M]
   └─ PromptPanel                Klipper action:prompt dialogs
```

\* In `PrinterTunePanel`, `RetractPanel`, `ProPanel` and `PidPanel` are by-value **copies** of the ones owned by `PrintStatusPanel`. They share the same LVGL objects (no extra widgets), but only the originals receive status updates and events, and both copies' destructors delete the same widgets. See `NOTES.md`.

Sub-panels are usually built once and shown with `foreground()` (`lv_obj_move_foreground` or clearing `LV_OBJ_FLAG_HIDDEN`), then hidden with a back button. Exceptions that build or rebuild content at runtime:
- `MacrosPanel::populate` on every connect;
- sensor, fan and LED widgets on connect;
- file lists and thumbnails while browsing;
- console lines.

Reusable widgets: `ButtonContainer`, `SliderContainer`, `SwitchContainer`, `SensorContainer`, `ImageLabel`, `Selector`, `SpinboxSelector`, `Numpad`, `Tree`.

## Configuration (`src/config.cpp`)

The config lives at `<exe dir>/guppyconfig.json`; zMod ships variants such as `guppyconfig_en.json` / `_ru` / `_pro`.
- Global keys include `display_sleep_sec`, `display_rotate`, `thumbnail_path`, log level and path.
- Per-printer keys live under `Config::df()` = `/printers/<default_printer>/`: `moonraker_host`, `moonraker_port`, `monitored_sensors`, fans, LEDs, and so on.

## Platform guards

| Define | Meaning |
|---|---|
| `GUPPY_FF5M` | Flashforge build. Compiles out wifi / `wpa_supplicant`, printer selection, TMC and power panels, and some sysinfo / settings / print / tune features. Uses tslib touch input (`/dev/input/event4` default; `S80guppyscreen` sets `TSLIB_TSDEVICE`). |
| `SIMULATOR` | Desktop build with SDL2; set when `CROSS_COMPILE` is unset. `main.cpp` still includes `tslib.h`. |
| `OS_ANDROID`, `ZBOLT`, `GUPPY_SMALL_SCREEN`, `GUPPY_ROTATE`, `EVDEV_CALIBRATE` | upstream options, mostly unused here |

Keep upstream code paths behind these guards rather than deleting them.

## zMod integration

- **Multiple tools (AD5X T0–T3).** Read from zMod's `zmod_color` object (`total_tools`, `active_tool_id`). Extruders map to `extruder`, `extruder1`, ...; see `extruder_panel`, `pid_panel` and `print_status_panel`.
- **Calibration graphs.** zMod scripts write PNGs to `/opt/config/mod_data/` (`resonances_x/y.png`, `belts_calibration.png`). `InputShaperPanel` and `BeltsCalibrationPanel` display them through LVGL's PNG decoder, which goes through the image cache.
- **Thumbnails.** For a G-code file, the thumbnail closest to 300 px wide is chosen from the Moonraker metadata (`KUtils::get_thumbnail`) and loaded from the thumbnail directory.
- **Locale** files come from `/usr/data/zmod/zmod/.shell/locale`. Rebuild the catalogs with `src/locale/Make_pot`.

## Memory-relevant facts

Details and the plan are in `ram-optimization.md`.
- **Draw buffers:** two static buffers of 128K pixels × 4 bytes each, 1 MiB BSS in total.
- **Exclude-object canvas:** `ExcludePanel`'s constructor allocates a ~902 KB canvas buffer (`map_buf`) at startup and never frees it.
- **Which thread allocates:** WebSocket callbacks run on the libhv thread, so everything they allocate (parsed messages, `State`, widgets created on connect) lands in that thread's separate glibc malloc arena.
- **LVGL heap:** `LV_MEM_CUSTOM 1`, so LVGL uses the system `malloc` with no fixed pool.
- **PNG cache:** decoded PNGs are kept in LVGL's image cache (`LV_IMG_CACHE_DEF_SIZE 5`) and are never invalidated explicitly.
- **`State`:** holds every Klipper object with all fields, including `configfile` (the full config twice, macro G-code included).
- **Object tree:** every panel's LVGL objects exist for the whole life of the process.
